# 使用Object.wait/notify实现生产者和消费者
## 第一种模式
直接使用外部的ArrayList或者其他集合作为共享对象
由生产者线程实现生产的功能
有消费者线程实现消费的功能

1. 生产者代码：
```java
/**
 * @author sunchangtan
 * @date 2019/2/25 12:59
 */
public class Producer implements Runnable {
    private final List<Goods> goods;
    private int maxCapiblity;

    public Producer(List<Goods> goods, int maxCapiblity) {
        this.goods = goods;
        this.maxCapiblity = maxCapiblity;
    }

    @Override
    public void run() {
        while (true) {
            synchronized (goods) { //要使用goods或者全局锁对象，不能用this，因为对象头中的monitor必须要相同一个
                while (goods.size() >= maxCapiblity) { // 必须使用while，避免线程假唤醒，因为wait操作有三个步骤，而且wait会释放锁，需要理解假唤醒的原因
                    try {
                        System.out.println("仓库已满，等待消费");
                        goods.wait();  //需要同一个monitor，否则将会线程一直waiting
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                Goods newGoods = this.produceNewGoods();
                this.goods.add(newGoods);
                System.out.println("生产商品" + newGoods.getGoodsId());
                goods.notifyAll();  //需要同一个monitor，否则将会该线程不可能会唤醒其他Consumer线程。需要理解monitor作用
            }
        }
    }

    private Goods produceNewGoods() {
        Goods goods = new Goods();
        goods.setGoodsId(UUID.randomUUID().toString());
        goods.setGoodsName(goods.getGoodsId());
        goods.setPrice(new BigDecimal(10));
        goods.setQuantity(1);
        return goods;
    }
}
```

2. 消费者代码
```java
public class Consumer implements Runnable {
    private final List<Goods> goods;

    public Consumer(List<Goods> goods) {
        this.goods = goods;
    }

    @Override
    public void run() {
        while (true) {
            synchronized (goods) { //要使用goods或者全局锁对象，不能用this，因为对象头中的monitor必须要相同一个。 Consumer和Producer操作goods不在一起
                while (goods.size() == 0) { // 必须使用while，避免线程假唤醒，因为wait操作有三个步骤，而且wait会释放锁，需要理解假唤醒的原因
                    try {
                        System.out.println("没有商品，等待生产");
                        goods.wait();  //需要同一个monitor
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                Goods goodsToConsume = this.goods.remove(0);
                System.out.println("消费商品" + goodsToConsume.getGoodsId());
                goods.notifyAll(); //需要同一个monitor，否则将会该线程不可能会唤醒其他Producer线程。需要理解monitor作用
            }
        }

    }
}
```

3. 测试代码：
```java
/**
 * @author sunchangtan
 * @date 2019/2/25 09:47
 */
public class AwaitAndNotifyTest {
    private List<Goods> goodsList = Lists.newLinkedList();

    public static void main(String[] args) {
        ThreadPoolExecutor consumerExecutor = new ThreadPoolExecutor(10, 10,
                60L, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(100), r -> new Thread(r, "consumer"));
        ThreadPoolExecutor producerExecutor = new ThreadPoolExecutor(10, 10,
                60L, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(100), r -> new Thread(r, "producer"));

        List<Goods> goods = Lists.newLinkedList();

        Producer producer = new Producer(goods, 1000);
        producerExecutor.execute(producer);
        producerExecutor.execute(producer);


        Consumer consumer = new Consumer(goods);
        consumerExecutor.execute(consumer);
        consumerExecutor.execute(consumer);
        consumerExecutor.execute(consumer);
        consumerExecutor.execute(consumer);
        consumerExecutor.execute(consumer);

        producerExecutor.shutdown();
        consumerExecutor.shutdown();

    }
}
```

## 第二种模式
自定义共享对象集合，在共享对象集合中，实现增加对象的功能和消费对象的功能

1. 共享对象集合
```java
/**
 * @author sunchangtan
 * @date 2019/2/26 14:03
 */
public class GoodsWarehouse {
    //环形数组
    private int maxCapacity;
    private Goods[] goodsArr;
    private int head;
    private int tail;

    public GoodsWarehouse(int maxCapacity) {
        this.maxCapacity = maxCapacity;
        goodsArr = new Goods[this.maxCapacity];
        head = 0;
        tail = 0;
    }

    /**
     * 向GoodsWarehouse生产商品
     * @param newGoods
     */
    public void offer(Goods newGoods) {
        synchronized (this) { //要可以用this，因为GoodsWarehouse对象头中的monitor可以生产和消费共享
            while ((tail > head) ? (tail - head + 1 >= maxCapacity) : (head - tail - 1 >= 0)) { //使用循环数组，减少内存分配，而ArrayList是需要拷贝的
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }


            this.goodsArr[tail++] = newGoods;
            if(tail == maxCapacity - 1) {
                tail = 0;
            }
            this.notifyAll(); //要可以用this，因为GoodsWarehouse对象头中的monitor可以生产和消费共享的
        }
    }

    /**
     * 从GoodsWarehouse中消费商品
     * @return
     */
    public Goods take() {
        synchronized (this) {
            while (tail == head) {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            Goods goods = this.goodsArr[head++];
            if(head == maxCapacity - 1) {
                head = 0;
            }
            this.notifyAll();
            return goods;
        }
    }
}
```

2. 生产者代码：
```java
public class Producer implements Runnable {
    private GoodsWarehouse goodsWarehouse;

    public Producer(GoodsWarehouse goodsWarehouse) {
        this.goodsWarehouse = goodsWarehouse;
    }

    @Override
    public void run() {
        while (true) {
            Goods newGoods = new Goods();
            newGoods.setGoodsId(UUID.randomUUID().toString());

            goodsWarehouse.offer(newGoods);

            System.out.println(Thread.currentThread().getName() + "生产商品: " + newGoods.getGoodsId());


            try {
                Thread.sleep((long) (Math.random() * 1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

3. 消费者代码
```java
/**
 * @author sunchangtan
 * @date 2019/2/26 15:55
 */
public class Consumer implements Runnable {
    private GoodsWarehouse goodsWarehouse;

    public Consumer(GoodsWarehouse goodsWarehouse) {
        this.goodsWarehouse = goodsWarehouse;
    }


    @Override
    public void run() {
        while (true) {
            Goods goods = this.goodsWarehouse.take();

            System.out.println(Thread.currentThread().getName() + "消费商品: " + goods.getGoodsId());

            try {
                Thread.sleep((long) (Math.random() * 1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

4. 测试代码
```java
/**
 * @author sunchangtan
 * @date 2019/2/26 15:59
 */
public class Test {
    public static void main(String[] args) {
        ThreadPoolExecutor consumerExecutor = new ThreadPoolExecutor(10, 10,
                60L, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(100), new ThreadFactory() {
                    AtomicInteger number = new AtomicInteger(0);
                    @Override
                    public Thread newThread(Runnable r) {
                        return new Thread(r, "consumer_" + number);
                    }
            });

        ThreadPoolExecutor producerExecutor = new ThreadPoolExecutor(10, 10,
                60L, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(100), r -> new Thread(r, "producer"));

        GoodsWarehouse goodsWarehouse = new GoodsWarehouse(100);

        Producer producer = new Producer(goodsWarehouse);
        producerExecutor.execute(producer);
        producerExecutor.execute(producer);


        Consumer consumer = new Consumer(goodsWarehouse);
        consumerExecutor.execute(consumer);
        consumerExecutor.execute(consumer);
        consumerExecutor.execute(consumer);
        consumerExecutor.execute(consumer);
        consumerExecutor.execute(consumer);

        producerExecutor.shutdown();
        consumerExecutor.shutdown();

    }
}
```


## 通用对象
```java
/**
 * @author sunchangtan
 * @date 2019/2/25 09:48
 */
@Data
public class Goods {
    private String goodsId;
    private String goodsName;
    private Integer quantity;
    private BigDecimal price;
}

```