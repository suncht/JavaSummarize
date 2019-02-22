# 详解 Spring 中 Bean 的 this 调用导致 AOP 失效的原因
引用： https://my.oschina.net/guangshan/blog/1807721

## 前言

在我们使用Spring时，可能有前辈教导过我们，在bean中不要使用this来调用被@Async、@Transactional、@Cacheable等注解标注的方法，this下注解是不生效的。

那么大家可曾想过以下问题
1. 为何致this调用的方法，注解会不生效
2. 这些注解生效的原理又是什么
3. 如果确实需要调用本类方法，且还需要注解生效，该怎么做？
4. 代理是否可以做到this调用注解就直接生效？
通过本文，上面的疑问都可以解决，而且可以学到很多相关原理知识，信息量较大，那么就开始吧

## 现象
以@Async注解为例，@Async注解标记的方法，在执行时会被AOP处理为异步调用，调用此方法处直接返回，@Async标注的方法使用其他线程执行。
使用Spring Boot驱动
```java
@SpringBootApplication
@EnableAsync
public class Starter {
 
    public static void main(String[] args) {
        SpringApplication.run(Starter.class, args);
    }
}
 
@Component
public class AsyncService {
 
    public void async1() {
        System.out.println("1:" + Thread.currentThread().getName());
        this.async2();
    }
 
    @Async
    public void async2() {
        System.out.println("2:" + Thread.currentThread().getName());
    }
}
 
@RunWith(SpringRunner.class) 
@SpringBootTest(classes = Starter.class)
public class BaseTest {
 
    @Autowired
    AsyncService asyncService;
 
    @Test
    public void testAsync() {
        asyncService.async1();
        asyncService.async2();
    }
}
```
输出内容为：
```
1:main
2:main
2:SimpleAsyncTaskExecutor-2
```
第一行第二行对应async1()方法，第三行对应async2()方法，可以看到直接使用asyncService.async2()调用时使用的线程为SimpleAsyncTaskExecutor，而在async1()方法中使用this调用，结果却是主线程，原调用线程一致。这说明@Async在this调用时没有生效。

## 思考&猜测
已知对于AOP动态代理，非接口的类使用的是基于CGLIB的动态代理，而CGLIB的动态代理，是基于现有类创建一个子类，并实例化子类对象。在调用动态代理对象方法时，都是先调用子类方法，子类方法中使用方法增强Advice或者拦截器MethodInterceptor处理子类方法调用后，选择性的决定是否执行父类方法。

那么假设在调用async1方法时，使用的是动态生成的子类的实例，那么this其实是基于动态代理的子类实例对象，this调用是可以被Advice或者MethodInterceptor等处理逻辑拦截的，那么为何理论和实际不同呢？

这里大胆推测一下，其实async1方法中的this不是动态代理的子类对象，而是原始的对象，故this调用无法通过动态代理来增强。

关于上面AOP动态代理使用CGLIB相关的只是，可以参考完全读懂[Spring框架之AOP实现原理](https://my.oschina.net/guangshan/blog/1797461)这篇文章。

下面开始详细分析。

## 源码调试分析原理
首先要弄清楚@Async是如何生效的：
### 1.分析Async相关组件
从生效入口开始看，@EnableAsync注解上标注了@Import(AsyncConfigurationSelector.class)

@Import的作用是把后面的@Configuration类、ImportSelector类或者ImportBeanDefinitionRegistrar类中import的内容自动注册到ApplicationContext中。关于这三种可Import的类，这里先不详细说明，有兴趣的读者可以自行去Spring官网查看文档或者等待我的后续文章。

这里导入了AsyncConfigurationSelector，而AsyncConfigurationSelector在默认情况下，会选择出来ProxyAsyncConfiguration类进行导入，即把ProxyAsyncConfiguration类作为@Configuration类配置到ApplicationContext中。那么这里的关键就是ProxyAsyncConfiguration类，看代码:
```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyAsyncConfiguration extends AbstractAsyncConfiguration {
 
    @Bean(name = TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
        Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
        AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
        Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
        if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
            bpp.setAsyncAnnotationType(customAsyncAnnotation);
        }
        if (this.executor != null) {
            bpp.setExecutor(this.executor);
        }
        if (this.exceptionHandler != null) {
            bpp.setExceptionHandler(this.exceptionHandler);
        }
        bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
        bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
        return bpp;
    }
 
}
```
这段代码的作用是把AsyncAnnotationBeanPostProcessor作为Bean注册到Context中。那么核心就是把AsyncAnnotationBeanPostProcessor这个BeanPostProcessor，也就是Spring大名鼎鼎的BPP。
在一个Bean实例生成后，会交给BPP的postProcessBeforeInitialization方法进行加工，此时可以返回与此Bean相兼容的其他Bean实例，例如最常见的就是在这里返回原对象的动态代理对象。
在这个方法执行后，会调用Bean实例的init相关方法。调用的方法是InitializingBean接口的afterPropertiesSet方法，以及@Bean声明中initMethod指定的初始化方法。
在调用init方法之后，会调用BPP的postProcessAfterInitialization方法进行后置处理。此时处理同postProcessBeforeInitialization，也可以替换原bean的实例。
我们看下这个Async相关的BPP做了什么操作：
```java
// 潜质处理不做任何动作，可保证在调用bean的init之前，bean本身没有任何变化。
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
    return bean;
}
 
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    // 如果是AOP相关的基础组件bean，如ProxyProcessorSupport类及其子类，则直接返回。
    if (bean instanceof AopInfrastructureBean) {
        // Ignore AOP infrastructure such as scoped proxies.
        return bean;
    }
 
    if (bean instanceof Advised) {
        // 如果已经是Advised的，即已经是被动态代理的实例，则直接添加advisor。
        Advised advised = (Advised) bean;
        if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
            // 如果没有被frozen(即冷冻，不再做改动的动态代理实例)且是Eligbile(合适的)，则把其添加到advisor中。根据配置决定插入位置。
            // Add our local Advisor to the existing proxy's Advisor chain...
            if (this.beforeExistingAdvisors) {
                advised.addAdvisor(0, this.advisor);
            }
            else {
                advised.addAdvisor(this.advisor);
            }
            return bean;
        }
    }
 
    if (isEligible(bean, beanName)) {
        // 如果是Eligible合适的，且还不是被代理的类，则创建一个代理类的实例并返回。
        ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
        if (!proxyFactory.isProxyTargetClass()) {
            evaluateProxyInterfaces(bean.getClass(), proxyFactory);
        }
        proxyFactory.addAdvisor(this.advisor);
        customizeProxyFactory(proxyFactory);
        return proxyFactory.getProxy(getProxyClassLoader());
    }
 
    // No async proxy needed.
    return bean;
}
// 准备ProxyFactory对象
protected ProxyFactory prepareProxyFactory(Object bean, String beanName) {
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);
    // 设置被代理的bean为target，这个bean是真实的bean。
    proxyFactory.setTarget(bean);
    return proxyFactory;
}
```
Spring在对一个类进行AOP代理后，会为此类加上Advised接口，返回的动态代理对象都会带上Advised接口修饰，那么第一段逻辑判断bean instanceof Advised的目的就是判断是否已经是被动态代理的类，如果是，则为其添加一个Advisor增强器。

如果不是动态代理的对象，因为@Async要为方法增加代理，并转换为异步执行，故需要把原始bean转换为被AOP动态代理的bean。也就是下面的逻辑。

有关上面这一段代码创建动态代理的详细原理，请参考我的这篇文章：[完全读懂Spring框架之AOP实现原理](https://my.oschina.net/guangshan/blog/1797461)。

关于@Async再多提一点：上面注册进去的advisor类型是AsyncAnnotationAdvisor。其中包括了PointCut，类型是AnnotationMatchingPointcut，指定了只有@Async标记的方法或者类此AOP增强器才生效。还有一个Advice，用于增强@Async标记的方法，转换为异步，类型是AnnotationAsyncExecutionInterceptor，其中的invoke方法是真正调用真实方法的地方，大家有兴趣可以仔细研究其中的内容，这样就能摸清楚@Async方法的真实执行逻辑了。

相关组件上面都已经提及并进行了简单的分析，现在我们进入下一阶段，通过真正的执行逻辑来分析this调用不生效的原因。

### 2.深入真实调用逻辑
@Async大多数都是标记的类中的方法，故AOP的实现也多是基于CGLIB的，下面以CGLIB动态代理为例分析真实调用逻辑。
通过完全读懂Spring框架之AOP实现原理这篇文章，可以得知，一个基于CGLIB的AOP动态代理bean，真实的执行逻辑是在DynamicAdvisedInterceptor中：
```java
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    Class<?> targetClass = null;
    Object target = null;
    try {
        if (this.advised.exposeProxy) {
            // 需要则暴露
            // Make invocation available if necessary.
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }
        // May be null. Get as late as possible to minimize the time we
        // "own" the target, in case it comes from a pool...
        // 重点：获取被代理的目标对象
        target = getTarget();
        if (target != null) {
            targetClass = target.getClass();
        }
        // 获取拦截器链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        Object retVal;
        // Check whether we only have one InvokerInterceptor: that is,
        // no real advice, but just reflective invocation of the target.
        if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
            // We can skip creating a MethodInvocation: just invoke the target directly.
            // Note that the final invoker must be an InvokerInterceptor, so we know
            // it does nothing but a reflective operation on the target, and no hot
            // swapping or fancy proxying.
            // 如果链是空且是public方法，则直接调用
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = methodProxy.invoke(target, argsToUse);
        }
        else {
            // We need to create a method invocation...
            // 否则创建一个CglibMethodInvocation以便驱动拦截器链
            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
        }
        // 处理返回值，同JDK动态代理
        retVal = processReturnType(proxy, target, method, retVal);
        return retVal;
    }
    finally {
        if (target != null) {
            releaseTarget(target);
        }
        if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```
注意上面真实调用的部分，在没有advisor的情况下，使用的其实是：

methodProxy.invoke(target, argsToUse)

在有代理的情况下，使用的是：

new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();

而在CglibMethodInvocation中，检查到调用链执行完之后，会调用真实的方法：invokeJoinpoint。在CglibMethodInvocation中，该方法的实现是:
```java
// CglibMethodInvocation中的实现
protected Object invokeJoinpoint() throws Throwable {
    if (this.publicMethod) {
        return this.methodProxy.invoke(this.target, this.arguments);
    }
    else {
        return super.invokeJoinpoint();
    }
}
// 父类实现是
protected Object invokeJoinpoint() throws Throwable {
    return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
}
```
可以看到调用方法时，传入的实例都是target，这个target是从DynamicAdvisedInterceptor的getTarget方法中获得的，代码如下:
```java
protected Object getTarget() throws Exception {
    return this.advised.getTargetSource().getTarget();
}
```
而这个advised的target则是在ProxyFactory的实例方法中设置的：proxyFactory.setTarget(bean);

也就是说这个target其实是真实的被代理的bean。

通过上面的分析，我们可以得到结论，在一个被动态代理的对象，在执行完AOP所有的增强逻辑之后，最终都会使用被代理对象作为实例调用真实的方法，即相当于调用了：target.method()方法。由此得出结论，在target.method()方法中，this引用必然是target自身，而不是生成的动态代理对象实例。
![](https://note.youdao.com/yws/api/personal/file/68456C91529245FBBE348C6154CA385B?method=download&shareKey=831bf970fede1382613d72a1fb81940e)
总结： 因为AOP动态代理的方法真实调用，会使用真实被代理对象实例进行方法调用，故在实例方法中通过this获取的都是被代理的真实对象的实例，而不是代理对象自身。

### 3. 解决this调用的几个替代方法
既然已知原因，那么解决的方法就有定向了，核心就是如何获得动态代理对象，而不是使用this去调用。

提供以下几种方法：
1. 通过ApplicationContext来获得动态代理对象
```java
@Component
public class AsyncService implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    public void async1() {
        System.out.println("1:" + Thread.currentThread().getName());
        // 使用AppicationContext来获得动态代理的bean
        this.applicationContext.getBean(AsyncService.class).async2();
    }

    @Async
    public void async2() {
        System.out.println("2:" + Thread.currentThread().getName());
    }

    // 注入ApplicationContext
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```
执行结果是：
```
1:main
2:SimpleAsyncTaskExecutor-2
2:SimpleAsyncTaskExecutor-3
```
可以看到完美达到了我们的目的。同理是用BeanFactoryAware可达到同样的效果。

2. 通过AopContext获取动态代理对象
```java
@Component
public class AsyncService {

    public void async1() {
        System.out.println("1:" + Thread.currentThread().getName());
        ((AsyncService) AopContext.currentProxy()).async2();
    }

    @Async
    public void async2() {
        System.out.println("2:" + Thread.currentThread().getName());
    }

}
```
这种做法非常简洁，但是在默认情况下是不起作用的！ 因为AopContext中拿不到currentProxy，会报空指针。
通过上面的动态代理执行源码的地方可以看到逻辑：
```java
if (this.advised.exposeProxy) {
	// Make invocation available if necessary.
	oldProxy = AopContext.setCurrentProxy(proxy);
	setProxyContext = true;
}
```
而在ProxyConfig类中，有如下注释用来说明exposeProxy的作用，就是用于在方法中获取动态代理的对象的。
```java
/**
 * Set whether the proxy should be exposed by the AOP framework as a
 * ThreadLocal for retrieval via the AopContext class. This is useful
 * if an advised object needs to call another advised method on itself.
 * (If it uses {@code this}, the invocation will not be advised).
 * <p>Default is "false", in order to avoid unnecessary extra interception.
 * This means that no guarantees are provided that AopContext access will
 * work consistently within any method of the advised object.
 */
public void setExposeProxy(boolean exposeProxy) {
	this.exposeProxy = exposeProxy;
}
```
即只有exposeProxy为true时，才会把proxy动态代理对象设置到AopContext上下文中，这个配置默认是false。那么这个配置怎么修改呢？

在xml时代，我们可以通过配置：
```xml
<aop:aspectj-autoproxy proxy-target-class="true" expose-proxy="true"/>  
```
来修改全局的暴露逻辑。
在基于注解的配置中，我们需要使用
```java
@EnableAspectJAutoProxy(proxyTargteClass = true, exposeProxy = true)
```
来配置。
遗憾的是，对于@Async，如此配置下依然不能生效。因为@Async使用的不是AspectJ的自动代理，而是使用代码中固定的创建代理方式进行代理创建的。
如果是@Transactional事务注解的话， 则是生效的。具体生效机制是通过@EnableTransactionManagement注解中的TransactionManagementConfigurationSelector类声明，其中声明导入了AutoProxyRegistrar类，该类获取注解中proxy相关注解配置，并根据配置情况，在BeanDefinition中注册一个可用于自动生成代理对象的AutoProxyCreator：
```java
AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
	return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
}
```
而在@EnableAspectJAutoProxy注解中，@Import的AspectJAutoProxyRegistrar类又把这个BeanDefinition修改了类，同时修改了其中的exposeProxy属性。
```java
AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
	return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
}
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
	return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```
后面替换掉了前面的AutoProxyCreator，替换逻辑是使用优先级替换，优先级分别为：
```java
APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);
APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);
APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);
```
这个逻辑都在registerOrEscalateApcAsRequired中，读者可以自己再看一下。
因为@Transactional注解和AspectJ相关注解的生成动态代理类都是使用的同一个Bean即上面的AutoProxyCreator处理的，该bean的name是`org.springframework.aop.config.internalAutoProxyCreator`，他们公用相同的属性，故对于@Transactional来说，@EnableAspectJAutoProxy的属性exposeProxy=true也是生效的。但是@Async的注解生成的代理类并不是通过这个autoProxyCreator来生成的，故不能享受到上面的配置。

3. 基于上面的源码，我们可以得到第三种处理方法
在某个切入时机，手动执行`AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);`静态方法，当然前提是有一个BeanDefinitionRegistry，且时机要在BeanDefinition已经创建且动态代理对象还没有生成时调用。

使用这种方式，无需使用@EnableAspectJAutoProxy即可。

这种方式同样不适用于@Async，适用于@Transactional。
4. 手动修改各种BeanPostProcessor的属性
以@Async为例，其通过AsyncAnnotationBeanPostProcessor来生成动态代理类，我们只要在合适时机即该BPP已创建，但是还未被使用时，修改其中的exposeProxy属性，使用`AsyncAnnotationBeanPostProcessor.setExposeProxy(true)`即可。

这种方式要针对性的设置特定的bean的exposeProxy属性true。适用于@Async，观察原理可以知道3和4其实核心都是相同的，就是设置AutoProxyCreater的exposed属性为true。AsyncAnnotationBeanPostProcessor其实也是一个AutoProxyCreater，他是ProxyProcessorSupport的子类。
对于@Async可以使用1、4方式，对于@Transactional则可以使用这四种任意方式。
...

### 4. 是否可以做到this调用使动态代理生效
基于我们的推测，如果this引用是动态代理对象的话，则this调用其实是可以调用到父类的方法的，只要调用的是父类方法，那么在父类重写的方法中加入的动态代理拦截就是可以生效的。此种场景在Spring中是否存在呢？答案是肯定的，就在Spring提供的@Configuration配置类中，就有这种场景的应用，下面见示例：
```java
@Configuration
public class TestConfig {
    
    @Bean
    public Config config() {
        return new Config();
    }
    
    @Bean
    public ConfigOut configOut() {
        Config c1 = this.config();
        Config c2 = this.config();
        System.out.println(c1 == c2);
        ConfigOut configOut = new ConfigOut(this.config());
        return configOut;
    }

    public static class Config {}
    
    public static class ConfigOut {
        
        private Config config;
        
        private ConfigOut(Config config) {
            this.config = config;
        }
        
    }
    
}
```
在configOut方法中加入断点，调试观察c1与才 的值，也即this.config()返回的值，可以看到c1和c2是同一个对象引用，而不是每次调用方法都new一个新的对象。 
![](https://note.youdao.com/yws/api/personal/file/9D878CFAF23B4B55B36DB8E3FC0A9979?method=download&shareKey=8dc359d288f20ebe400b3ac6dabe5101)
那么这里是怎么做到this调用多次都返回同一个实例的呢？我们继续跟踪调试断点，查看整体的调用堆栈，发现这个方法configOut的调用处以及config方法的真实调用处是在ConfigurationClassEnhancer的内部类BeanMethodInterceptor中，为什么是这个方法呢？因为真实的Configuration类被动态替换为基于CGLIB创建的子类了。而这个@Configuration类的处理，是基于ConfigurationClassPostProcessor这个BeanFactoryPostProcessor处理器来做的，在ConfigurationClassPostProcessor中的postProcessBeanDefinitionRegistry方法中，检查所有的bean，如果bean是被@Configuration、@Component、@ComponentScan、@Import、@ImportResource其中一个标注的，那么此类就会被视为Configuration类。在postProcessBeanDefinition方法中，会把@Configuration类动态代理为一个新类，使用CGLIB的enhancer来增强Configuration类。使用ConfigurationClassEnhancer的enhance方法处理为原有类的子类，参考代码：
```java
/**
 * Loads the specified class and generates a CGLIB subclass of it equipped with
 * 加载特殊的Configuration类时，为其生成一个CGLIB的子类
 * container-aware callbacks capable of respecting scoping and other bean semantics.
 * 以便实现对@Bean方法的拦截或者增强
 * @return the enhanced subclass
 */
public Class<?> enhance(Class<?> configClass, ClassLoader classLoader) {
	if (EnhancedConfiguration.class.isAssignableFrom(configClass)) {
	    // 如果已经是被增强的Configuration，则直接跳过
		if (logger.isDebugEnabled()) {
			logger.debug(String.format("Ignoring request to enhance %s as it has " +
					"already been enhanced. This usually indicates that more than one " +
					"ConfigurationClassPostProcessor has been registered (e.g. via " +
					"<context:annotation-config>). This is harmless, but you may " +
					"want check your configuration and remove one CCPP if possible",
					configClass.getName()));
		}
		return configClass;
	}
	// 否则生成增强后的新的子类
	Class<?> enhancedClass = createClass(newEnhancer(configClass, classLoader));
	if (logger.isDebugEnabled()) {
		logger.debug(String.format("Successfully enhanced %s; enhanced class name is: %s",
				configClass.getName(), enhancedClass.getName()));
	}
	return enhancedClass;
}

/**
 * Creates a new CGLIB {@link Enhancer} instance.
 * 创建增强的CGLIB子类
 */
private Enhancer newEnhancer(Class<?> superclass, ClassLoader classLoader) {
	Enhancer enhancer = new Enhancer();
	enhancer.setSuperclass(superclass);
	// 增加接口以标记是被增强的子类，同时增加setBeanFactory方法，设置内部成员为BeanFactory。
	enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
	enhancer.setUseFactory(false);
	enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
	// BeanFactoryAwareGeneratorStrategy生成策略为生成的CGLIB类中添加成员变量$$beanFactory
	// 同时基于接口EnhancedConfiguration的父接口BeanFactoryAware中的setBeanFactory方法，设置此变量的值为当前Context中的beanFactory
	// 该BeanFactory的作用是在this调用时拦截该调用，并直接在beanFactory中获得目标bean。
	enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
	// 设置CALLBACK_FILTER，
	enhancer.setCallbackFilter(CALLBACK_FILTER);
	enhancer.setCallbackTypes(CALLBACK_FILTER.getCallbackTypes());
	return enhancer;
}

// 增强时要使用的filters
// The callbacks to use. Note that these callbacks must be stateless.
private static final Callback[] CALLBACKS = new Callback[] {
        // 用于拦截@Bean方法的调用，并直接从BeanFactory中获取目标bean，而不是通过执行方法。
		new BeanMethodInterceptor(),
		// 用于拦截BeanFactoryAware接口中的setBeanFactory方法的嗲用，以便设置$$beanFactory的值。
		new BeanFactoryAwareMethodInterceptor(),
		// 不做任何操作
		NoOp.INSTANCE
};

/**
 * Uses enhancer to generate a subclass of superclass,
 * ensuring that callbacks are registered for the new subclass.
 * 设置callbacks到静态变量中，因为还没有实例化，所以只能放在静态变量中。
 */
private Class<?> createClass(Enhancer enhancer) {
	Class<?> subclass = enhancer.createClass();
	// Registering callbacks statically (as opposed to thread-local)
	// is critical for usage in an OSGi environment (SPR-5932)...
	Enhancer.registerStaticCallbacks(subclass, CALLBACKS);
	return subclass;
}
```
可以看到这里的callbacks是注册到生成的子类的static中，这里只生成class而不实例化。
把此类设置到BeanDefinition中的beanClass属性中，在BeanDefinition初始化时会自动初始化子类。
上面的关键是CALLBACKS、CALLBACK_FILTER，分别代表增强器和增强器的过滤器。
关于Configuration类的CGLIB动态代理创建可以与SpringAOP体系创建的CGLIB动态代理做一个对比，区别是这里的动态代理的CALLBACKS和CALLBACK_FILTER。
这里我们以上面提到的BeanMethodInterceptor为例，来说明他的作用，以及this调用在这种情况下可以被动态代理拦截的原因。代码如下：
```java
/**
 * enhancedConfigInstance: 被CGLIB增强的config类的实例，即CGLIB动态生成的子类的实例
 * beanMethod : @Bean标记的方法，即当前调用的方法，这个是通过CallbackFilter的accept方法筛选出来的，只可能是@Bean标注的方法。
 * beanMethodArgs : 方法调用的参数
 * cglibMethodProxy : cglib方法调用的代理，可以用来直接调用父类的真实方法。
*/
@Override
public Object intercept(Object enhancedConfigInstance, Method beanMethod, Object[] beanMethodArgs,
			MethodProxy cglibMethodProxy) throws Throwable {
    // 通过enhancedConfigInstance中cglib生成的成员变量$$beanFactory获得beanFactory。
	ConfigurableBeanFactory beanFactory = getBeanFactory(enhancedConfigInstance);
	// 确认真实的beanName,用于在beanFactory中获得bean实例
	String beanName = BeanAnnotationHelper.determineBeanNameFor(beanMethod);

	// Determine whether this bean is a scoped-proxy
	// 后面这个是确认是否是scoped作用域的bean，这里暂时不考虑，后续文章详细分析Scoped相关的逻辑和bean。
	Scope scope = AnnotatedElementUtils.findMergedAnnotation(beanMethod, Scope.class);
	if (scope != null && scope.proxyMode() != ScopedProxyMode.NO) {
		String scopedBeanName = ScopedProxyCreator.getTargetBeanName(beanName);
		if (beanFactory.isCurrentlyInCreation(scopedBeanName)) {
			beanName = scopedBeanName;
		}
	}

	// To handle the case of an inter-bean method reference, we must explicitly check the
	// container for already cached instances.
	// 拦截内部bean方法的调用，检查bean实例是否已经生成

	// First, check to see if the requested bean is a FactoryBean. If so, create a subclass
	// proxy that intercepts calls to getObject() and returns any cached bean instance.
	// This ensures that the semantics of calling a FactoryBean from within @Bean methods
	// is the same as that of referring to a FactoryBean within XML. See SPR-6602.
	// 检查是否是FactoryBean，当是FactoryBean时，即使是this调用也不能生成多次
	// 更特殊的，调用FactoryBean的getObject方法时，也不能生成多次新的Bean，否则取到的bean就是多个了，有违单例bean的场景。
	// 所以这里判断如果当前方法返回的bean，如果是FactoryBean的话，对FactoryBean进行代理
	// 代理的结果是拦截factoryBean实例的getObject方法，转化为通过BeanFactory的getBean方法来调用
	if (factoryContainsBean(beanFactory, BeanFactory.FACTORY_BEAN_PREFIX + beanName) &&
			factoryContainsBean(beanFactory, beanName)) {
		// 上面加入BeanFactory.FACTORY_BEAN_PREFIX + beanName用来判断当前bean是否是一个FactoryBean。在BeanFactory中是通过FACTORY_BEAN_PREFIX前缀来区分当前要判断的目标类型的，
		// 如果是FACTORY_BEAN_PREFIX前缀的beanName，则获取之后会判断是否是FactoryBean，是则为true，否则为false。
		// 同时还判断了当前的Bean是否是在创建中，只有不是在创建中，才会返回true。第一个拿FactoryBean的name去判断，则肯定不在创建中。第二个的判断才是真正生效的可判断出是否在创建中的方法。
		Object factoryBean = beanFactory.getBean(BeanFactory.FACTORY_BEAN_PREFIX + beanName);
		// 只有不在创建中，才能调用BeanFactory去获取或者创建，否则会无限递归调用。
		// 上面的调用获取时，才会进行真正的初始化，实例化时还会再进一次这个方法，但是并不会执行到这个逻辑中，因为再进入时，会被标记为正在创建。真正的初始化时调用@Bean方法进行的，是在下面的逻辑中。
		if (factoryBean instanceof ScopedProxyFactoryBean) {
			// Scoped proxy factory beans are a special case and should not be further proxied
		}
		else {
			// It is a candidate FactoryBean - go ahead with enhancement
			return enhanceFactoryBean(factoryBean, beanMethod.getReturnType(), beanFactory, beanName);
		}
	}

	if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
	    // 上面这个用于判断当前的工厂方法，也就是@Bean标注的方法是否是在调用中。如果是在调用中，则说明需要真正的实例化了，此时调用父类真是方法来创建实例。
		// The factory is calling the bean method in order to instantiate and register the bean
		// (i.e. via a getBean() call) -> invoke the super implementation of the method to actually
		// create the bean instance.
		if (logger.isWarnEnabled() &&
				BeanFactoryPostProcessor.class.isAssignableFrom(beanMethod.getReturnType())) {
		    // 如果是BeanFactoryPostProcessor类型的话则提出警告，表明可能并不能正确执行BeanFactoryPostProcessor的方法。
			logger.warn(String.format("@Bean method %s.%s is non-static and returns an object " +
							"assignable to Spring's BeanFactoryPostProcessor interface. This will " +
							"result in a failure to process annotations such as @Autowired, " +
							"@Resource and @PostConstruct within the method's declaring " +
							"@Configuration class. Add the 'static' modifier to this method to avoid " +
							"these container lifecycle issues; see @Bean javadoc for complete details.",
					beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName()));
		}
		// 调用父类真实方法实例化。
		return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
	}
    // 这个方法尝试从beanFactory中获得目标bean，这样便可另所有此方法调用获得bean最终都是从beanFactory中获得的，达到了单例的目的。
	return obtainBeanInstanceFromFactory(beanMethod, beanMethodArgs, beanFactory, beanName);
	// 在Bean的方法A使用this引用调用方法B时，会先进入一次这个方法的逻辑，此时因为还没真正进行实例化，
	// isCurrentlyInvokedFactoryMethod(beanMethod)得到的结过是false，故会调用obtainBeanInstanceFromFactory，此时会从beanFactory中获得bean。
	// 在获得Bean时，会再次调用B方法，因为这个Bean需要调用@Bean的方法才能生成。调用前先打上正在调用的标记，同时再次进入这个方法逻辑，此时上面判断isCurrentlyInvokedFactoryMethod结过为true，调用父类方法进行真实的实例化。
}

/**
 * 该方法为FactoryBean返回被代理的新实例，新的实例拦截getObject方法，并从beanFactory中获得单例bean。
 */
private Object enhanceFactoryBean(final Object factoryBean, Class<?> exposedType,
		final ConfigurableBeanFactory beanFactory, final String beanName) {

	try {
		Class<?> clazz = factoryBean.getClass();
		boolean finalClass = Modifier.isFinal(clazz.getModifiers());
		boolean finalMethod = Modifier.isFinal(clazz.getMethod("getObject").getModifiers());
		// 判断真实FactoryBean的类型和getObject方法，如果是final的，说明不能通过CGLIB代理，则尝试使用JDK代理
		if (finalClass || finalMethod) {
			if (exposedType.isInterface()) {
			    // 如果方法返回类型，即exposedType是接口，则这个接口一般都是FactoryBean，则通过jdk动态代理创建代理
				if (logger.isDebugEnabled()) {
					logger.debug("Creating interface proxy for FactoryBean '" + beanName + "' of type [" +
							clazz.getName() + "] for use within another @Bean method because its " +
							(finalClass ? "implementation class" : "getObject() method") +
							" is final: Otherwise a getObject() call would not be routed to the factory.");
				}
				return createInterfaceProxyForFactoryBean(factoryBean, exposedType, beanFactory, beanName);
			}
			else {
			    // 不是接口就没办法了，只能直接返回原始的factoryBean，如果在这个factoryBean里getObject生成了新对象，多次调用生成的结果bean将不会是同一个实例。
				if (logger.isInfoEnabled()) {
					logger.info("Unable to proxy FactoryBean '" + beanName + "' of type [" +
							clazz.getName() + "] for use within another @Bean method because its " +
							(finalClass ? "implementation class" : "getObject() method") +
							" is final: A getObject() call will NOT be routed to the factory. " +
							"Consider declaring the return type as a FactoryBean interface.");
				}
				return factoryBean;
			}
		}
	}
	catch (NoSuchMethodException ex) {
		// No getObject() method -> shouldn't happen, but as long as nobody is trying to call it...
	}
    // 可以使用CGLIB代理类。
	return createCglibProxyForFactoryBean(factoryBean, beanFactory, beanName);
	// 假设A方法调用了@Bean的B方法，B方法返回FactoryBean实例
	// 那么在A调用B时，会先进入BeanMethodInterceptor.intercept方法
	// 在方法中判断目标bean是一个FactoryBean，且不是在创建中，则调用beanFactory的getBean尝试获取目标bean。
	// 在获取的过程中，最终又会执行方法B，此时被拦截再次进入这个intercept方法
	// 由于标记为创建中，故这里会进入下面的创建中逻辑，通过invokeSuper调用了真实的方法逻辑返回真实的FactoryBean。
	// 这个真实的FactoryBean返回之后，在第一次的intercept方法中，对这个FactoryBean实例进行代理，返回一个被代理的FactoryBean对象给方法A中的逻辑使用，这样就可以保证在A中调用FactoryBean.getObject时拿到的是beanFactory的bean实例了。
}
```
通过BeanMethodInterceptor.intercept方法，我们可以看到，真实的方法调用是通过cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs)来执行的，enhancedConfigInstance是动态代理产生的子类的实例，这里直接调用该对象的父类方法，即相当于调用的真实方法，这一点与Spring AOP体系中的把真实对象target作为真实调用实例来调用是有区别的，也就是这个区别，给this调用带来的上面的特性。

即在这种情况下this都是被CGLIB动态代理产生的子类的实例，在调用this.method()时，其实是调用了子类实例的该方法，此方法可以被方法拦截器拦截到，在拦截的逻辑中做一定的处理，如果需要调用真实对象的相应方法，直接使用invokeSuper来进行父类方法调用，而不是传入真实被动态代理对象的实例来进行调用。真实对象其实并没有创建，也就是说对应于Spring AOP，其中的target是不存在的，只有子类对象动态代理自身的实例，而没有真实对象实例。

由此我们便明了了this调用被动态拦截的实现方式。

对于上面Configuration的类的调用，可参考如下例子，对比调试后可以更加深入的理解这个问题。
```java
import org.springframework.beans.factory.FactoryBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TestConfig {

    @Bean
    public ConfigOut configOut() {
        Config c1 = this.config();
        Config c2 = this.config();
        // 这里返回同一个实例
        System.out.println(c1 == c2);
        ConfigOut configOut = new ConfigOut(this.config());
        FactoryBean ob1 = this.objectFactoryBean();
        FactoryBean ob2 = this.objectFactoryBean();
        // 这里也是 同一个实例
        System.out.println(ob1 == ob2);
        MyObject myObject1 = this.objectFactoryBean().getObject();
        MyObject myObject2 = this.objectFactoryBean().getObject();
        // 如果objectFactoryBean方法返回类型为FactoryBean则这两个相同
        // 如果是ObjectFactoryBean则两个不相同，上面已分析过原因
        System.out.println(myObject1 == myObject2);
        return configOut;
    }

    @Bean
    public Config config() {
        return new Config();
    }

    @Bean
    public FactoryBean objectFactoryBean() {
        return new ObjectFactoryBean();
    }

    public static class Config {}

    public static class ConfigOut {

        private Config config;

        private ConfigOut(Config config) {
            this.config = config;
        }

    }

    public static final class ObjectFactoryBean implements FactoryBean<MyObject> {

        @Override
        public final MyObject getObject() {
            return new MyObject();
        }

        @Override
        public Class<?> getObjectType() {
            return MyObject.class;
        }

        @Override
        public boolean isSingleton() {
            return true;
        }
    }

    public static class MyObject {}

}
```
### 后记
本文根据实际场景，详细的分析了this调用导致AOP失效的原因，以及如何解决这个问题。并扩展了this调用可使AOP生效的场景。只要大家能理解到原理面，应该都能够分析出来原因。平时一些需要遵守的代码规范，在原理层面都是有其表现和原因的，分析真实原因得到最终结论，这个过程是对知识的升华过程，希望大家能够看到开心。