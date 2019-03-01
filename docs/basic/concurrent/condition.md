# Lock的Condition源码
引用：https://www.jianshu.com/p/28387056eeb4

## 1.Condition简介
任何一个java对象都天然继承于Object类，在线程间实现通信的往往会应用到Object的几个方法，比如wait(),wait(long timeout),wait(long timeout, int nanos)与notify(),notifyAll()几个方法实现等待/通知机制，同样的， 在java Lock体系下依然会有同样的方法实现等待/通知机制。从整体上来看**Object的wait和notify/notify是与对象监视器配合完成线程间的等待/通知机制，而Condition与Lock配合完成等待通知机制，前者是java底层级别的，后者是语言级别的，具有更高的可控制性和扩展性**。两者除了在使用方式上不同外，在功能特性上还是有很多的不同：

1. Condition能够支持不响应中断，而通过使用Object方式不支持；
2. Condition能够支持多个等待队列（new 多个Condition对象），而Object方式只能支持一个；
3. Condition能够支持超时时间的设置，而Object不支持

参照Object的wait和notify/notifyAll方法，Condition也提供了同样的方法

> 针对Object的wait方法

1. void await() throws InterruptedException:当前线程进入等待状态，如果其他线程调用condition的signal或者signalAll方法并且当前线程获取Lock从await方法返回，如果在等待状态中被中断会抛出被中断异常；
2. long awaitNanos(long nanosTimeout)：当前线程进入等待状态直到被通知，中断或者超时；
3. boolean await(long time, TimeUnit unit)throws InterruptedException：同第二种，支持自定义时间单位
4. boolean awaitUntil(Date deadline) throws InterruptedException：当前线程进入等待状态直到被通知，中断或者到了某个时间

> 针对Object的notify/notifyAll方法

1. void signal()：唤醒一个等待在condition上的线程，将该线程从等待队列中转移到同步队列中，如果在同步队列中能够竞争到Lock则可以从等待方法中返回。
2. void signalAll()：与1的区别在于能够唤醒所有等待在condition上的线程

## 2.Condition实现原理分析
### 2.1 等待队列
要想能够深入的掌握condition还是应该知道它的实现原理，现在我们一起来看看condition的源码。创建一个condition对象是通过`lock.newCondition()`,而这个方法实际上是会new出一个ConditionObject对象，该类是AQS（[AQS的实现原理的文章](https://www.jianshu.com/p/cc308d82cc71)）的一个内部类，有兴趣可以去看看。前面我们说过，condition是要和lock配合使用的也就是condition和Lock是绑定在一起的，而lock的实现原理又依赖于AQS，自然而然ConditionObject作为AQS的一个内部类无可厚非。我们知道在锁机制的实现上，AQS内部维护了一个同步队列，如果是独占式锁的话，所有获取锁失败的线程的尾插入到同步队列，同样的，condition内部也是使用同样的方式，内部维护了一个 等待队列，所有调用condition.await方法的线程会加入到等待队列中，并且线程状态转换为等待状态。另外注意到ConditionObject中有两个成员变量：
```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

这样我们就可以看出来ConditionObject通过持有等待队列的头尾指针来管理等待队列。主要注意的是Node类复用了在AQS中的Node类，其节点状态和相关属性可以去看AQS的实现原理的文章，如果您仔细看完这篇文章对condition的理解易如反掌，对lock体系的实现也会有一个质的提升。Node类有这样一个属性：
```java
//后继节点
Node nextWaiter;
```
进一步说明，等待队列是一个单向队列，而在之前说AQS时知道同步队列是一个双向队列。接下来我们用一个demo，通过debug进去看是不是符合我们的猜想：
```java
public static void main(String[] args) {
    for (int i = 0; i < 10; i++) {
        Thread thread = new Thread(() -> {
            lock.lock();
            try {
                condition.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                lock.unlock();
            }
        });
        thread.start();
    }
}
```
这段代码没有任何实际意义，甚至很臭，只是想说明下我们刚才所想的。新建了10个线程，没有线程先获取锁，然后调用condition.await方法释放锁将当前线程加入到等待队列中，通过debug控制当走到第10个线程的时候查看firstWaiter即等待队列中的头结点，debug模式下情景图如下：
![](../images/2019-03-01-11-19-31.png)

从这个图我们可以很清楚的看到这样几点：1. 调用condition.await方法后线程依次尾插入到等待队列中，如图队列中的线程引用依次为Thread-0,Thread-1,Thread-2....Thread-8；2. 等待队列是一个单向队列。通过我们的猜想然后进行实验验证，我们可以得出等待队列的示意图如下图所示：
![](../images/2019-03-01-11-20-21.png)
同时还有一点需要注意的是：我们可以多次调用lock.newCondition()方法创建多个condition对象，也就是一个lock可以持有多个等待队列。而在之前利用Object的方式实际上是指在对象Object对象监视器上只能拥有一个同步队列和一个等待队列，而并发包中的Lock拥有一个同步队列和多个等待队列。示意图如下：
![](../images/2019-03-01-11-21-16.png)
如图所示，ConditionObject是AQS的内部类，因此每个ConditionObject能够访问到AQS提供的方法，相当于每个Condition都拥有所属同步器的引用。

### 2.2 await实现原理
**当调用condition.await()方法后会使得当前获取lock的线程进入到等待队列，如果该线程能够从await()方法返回的话一定是该线程获取了与condition相关联的lock**。接下来，我们还是从源码的角度去看，只有熟悉了源码的逻辑我们的理解才是最深的。await()方法源码为：
```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 1. 将当前线程包装成Node，尾插入到等待队列中
    Node node = addConditionWaiter();
    // 2. 释放当前线程所占用的lock，在释放的过程中会唤醒同步队列中的下一个节点
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        // 3. 当前线程进入到等待状态
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 4. 自旋等待获取到同步状态（即获取到lock）
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 5. 处理被中断的情况
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
代码的主要逻辑请看注释，我们都知道**当当前线程调用condition.await()方法后，会使得当前线程释放lock然后加入到等待队列中，直至被signal/signalAll后会使得当前线程从等待队列中移至到同步队列中去，直到获得了lock后才会从await方法返回，或者在等待时被中断会做中断处理**。那么关于这个实现过程我们会有这样几个问题：1. 是怎样将当前线程添加到等待队列中去的？2.释放锁的过程？3.怎样才能从await方法退出？而这段代码的逻辑就是告诉我们这三个问题的答案。具体请看注释，在第1步中调用addConditionWaiter将当前线程添加到等待队列中，该方法源码为：
```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //将当前线程包装成Node
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        //尾插入
        t.nextWaiter = node;
    //更新lastWaiter
    lastWaiter = node;
    return node;
}
```
这段代码就很容易理解了，将当前节点包装成Node，如果等待队列的firstWaiter为null的话（等待队列为空队列），则将firstWaiter指向当前的Node,否则，更新lastWaiter(尾节点)即可。就是通过尾插入的方式将当前线程封装的Node插入到等待队列中即可，同时可以看出等待队列是一个不带头结点的链式队列，之前我们学习AQS时知道同步队列是一个带头结点的链式队列，这是两者的一个区别。将当前节点插入到等待对列之后，会使当前线程释放lock，由fullyRelease方法实现，fullyRelease源码为：
```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            //成功释放同步状态
            failed = false;
            return savedState;
        } else {
            //不成功释放同步状态抛出异常
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```
这段代码就很容易理解了，**调用AQS的模板方法release方法释放AQS的同步状态并且唤醒在同步队列中头结点的后继节点引用的线程，如果释放成功则正常返回，若失败的话就抛出异常。**到目前为止，这两段代码已经解决了前面的两个问题的答案了，还剩下第三个问题，怎样从await方法退出？现在回过头再来看await方法有这样一段逻辑：
```java
while (!isOnSyncQueue(node)) {
    // 3. 当前线程进入到等待状态
    LockSupport.park(this);
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
        break;
}
```
很显然，当线程第一次调用condition.await()方法时，会进入到这个while()循环中，然后通过LockSupport.park(this)方法使得当前线程进入等待状态，那么要想退出这个await方法第一个前提条件自然而然的是要先退出这个while循环，出口就只剩下两个地方：1. **逻辑走到break退出while循环；2. while循环中的逻辑判断为false**。再看代码出现第1种情况的条件是当前等待的线程被中断后代码会走到break退出，第二种情况是当前节点被移动到了同步队列中（即另外线程调用的condition的signal或者signalAll方法），while中逻辑判断为false后结束while循环。总结下，就是**当前线程被中断或者调用condition.signal/condition.signalAll方法当前节点移动到了同步队列后** ，这是当前线程退出await方法的前提条件。当退出while循环后就会调用acquireQueued(node, savedState)，这个方法在介绍AQS的底层实现时说过了，若感兴趣的话可以去看这篇文章，该方法的作用是**在自旋过程中线程不断尝试获取同步状态，直至成功（线程获取到lock）**。这样也说明了**退出await方法必须是已经获得了condition引用（关联）的lock**。到目前为止，开头的三个问题我们通过阅读源码的方式已经完全找到了答案，也对await方法的理解加深。await方法示意图如下图：
![](../images/2019-03-01-11-26-19.png)

如图，调用condition.await方法的线程必须是已经获得了lock，也就是当前线程是同步队列中的头结点。调用该方法后会使得当前线程所封装的Node尾插入到等待队列中。

....

