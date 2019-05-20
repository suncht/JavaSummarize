### 1. 安全获取Unsafe
```java
private static final Unsafe THE_UNSAFE;

static
{
    try
    {
        final PrivilegedExceptionAction<Unsafe> action = new PrivilegedExceptionAction<Unsafe>()
        {
            public Unsafe run() throws Exception
            {
                Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
                theUnsafe.setAccessible(true);
                return (Unsafe) theUnsafe.get(null);
            }
        };

        THE_UNSAFE = AccessController.doPrivileged(action);
    }
    catch (Exception e)
    {
        throw new RuntimeException("Unable to load unsafe", e);
    }
}

public static Unsafe getUnsafe()
{
    return THE_UNSAFE;
}

//使用
private static final Unsafe UNSAFE = Util.getUnsafe();

static
{
    UNSAFE = Util.getUnsafe();
    try
    {
        VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.class.getDeclaredField("value"));
    }
    catch (final Exception e)
    {
        throw new RuntimeException(e);
    }
}
```

> 关于AccessController，可参考：https://www.zybuluo.com/changedi/note/417132

### 2. CAS编写
```java
protected volatile int value;

public final int incrementAndGet() {
    int oldValue, newValue;
    do {
        oldValue = value;
        newValue = oldValue + 1;
    } while (!unsafe.compareAndSwapInt(this, valueOffset, oldValue, newValue));
    return newValue;
}
```

### 3. 在JDK1.6以下，因Cache Line导致的伪共享问题， 如果要写CAS时，注意需要以下写法。 在JDK1.8以上，可以使用@Contended注解
```java
class LhsPadding
{
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding
{
    protected volatile long value;
}

class RhsPadding extends Value
{
    protected long p9, p10, p11, p12, p13, p14, p15;
}
```

```java
@sun.misc.Contended 
static final class Cell {
    volatile long value;
}
```

> Cache Line 默认是64字节

### 4. double、float的比较的正确使用方法
```java
Double.compare(d, 0.0);
new Double(d1).compareTo(new Double(d2)) > 0; //d1是否大于d2

Float.compare(d, 0.0);
new Float(d1).compareTo(new Float(d2));
```
本质是：`long thisBits = Double.doubleToLongBits(d1);`  //将浮点数变成整数


### 5. 线程池最佳使用方式
```java
ThreadPoolExecutor producerExecutor = new ThreadPoolExecutor(10, 10, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100), 
    new ThreadFactory() {
    AtomicInteger index = new AtomicInteger(0);
        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, "producer-thread-" + index.incrementAndGet());
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy()
);

Producer producer = new Producer();
producerExecutor.execute(producer);
producerExecutor.execute(producer);
producerExecutor.shutdown();
```

### 6. Spring事务AOP的配置
```java
import org.aspectj.lang.annotation.Aspect;
import org.springframework.aop.Advisor;
import org.springframework.aop.aspectj.AspectJExpressionPointcut;
import org.springframework.aop.support.DefaultPointcutAdvisor;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.interceptor.*;

import javax.annotation.Resource;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

/**
 * 事务管理的配置
 * @author sunchangtan
 * @date 2018/8/30 11:22
 */

@Aspect
@Configuration
public class TransactionManagerConfigurer {
    private static final int TX_METHOD_TIMEOUT = 50000;

    private static final String AOP_POINTCUT_EXPRESSION = "execution(* com.sample.***.service..*.*(..))";

    @Resource
    private PlatformTransactionManager transactionManager;
    /**
     * 事务的实现Advice
     * @return
     */
    @Bean
    public TransactionInterceptor txAdvice() {
        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();

        RuleBasedTransactionAttribute readOnlyTx = new RuleBasedTransactionAttribute();
        readOnlyTx.setReadOnly(true);
        //使用PROPAGATION_SUPPORTS：支持当前事务，如果当前没有事务，就以非事务方式执行。 如果查询中出现异常，那么当前事务也可以回滚
        readOnlyTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_SUPPORTS);

        RuleBasedTransactionAttribute requiredTx = new RuleBasedTransactionAttribute();
        requiredTx.setRollbackRules(Collections.singletonList(new RollbackRuleAttribute(Exception.class)));
        //使用PROPAGATION_REQUIRED：如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。 如果需要数据库增删改，必须要使用事务
        requiredTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        requiredTx.setTimeout(TX_METHOD_TIMEOUT);

        Map<String, TransactionAttribute> txMap = new HashMap<>();
        txMap.put("add*", requiredTx);
        txMap.put("save*", requiredTx);
        txMap.put("insert*", requiredTx);
        txMap.put("update*", requiredTx);
        txMap.put("delete*", requiredTx);
        txMap.put("remove*", requiredTx);
        txMap.put("upload*", requiredTx);
        txMap.put("generate*", requiredTx);
        txMap.put("import*", requiredTx);
        txMap.put("bind*", requiredTx);
        txMap.put("unbind*", requiredTx);
        txMap.put("cancel*", requiredTx);
        txMap.put("send*", requiredTx);
        txMap.put("create*", requiredTx);
        txMap.put("compute*", requiredTx);
        txMap.put("recompute*", requiredTx);
        txMap.put("execute*", requiredTx);
        txMap.put("submit*", requiredTx);

        txMap.put("get*", readOnlyTx);
        txMap.put("query*", readOnlyTx);
        txMap.put("list*", readOnlyTx);
        txMap.put("has*", readOnlyTx);
        txMap.put("exist*", readOnlyTx);
        txMap.put("download*", readOnlyTx);
        txMap.put("export*", readOnlyTx);
        txMap.put("search*", readOnlyTx);
        txMap.put("check*", readOnlyTx);
        txMap.put("load*", readOnlyTx);
        txMap.put("find*", readOnlyTx);
        source.setNameMap(txMap);
        return new TransactionInterceptor(transactionManager, source);
    }

    /**
     * 切面的定义,pointcut及advice
     * @param txAdvice
     * @return
     */
    @Bean
    public Advisor txAdviceAdvisor(@Qualifier("txAdvice") TransactionInterceptor txAdvice) {
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression(AOP_POINTCUT_EXPRESSION);
        return new DefaultPointcutAdvisor(pointcut, txAdvice);
    }
}

```

### 6. 线程池配置合理线程数
CPU密集型任务配置尽可能少的线程数量：
一般公式：CPU核数+1个线程的线程数

IO密集型不是一直在执行任务，则尽可能多的线程数量
参考公式：CPU核数 / 1 - 阻塞系数      阻塞系统在0.8~0.9之间
比如：8核 / 1 - 0.9 


### 7. web服务器jvm参数配置
参考： https://blog.csdn.net/boonya/article/details/69230214

-server//服务器模式
-Xmx2g //JVM最大允许分配的堆内存，按需分配
-Xms2g //JVM初始分配的堆内存，一般和Xmx配置成一样以避免每次gc后JVM重新分配内存。
-Xmn256m //年轻代内存大小，整个JVM内存=年轻代 + 年老代 + 持久代
-XX:PermSize=128m //持久代内存大小
-Xss256k //设置每个线程的堆栈大小
-XX:+DisableExplicitGC //忽略手动调用GC, System.gc()的调用就会变成一个空调用，完全不触发GC
-XX:+UseConcMarkSweepGC //并发标记清除（CMS）收集器
-XX:+CMSParallelRemarkEnabled //降低标记停顿
-XX:+UseCMSCompactAtFullCollection //在FULL GC的时候对年老代的压缩
-XX:LargePageSizeInBytes=128m //内存页的大小
-XX:+UseFastAccessorMethods //原始类型的快速优化
-XX:+UseCMSInitiatingOccupancyOnly //使用手动定义初始化定义开始CMS收集
-XX:CMSInitiatingOccupancyFraction=70 //使用cms作为垃圾回收使用70％后开始CMS收集

### 8. 数据库连接池大小
参考：https://mp.weixin.qq.com/s/n1iE63B0N-vsXnro_NZzJw
https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing

连接数计算公式
连接数 = ((核心数 * 2) + 有效磁盘数)
比如：如果服务器 CPU 是 4核 i7 的，连接池大小应该为 ((4*2)+1)=9。取个整, 设置为 10。