# HashMap底层原理
## JDK8的HashMap

### 源码理解
1. tableSizeFor方法
```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
tableSizeFor(5) ---> 结果是8
tableSizeFor(11) ---> 结果是16
tableSizeFor方法对长度进行2幂取整，也就是HashMap长度强制为2幂，包括初始化

2. hash方法
```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
(h = key.hashCode()) ^ (h >>> 16)：高16位和低16位进行异或运算，可保证hash更加均匀分布

3. putVal方法中获取数组位置
```java
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```
(n - 1) & hash：比hash % (n-1)取余要高效

4. resize()扩容方法，用在2个地方
    + Size大于thresholds时
        ```java
        if (++size > threshold)
            resize();
        ```
    + 链表转换为红黑树时treeifyBin
        ```java
        //长度小于MIN_TREEIFY_CAPACITY=64时，尝试扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        ```



## 关于HashMap其他问题
1. HashMap可以使用null作为key，总是存储在table数组的第一个节点上
2. ConcurrentHashMap不能使用null的key和null的value
3. hash & 0x7FFFFFFF, 为了去掉hash的符号，因为hash可能会有负数
    - 0x7FFFFFFF is 0111 1111 1111 1111 1111 1111 1111 1111 : all 1 except the sign bit.
    - (hash & 0x7FFFFFFF) will result in a positive integer.
    - (hash & 0x7FFFFFFF) % tab.length will be in the range of the tab length.
4. hashmap、hashtable、ConcurrentHashMap 扩容的比较
    1. hashmap扩容
        + 默认大小： 16， 增长因子：0.75
        + 扩容点规则(什么时候扩容)： 16*0.75=12
        + 存储方式：数组+链表
        + 扩容方法：扩容为原来的2倍
        + 扩容条件：
            > 1. 判断当前个数是否大于等于阈值
            > 2. 当前存放是否发生哈希碰撞
        + 补充：​(JDK8中)如果新增节点之后，所在链表的元素个数达到了阈值 8，则会调用treeifyBin方法把链表转换成红黑树，不过在结构转换之前，会对数组长度进行判断，实现如下：
如果数组长度n小于阈值MIN_TREEIFY_CAPACITY，默认是64，则会调用resize方法把数组长度扩大到原来的两倍，重新调整节点的位置。
            ```java
            final void treeifyBin(Node<K,V>[] tab, int hash) {
                int n, index; Node<K,V> e;
                if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
                    resize();
                else if ((e = tab[index = (n - 1) & hash]) != null) {
                    TreeNode<K,V> hd = null, tl = null;
                    do {
                        TreeNode<K,V> p = replacementTreeNode(e, null);
                        if (tl == null)
                            hd = p;
                        else {
                            p.prev = tl;
                            tl.next = p;
                        }
                        tl = p;
                    } while ((e = e.next) != null);
                    if ((tab[index] = hd) != null)
                        hd.treeify(tab);
                }
            }​​
            ```
    2. hashtable扩容
        + 默认大小： 11， 增长因子：0.75
        + 扩容点规则(什么时候扩容)： 11*0.75=12
        ```java
        (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1); MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
        ```
        + 存储方式：数组+链表
        + 扩容方法：扩容为原来的2倍+1
    3. ConcurrentHashMap扩容
        + 默认大小： 
            + (JDK7)segment位16个，桶的数量是16，桶内hash表的数量是1，加载因子是0.75。
            + (JDK8)与HashMap基本相同，不同的是增加并发CAS操作
        + 扩容点规则(什么时候扩容)：
当往hashMap中成功插入一个key/value节点时，有可能触发扩容动作：
            > 1. 如果新增节点之后，所在链表的元素个数达到了阈值 8，则会调用treeifyBin方法把链表转换成红黑树，不过在结构转换之前，会对数组长度进行判断，实现如下：
            如果数组长度n小于阈值MIN_TREEIFY_CAPACITY，默认是64，则会调用tryPresize方法把数组长度扩大到原来的两倍，并触发transfer方法，重新调整节点的位置。
            > 2. 新增节点之后，会调用addCount方法记录元素个数，并检查是否需要进行扩容，当数组元素个数达到阈值时，会触发transfer方法，重新调整节点的位置。
        + 存储方式：
            + (JDK7)segment+数组+链表
            + (JDK8)数值+链表