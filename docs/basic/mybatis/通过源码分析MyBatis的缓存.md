# 通过源码分析MyBatis的缓存

引用文章： https://www.cnblogs.com/fangjian0423/p/mybatis-cache.html

# MyBatis缓存介绍

MyBatis支持声明式数据缓存（declarative data caching）。当一条SQL语句被标记为“可缓存”后，首次执行它时从数据库获取的所有数据会被存储在一段高速缓存中，今后执行这条语句时就会从高速缓存中读取结果，而不是再次命中数据库。MyBatis提供了默认下基于Java HashMap的缓存实现，以及用于与OSCache、Ehcache、Hazelcast和Memcached连接的默认连接器。MyBatis还提供API供其他缓存实现使用。

重点的那句话就是：MyBatis执行SQL语句之后，这条语句就是被缓存，以后再执行这条语句的时候，会直接从缓存中拿结果，而不是再次执行SQL

这也就是大家常说的MyBatis一级缓存，一级缓存的作用域scope是SqlSession。

MyBatis同时还提供了一种全局作用域global scope的缓存，这也叫做二级缓存，也称作全局缓存。

## 一级缓存

### 源码分析

在分析MyBatis的一级缓存之前，我们先简单看下MyBatis中几个重要的类和接口：
`org.apache.ibatis.session.Configuration`类：MyBatis全局配置信息类
`org.apache.ibatis.session.SqlSessionFactory`接口：操作SqlSession的工厂接口，具体的实现类是DefaultSqlSessionFactory
`org.apache.ibatis.session.SqlSession`接口：执行sql，管理事务的接口，具体的实现类是DefaultSqlSession
`org.apache.ibatis.executor.Executor`接口：sql执行器，SqlSession执行sql最终是通过该接口实现的，常用的实现类有SimpleExecutor和CachingExecutor,这些实现类都使用了装饰者设计模式
一级缓存的作用域是SqlSession，那么我们就先看一下SqlSession的select过程：
这是DefaultSqlSession（SqlSession接口实现类，MyBatis默认使用这个类）的selectList源码
```java
  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
我们看到SqlSession最终会调用Executor接口的方法。
接下来我们看下DefaultSqlSession中的executor接口属性具体是哪个实现类。
DefaultSqlSession的构造过程（DefaultSqlSessionFactory内部）：
```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType, autoCommit);
      return new DefaultSqlSession(configuration, executor);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
}
```
我们看到DefaultSqlSessionFactory构造DefaultSqlSession的时候，Executor接口的实现类是由Configuration构造的：
```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }

    if (cacheEnabled) { //cacheEnable默认开启一级缓存
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
Executor根据ExecutorType的不同而创建，最常用的是SimpleExecutor，本文的例子也是创建这个实现类。 最后我们发现如果cacheEnabled这个属性为true的话，那么executor会被包一层装饰器，这个装饰器是CachingExecutor。其中cacheEnabled这个属性是mybatis总配置文件中settings节点中cacheEnabled子节点的值，默认就是true，也就是说我们在mybatis总配置文件中不配cacheEnabled的话，它也是默认为打开的。
现在，问题就剩下一个了，CachingExecutor执行sql的时候到底做了什么？

带着这个问题，我们继续走下去（CachingExecutor的query方法）：
```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache(); //这里的cache是二级缓存
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, parameterObject, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```
其中Cache cache = ms.getCache();这句代码中，这个cache实际上就是个二级缓存，由于我们没有开启二级缓存(二级缓存的内容下面会分析)，因此这里执行了最后一句话。这里的delegate也就是SimpleExecutor,SimpleExecutor没有Override父类的query方法，因此最终执行了SimpleExecutor的父类BaseExecutor的query方法。

所以一级缓存最重要的代码就是BaseExecutor的query方法!
```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      //localCache是BaseExecutor内部属性，该属性就是一级缓存，先从一级缓存中获取数据
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        //如果一级缓存拿不到数据，则从数据库中查询
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```