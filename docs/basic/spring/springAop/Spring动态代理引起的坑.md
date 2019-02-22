# Spring AOP动态代理引起的坑的一个思考
## 前言
大家在Spring Aop动态代理注解（比如：异步注解@Async、事务注解@Transactional）时，都可能会遇到动态代理失效问题，这个问题在百度上查询一大把一大把（大家有兴趣的话可以自己去搜索），解决方案都是用AopContext.currentProxy()，但是大家细心发现并没有说明为什么会要使用这个AopContext.currentProxy()。

## 例子
先看一下几个问题，下面是段代码：
```java
public interface UserService{
	public void a();
	public void b();
}

public class UserServiceImpl implements UserService{
	@Transactional(propagation = Propagation.REQUIRED)
	public void a(){
		this.b(); //事务失效
	}
	@Transactional(propagation = Propagation.REQUIRED_NEW)
	public void b(){
		System.out.println("b has been called");
	}
}
```
1. b中的事务会不会生效？
> a的事务会生效，b中不会有事务，因为a中调用b属于内部调用，没有通过代理，所以不会有事务产生。

2. 如果想要b中有事务存在，要如何做？
 > `<aop:aspectj-autoproxy expose-proxy="true">`，设置expose-proxy属性为true，将代理暴露出来，使用`AopContext.currentProxy()`获取当前代理，将this.b()改为`((UserService)AopContext.currentProxy()).b()`

但对此，我抱有以下2个疑问：
1. Spring Aop动态代理生成了一个代理类， a方法就是在这个代理类，那么b方法什么不在这个代理类？？
2. a方法中调用`this.b()`，这里的this不是a所在的代理类对象吗？

## 源码分析
先@Asynch异步注解为例，进行源码走读：
我们可以打断点，代码会走到AsyncExecutionInterceptor类中invoke方法，看下invoke方法源码：
```java
@Override
public Object invoke(final MethodInvocation invocation) throws Throwable {
    Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
    Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
    final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);

    // <关键点> 获取异步任务执行器，根据方法名
    AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
    if (executor == null) {
        throw new IllegalStateException(
                "No executor specified and no default executor set on AsyncExecutionInterceptor either");
    }

    Callable<Object> task = new Callable<Object>() {
        @Override
        public Object call() throws Exception {
            try {
                // 调用原对象的指定方法
                Object result = invocation.proceed();
                if (result instanceof Future) {
                    return ((Future<?>) result).get();
                }
            }
            catch (ExecutionException ex) {
                handleError(ex.getCause(), userDeclaredMethod, invocation.getArguments());
            }
            catch (Throwable ex) {
                handleError(ex, userDeclaredMethod, invocation.getArguments());
            }
            return null;
        }
    };

    return doSubmit(task, executor, invocation.getMethod().getReturnType());
}
```
其中determineAsyncExecutor方法是关键点，看下这个方法的源码：
```java
protected AsyncTaskExecutor determineAsyncExecutor(Method method) {
    // 从缓存中获取异步任务执行器
    AsyncTaskExecutor executor = this.executors.get(method);
    if (executor == null) {
        Executor targetExecutor;
        // 生成异步任务执行器，并保存到缓存中去
        String qualifier = getExecutorQualifier(method);
        if (StringUtils.hasLength(qualifier)) {
            targetExecutor = findQualifiedExecutor(this.beanFactory, qualifier);
        }
        else {
            targetExecutor = this.defaultExecutor;
            if (targetExecutor == null) {
                synchronized (this.executors) {
                    if (this.defaultExecutor == null) {
                        // 生成异步任务执行器，如果没有配置TaskExecutor，则默认使用SimpleAsyncTaskExecutor
                        this.defaultExecutor = getDefaultExecutor(this.beanFactory);
                    }
                    targetExecutor = this.defaultExecutor;
                }
            }
        }
        if (targetExecutor == null) {
            return null;
        }
        executor = (targetExecutor instanceof AsyncListenableTaskExecutor ?
                (AsyncListenableTaskExecutor) targetExecutor : new TaskExecutorAdapter(targetExecutor));
        // <关键点> 保存到缓存中去，KEY是方法         
        this.executors.put(method, executor);
    }
    return executor;
}
```
从上述代码上看，用当前方法获取异步任务执行器，也就是说**一个方法对应一个异步任务执行器**。


整理下我的理解：
1. @Async注解的方法，只会在自己所在的异步任务执行器中执行，其他方法只是一般性的方法而已
2. 如果不设置异步任务执行器，默认会使用SimpleAsyncTaskExecutor，这只是使用Thread执行异步，没有使用线程池，实际使用会有很大隐患。需要需要手动配置线程池的异步执行器。






