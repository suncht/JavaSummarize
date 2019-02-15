# 一致性Hash算法实现
核心部分是采用Guava的hash算法和TreeMap的搜索二叉树功能
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
        nodeList.add("twitter.com");
        nodeList.add("weibo.com");

        HashFunction hf = Hashing.md5();

        ConsistentHash<String> consistentHash = new ConsistentHash<String>(hf, 100, nodeList);
        //根据一致性hash算法获取客户端对应的服务器节点
        System.out.println(consistentHash.get("lalala"));
    }


}
```