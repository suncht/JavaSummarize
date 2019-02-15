# CountDownLatch实现原理
## 一、前言
CountDownLatch用于同步一个或多个任务，强制他们等待由其他任务执行的一组操作完成。CountDownLatch典型的用法是将一个程序分为n个互相独立的可解决任务，并创建值为n的CountDownLatch。当每一个任务完成时，都会在这个锁存器上调用countDown，等待问题被解决的任务调用这个锁存器的await，将他们自己拦住，直至锁存器计数结束。
CountDownLatch主要有两个方法：countDown()和await()。countDown()方法用于使计数器减一，其一般是执行任务的线程调用，await()方法则使调用该方法的线程处于等待状态，其一般是主线程调用。这里需要注意的是，countDown()方法并没有规定一个线程只能调用一次，当同一个线程调用多次countDown()方法时，每次都会使计数器减一；另外，await()方法也并没有规定只能有一个线程执行该方法，如果多个线程同时执行await()方法，那么这几个线程都将处于等待状态，并且以共享模式享有同一个锁。
看下 Doug Lea 在 java doc 中给出的例子，这个例子非常实用
```java
class Driver2 { // ...
    void main() throws InterruptedException {
        CountDownLatch doneSignal = new CountDownLatch(N);
        Executor e = Executors.newFixedThreadPool(8);

        // 创建 N 个任务，提交给线程池来执行
        for (int i = 0; i < N; ++i) // create and start threads
            e.execute(new WorkerRunnable(doneSignal, i));

        // 等待所有的任务完成，这个方法才会返回
        doneSignal.await();           // wait for all to finish
    }
}

class WorkerRunnable implements Runnable {
    private final CountDownLatch doneSignal;
    private final int i;

    WorkerRunnable(CountDownLatch doneSignal, int i) {
        this.doneSignal = doneSignal;
        this.i = i;
    }

    public void run() {
        try {
            doWork(i);
            // 这个线程的任务完成了，调用 countDown 方法
            doneSignal.countDown();
        } catch (InterruptedException ex) {
        } // return;
    }

    void doWork() { ...}
}
```
 CountDownLatch 非常实用，我们常常会将一个比较大的任务进行拆分，然后开启多个线程来执行，等所有线程都执行完了以后，再往下执行其他操作。这里例子中，只有 main 线程调用了 await 方法。
 再来看另一个例子，这个例子很典型，用了两个 CountDownLatch：
 ```java
 class Driver { // ...
    void main() throws InterruptedException {
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(N);

        for (int i = 0; i < N; ++i) // create and start threads
            new Thread(new Worker(startSignal, doneSignal)).start();

        // 这边插入一些代码，确保上面的每个线程先启动起来，才执行下面的代码。
        doSomethingElse();            // don't let run yet
        // 因为这里 N == 1，所以，只要调用一次，那么所有的 await 方法都可以通过
        startSignal.countDown();      // let all threads proceed
        doSomethingElse();
        // 等待所有任务结束
        doneSignal.await();           // wait for all to finish
    }
}

class Worker implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;

    Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
    }

    public void run() {
        try {
            // 为了让所有线程同时开始任务，我们让所有线程先阻塞在这里
            // 等大家都准备好了，再打开这个门栓
            startSignal.await();
            doWork();
            doneSignal.countDown();
        } catch (InterruptedException ex) {
        } // return;
    }

    void doWork() { ...}
}
 ```
 N 个新开启的线程都调用了startSignal.await() 进行阻塞等待，它们阻塞在栅栏上，只有当条件满足的时候（startSignal.countDown()），它们才能同时通过这个栅栏。

## 二、CountDownLatch数据结构
从源码可知，其底层是由AQS提供支持，所以其数据结构可以参考AQS的数据结构，而AQS的数据结构核心就是两个虚拟队列：同步队列sync queue 和条件队列condition queue，不同的条件会有不同的条件队列。读者可以参考之前介绍的AQS。
## 三、源码分析
1. 内部类Sync
CountDownLatch类存在一个内部类Sync，继承自AbstractQueuedSynchronizer，其源代码如下。
```java
private static final class Sync extends AbstractQueuedSynchronizer {
    // 版本号
    private static final long serialVersionUID = 4982264981922014374L;
    
    // 构造器
    Sync(int count) {
        setState(count);
    }
    
    // 返回当前计数
    int getCount() {
        return getState();
    }

    // 试图在共享模式下获取对象状态
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    // 试图设置状态来反映共享模式下的一个释放
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        // 无限循环
        for (;;) {
            // 获取状态
            int c = getState();
            if (c == 0) // 没有被线程占有
                return false;
            // 下一个状态
            int nextc = c-1;
            if (compareAndSetState(c, nextc)) // 比较并且设置成功
                return nextc == 0;
        }
    }
}
```
说明：对CountDownLatch方法的调用会转发到对Sync或AQS的方法的调用，所以，AQS对CountDownLatch提供支持。

2. 构造函数
CountDownLatch(int) 型构造函数　　
```java
 public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    // 初始化状态数
    this.sync = new Sync(count);
}
```
说明：该构造函数可以构造一个用给定计数初始化的CountDownLatch，并且构造函数内完成了sync的初始化，并设置了状态数。

3. 核心函数分析
    3.1 await函数:
　　    此函数将会使当前线程在锁存器倒计数至零之前一直等待，除非线程被中断。其源码如下　　
    ```java
    public void await() throws InterruptedException {
        // 转发到sync对象上
        sync.acquireSharedInterruptibly(1);
    }
    ```
    说明：由源码可知，对CountDownLatch对象的await的调用会转发为对Sync的acquireSharedInterruptibly（从AQS继承的方法）方法的调用，acquireSharedInterruptibly源码如下　　
    ```java
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
    ```
    说明：从源码中可知，acquireSharedInterruptibly又调用了CountDownLatch的内部类Sync的tryAcquireShared和AQS的doAcquireSharedInterruptibly函数。tryAcquireShared函数的源码如下　
    ```java
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
    ```
    说明：该函数只是简单的判断AQS的state是否为0，为0则返回1，不为0则返回-1。doAcquireSharedInterruptibly函数的源码如下　　
    ```java
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        // 添加节点至等待队列
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) { // 无限循环
                // 获取node的前驱节点
                final Node p = node.predecessor();
                if (p == head) { // 前驱节点为头结点
                    // 试图在共享模式下获取对象状态
                    int r = tryAcquireShared(arg);
                    if (r >= 0) { // 获取成功
                        // 设置头结点并进行繁殖
                        setHeadAndPropagate(node, r);
                        // 设置节点next域
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt()) // 在获取失败后是否需要禁止线程并且进行中断检查
                    // 抛出异常
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    ```
    说明：在AQS的doAcquireSharedInterruptibly中可能会再次调用CountDownLatch的内部类Sync的tryAcquireShared方法和AQS的setHeadAndPropagate方法。setHeadAndPropagate方法源码如下。　　
    ```java
     private void setHeadAndPropagate(Node node, int propagate) {
        // 获取头结点
        Node h = head; // Record old head for check below
        // 设置头结点
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        // 进行判断
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            // 获取节点的后继
            Node s = node.next;
            if (s == null || s.isShared()) // 后继为空或者为共享模式
                // 以共享模式进行释放
                doReleaseShared();
        }
    }
    ```
    说明：该方法设置头结点并且释放头结点后面的满足条件的结点，该方法中可能会调用到AQS的doReleaseShared方法，其源码如下。
    ```java
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        // 无限循环
        for (;;) {
            // 保存头结点
            Node h = head;
            if (h != null && h != tail) { // 头结点不为空并且头结点不为尾结点
                // 获取头结点的等待状态
                int ws = h.waitStatus; 
                if (ws == Node.SIGNAL) { // 状态为SIGNAL
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) // 不成功就继续
                        continue;            // loop to recheck cases
                    // 释放后继结点
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) // 状态为0并且不成功，继续
                    continue;                // loop on failed CAS
            }
            if (h == head) // 若头结点改变，继续循环  
                break;
        }
    }
    ```
    　说明：该方法在共享模式下释放

    3.2 countDown函数
    此函数将递减锁存器的计数，如果计数到达零，则释放所有等待的线程　
    ```java
    public void countDown() {
        sync.releaseShared(1);
    }
    ```
    说明：对countDown的调用转换为对Sync对象的releaseShared（从AQS继承而来）方法的调用。releaseShared源码如下　
    ```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
    ```
    说明：此函数会以共享模式释放对象，并且在函数中会调用到CountDownLatch的tryReleaseShared函数，并且可能会调用AQS的doReleaseShared函数，其中，tryReleaseShared源码如下　　
    ```java
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        // 无限循环
        for (;;) {
            // 获取状态
            int c = getState();
            if (c == 0) // 没有被线程占有
                return false;
            // 下一个状态
            int nextc = c-1;
            if (compareAndSetState(c, nextc)) // 比较并且设置成功
                return nextc == 0;
        }
    }
    ```
    说明：此函数会试图设置状态来反映共享模式下的一个释放。
    AQS的doReleaseShared的源码如下:
    ```java
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        // 无限循环
        for (;;) {
            // 保存头结点
            Node h = head;
            if (h != null && h != tail) { // 头结点不为空并且头结点不为尾结点
                // 获取头结点的等待状态
                int ws = h.waitStatus; 
                if (ws == Node.SIGNAL) { // 状态为SIGNAL
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) // 不成功就继续
                        continue;            // loop to recheck cases
                    // 释放后继结点
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) // 状态为0并且不成功，继续
                    continue;                // loop on failed CAS
            }
            if (h == head) // 若头结点改变，继续循环  
                break;
        }
    }
    ```
## 四、示例
下面给出了一个使用CountDownLatch的示例。　
```java
import java.util.concurrent.CountDownLatch;

class MyThread extends Thread {
    private CountDownLatch countDownLatch;
    
    public MyThread(String name, CountDownLatch countDownLatch) {
        super(name);
        this.countDownLatch = countDownLatch;
    }
    
    public void run() {
        System.out.println(Thread.currentThread().getName() + " doing something");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " finish");
        countDownLatch.countDown();
    }
}

public class CountDownLatchDemo {
    public static void main(String[] args) {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        MyThread t1 = new MyThread("t1", countDownLatch);
        MyThread t2 = new MyThread("t2", countDownLatch);
        t1.start();
        t2.start();
        System.out.println("Waiting for t1 thread and t2 thread to finish");
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }            
        System.out.println(Thread.currentThread().getName() + " continue");        
    }
}
```
可能运行结果：
```text
Waiting for t1 thread and t2 thread to finish
t1 doing something
t2 doing something
t1 finish
t2 finish
main continue
```

### 参考文章
1. http://www.cnblogs.com/leesf456/p/5406191.html
2. https://javadoop.com/post/AbstractQueuedSynchronizer-3
3. https://www.jianshu.com/p/128476015902