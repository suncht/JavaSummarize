# 一致性hash算法
## 一、前言
在大型web应用中，缓存可算是当今的一个标准开发配置了。在大规模的缓存应用中，应运而生了分布式缓存系统。在高并发（比如20W/S QPS + 海量数据）环境下，如何让请求均匀分散到集群中，防止应用数据库雪崩？

## 二、Hash取模
1. 基本原理
计算方式：hash(key) % 缓存集群节点数量

2. 使用
这个算法的问题在于容错性和扩展性不好。所谓容错性是指当系统中某一个或几个服务器变得不可用时，整个系统是否可以正确高效运行；而扩展性是指当加入新的服务器后，整个系统是否可以正确高效运行。
现假设有一台服务器宕机了，那么为了填补空缺，要将宕机的服务器从编号列表中移除，后面的服务器按顺序前移一位并将其编号值减一，此时每个key就要按h = Hash(key) % (N-1)重新计算；同样，如果新增了一台服务器，虽然原有服务器编号不用改变，但是要按h = Hash(key) % (N+1)重新计算哈希值。因此系统中一旦有服务器变更，大量的key会被重定位到不同的服务器从而造成大量的缓存不命中。而这种情况在分布式系统中是非常糟糕的。

> 如果服务器从3台扩容到4台， 将有75%的缓存命不中。

> 如果服务器从N台扩容到N+1台，将有N/(N+1)的缓存命不中。

> 如果服务器从N台宕机了1台，将(N-1)/N的缓存命不中。

3. 缺点很明显：
* 如果节点增加或者减少，缓存命不中概率的都平均在80%以上。

## 三、 一致性hash算法
1. 基本原理
节点hash值映射到Hash环上**

2. 一致性hash的特性:
- 1. 单调性(Monotonicity)，单调性是指如果已经有一些请求通过哈希分派到了相应的服务器进行处理，又有新的服务器加入到系统中时候，应保证原有的请求可以被映射到原有的或者新的服务器中去，而不会被映射到原来的其它服务器上去。 这个通过上面新增服务器ip5可以证明，新增ip5后，原来被ip1处理的user6现在还是被ip1处理，原来被ip1处理的user5现在被新增的ip5处理。

- 2. 分散性(Spread)：分布式环境中，客户端请求时候可能不知道所有服务器的存在，可能只知道其中一部分服务器，在客户端看来他看到的部分服务器会形成一个完整的hash环。如果多个客户端都把部分服务器作为一个完整hash环，那么可能会导致，同一个用户的请求被路由到不同的服务器进行处理。这种情况显然是应该避免的，因为它不能保证同一个用户的请求落到同一个服务器。所谓分散性是指上述情况发生的严重程度。好的哈希算法应尽量避免尽量降低分散性。 一致性hash具有很低的分散性

- 3. 平衡性(Balance)：平衡性也就是说负载均衡，是指客户端hash后的请求应该能够分散到不同的服务器上去。一致性hash可以做到每个服务器都进行处理请求，但是不能保证每个服务器处理的请求的数量大致相同.

3. 使用
采用一致性哈希算法的分布式集群中将新的机器加入，其原理是通过使用与对象存储一样的Hash算法将机器也映射到环中（一般情况下对机器的hash计算是采用机器的IP或者机器唯一的别名作为输入值），然后以顺时针的方向计算，将所有对象存储到离自己最近的机器中。

假设现在有NODE1，NODE2，NODE3三台机器，通过Hash算法得到对应的KEY值，映射到环中，其示意图如下：
Hash(NODE1) = KEY1;
Hash(NODE2) = KEY2;
Hash(NODE3) = KEY3;

![一致性hash](https://github.com/suncht/JavaSummarize/raw/master/docs/algorithm/images/1.png)

4. 缺点：
+ 1. 如果节点hash值在hash环上分布不均匀，导致缓存数据在每个节点不均匀分布
+ 2. 节点增加或减少，需要重新分布的缓存数据也不均匀分配
![一致性hash分布不均匀](https://github.com/suncht/JavaSummarize/raw/master/docs/algorithm/images/2.jpg)

## 四、一致性Hash + 虚拟节点
1. 基本原理：
将一个节点计算获取N个hash值，每个hash值就是一个虚拟节点hash值，然后映射到hash环中；通常情况下N=100是比较合理**

2. 使用
虚拟节点hash值计算是采用机器的IP或者机器唯一的别名 + 序号作为输入值，相对均匀映射到hash环中，这样缓存数据将相对均衡的分布在每个节点中。
如果增加或减少节点，缓存数据相对比较均匀分配到新节点。
![一致性Hash+虚拟节点](https://github.com/suncht/JavaSummarize/raw/master/docs/algorithm/images/3.png)

3. Java实现
```java
import com.google.common.base.Charsets;
import com.google.common.hash.HashFunction;
import com.google.common.hash.Hashing;

import java.util.ArrayList;
import java.util.Collection;
import java.util.SortedMap;
import java.util.TreeMap;

public class ConsistentHash<T> {

    private final HashFunction hashFunction;                      // 所用的hash函数
    private final int numberOfReplicas;                  // server虚拟节点倍数(100左右比较合理)
    private final SortedMap<Integer, T> circle = new TreeMap<Integer, T>(); // server节点分布圆

    /**
     * 初始化一致性hash算法
     *
     * @param hashFunction
     * @param numberOfReplicas
     * @param nodes            server节点集合
     */
    public ConsistentHash(HashFunction hashFunction, int numberOfReplicas, Collection<T> nodes) {
        this.hashFunction = hashFunction;
        this.numberOfReplicas = numberOfReplicas;

        for (T node : nodes) {
            add(node);
        }
    }

    /**
     * 初始化一致性hash算法
     *
     * @param hashFunction
     * @param numberOfReplicas
     */
    public ConsistentHash(HashFunction hashFunction, int numberOfReplicas) {
        this.hashFunction = hashFunction;
        this.numberOfReplicas = numberOfReplicas;

    }

    public ConsistentHash(int numberOfReplicas) {
        this.hashFunction = Hashing.md5();
        this.numberOfReplicas = numberOfReplicas;
    }

    /**
     * 加入server节点
     *
     * @param node
     */
    public void add(T node) {
        for (int i = 0; i < numberOfReplicas; i++) {
            circle.put(hashFunction.hashString(node.toString() + i, Charsets.UTF_8).asInt(), node);
        }
    }

    /**
     * 移除server节点
     *
     * @param node
     */
    public void remove(T node) {
        for (int i = 0; i < numberOfReplicas; i++) {
            circle.remove(hashFunction.hashString(node.toString() + i, Charsets.UTF_8).asInt());
        }
    }

    /**
     * 获取client对应server节点
     *
     * @param key
     * @return
     */
    public T get(Object key) {
        if (circle.isEmpty()) {
            return null;
        }

        //生成client对应的hash值
        int hash = hashFunction.hashString(key.toString(), Charsets.UTF_8).asInt();

        //如果没有对应此hash的server节点，获取大于等于此hash后面的server节点；如果还没有，则获取server节点分布圆的第一个节点
        if (!circle.containsKey(hash)) {
            SortedMap<Integer, T> tailMap = circle.tailMap(hash);
            hash = tailMap.isEmpty() ? circle.firstKey() : tailMap.firstKey();
        }
        return circle.get(hash);
    }

    public static void main(String[] args) {

        ArrayList<String> nodeList = new ArrayList<String>();
        nodeList.add("www.google.com.hk");
        nodeList.add("www.apple.com.cn");
        nodeList.add("www.twitter.com");
        nodeList.add("www.weibo.com");

        HashFunction hf = Hashing.md5();

        ConsistentHash<String> consistentHash = new ConsistentHash<String>(hf, 100, nodeList);
        //根据一致性hash算法获取客户端对应的服务器节点
        System.out.println(consistentHash.get("1111"));
    }
}
```
