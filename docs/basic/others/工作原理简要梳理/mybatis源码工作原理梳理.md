## 一. MyBatis与Spring集成原理
1. MapperScan注解 -> @Import(MapperScannerRegistrar)

2. MapperScannerRegistrar实现ImportBeanDefinitionRegistrar， 获取Mapper接口

3. Mapper接口 -> BeanDefinition-> 由于Mapper接口不能生成对象，beanDefinition的BeanClass改成MapperFactoryBean **（核心部分）** -> MapperFactoryBean， 使用时的MapperFactoryBean(mapperInterface)的构造函数

> 任何Mapper接口都会生成MapperFactoryBean和MapperProxy代理类

4. 如何生成MapperProxy代理类
MapperFactoryBean.getObject() -> MapperRegistry.getMapper() -> 将Mapper接口包装到MapperProxy -> MapperProxyFactory.newInstance(MapperProxy)生成MapperProxy代理实例， MapperProxy实现InvocationHandler

5. 应用时，mapper调用方法时，就是调用MapperProxy.invoke -> MapperMethod的execute方法，执行SQL并返回结果。（见第二节）

## 二. 调用mapper方法时，如何执行SQL并返回结果
1. 如何执行SQL语句
首先应用中mapper接口被生成MapperProxy代理类（见上述第一节）， 调用mapper方法时，就是调用MapperProxy.invoke 
-> 获取MapperMethod（mapper接口 + SQL操作类型）
-> MapperMethod的execute方法， 根据SQL操作类型，选择具体操作方法(比如Select)， 获取参数值param 
-> 调用SqlSession的SelectList等方法 **（核心部分）**，传递参数值param， DefaultSqlSession是SqlSession具体实现 ，负责事务提交、关闭
-> 根据statement（也就是mapper.xml的域名+语句id），获取MappedStatement 
-> 调用executor执行器的query/update/queryCursor **（核心部分）**，默认是CachingExecutor，启动一级缓存，负责SQL执行
-> 从MappedStatement获取BoundSql（也就是SQL语句），参数值param 
-> 由StatementHandler执行sql，默认是PreparedStatementHandler， 调用JDBC相关方法

Executor实现类：SimpleExecutor、ReuseExecutor、BatchExecutor、CachingExecutor（包装模式）
StatementHandler实现类：SimpleStatementHandler、PreparedStatementHandler、CallableStatementHandler、RoutingStatementHandler（包装模式）


2. 如何返回结果

