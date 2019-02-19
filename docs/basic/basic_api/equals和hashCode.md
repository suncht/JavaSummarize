# 关于equals和hashcode方法

## 一、为什么在重写equals时还必须重写hashcode方法

首先我们先来看下String类的源码：可以发现String是重写了Object类的equals方法的，并且也重写了hashcode方法
```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}


public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

那为什么在重写equals方法时都要重写equals方法呢：
首先equals与hashcode间的关系是这样的：
1. 如果两个对象相同（即用equals比较返回true），那么它们的hashCode值一定要相同；
2. 如果两个对象的hashCode相同，它们并不一定相同(即用equals比较返回false)   

自我的理解：由于为了提高程序的效率才实现了hashcode方法，先进行hashcode的比较，如果不同，那没就不必在进行equals的比较了，这样就大大减少了equals比较的次数，这对比需要比较的数量很大的效率提高是很明显的，一个很好的例子就是在集合中的使用；

> 我们都知道java中的List集合是有序的，因此是可以重复的，而set集合是无序的，因此是不能重复的，那么怎么能保证不能被放入重复的元素呢，但靠equals方法一样比较的话，如果原来集合中以后又10000个元素了，那么放入10001个元素，难道要将前面的所有元素都进行比较，看看是否有重复，这个效率可想而知，因此hashcode就应遇而生了，java就采用了hash表，利用哈希算法（也叫散列算法），就是将对象数据根据该对象的特征使用特定的算法将其定义到一个地址上，那么在后面定义进来的数据只要看对应的hashcode地址上是否有值，那么就用equals比较，如果没有则直接插入，只要就大大减少了equals的使用次数，执行效率就大大提高了。

继续上面的话题，为什么必须要重写hashcode方法，其实简单的说就是为了保证同一个对象，保证在equals相同的情况下hashcode值必定相同，如果重写了equals而未重写hashcode方法，可能就会出现两个没有关系的对象equals相同的（因为equal都是根据对象的特征进行重写的），但hashcode确实不相同的。

## 二、equals规范
1. 自反性 ： x.equals(x) 结果应该返回true。
2. 对称性 ： x.equals(y) 结果返回true当且仅当y.equals(x)也应该返回true。
3. 传递性 ： x.equals(y) 返回true，并且y.equals(z) 返回true，那么x.equals(z) 也应该返回true。
4. 一致性 ： x.equals(y)的第一次调用为true，那么x.equals(y)的第二次，第三次等多次调用也应该为true，但是前提条件是在进行比较之前，x和y都没有被修改。
5. x.equals(null) 应该返回false。
这个方法返回true当且仅当x和y指向了同样的对象(x==y)，这句话也就是说明了在默认情况下，Object类中的equals方法默认比较的是对象的地址，因为只有是相同的地址才会相等(x == y)，如果没有重写equals方法，那么默认就是比较的是地址。
注意：无论何时这个equals方法被重写那么都是有必要去重写hashCode方法，这个是因为为了维持hashCode的一种约定，相同的对象必须要有相同的hashCode值。

## 三、hashCode规范
1. 在同一次的java程序应用过程中，对应同样的对象多次调用hashCode方法，hashCode方法必须一致性的返回同样的一个地址值，前提是这个对象不能改变
2. 两个对象相同是依据equals方法来的，那么其中的每一个对象调用hashCode方法都必须返回相同的一个integer值，也就是对象的地址。equals方法相等，那么hashCode方法也必须相等。
3. 如果两个对象依据equals方法返回的结果不相等，那么对于其中的每一个对象调用hashCode方法返回的结果也不是一定必须得相等（也就是说，equals方法的结果为false，那么hashCode方法返回的结果可以相同也可以不相同），但是，对于我们开发者来说，针对两个对象的不相等如果生成相同的hashCode则可以提高应用程序的性能。

hashCode方法的一致约定要求。
1. 在某个运行时期间，只要对象的（字段的）变化不会影响equals方法的决策结果，那么，在这个期间，无论调用多少次hashCode，都必须返回同一个散列码。

2. 通过equals调用返回true 的2个对象的hashCode一定一样。

3. 通过equasl返回false 的2个对象的散列码不需要不同，也就是他们的hashCode方法的返回值允许出现相同的情况。

**总结一句话：等价的(调用equals返回true)对象必须产生相同的散列码。不等价的对象，不要求产生的散列码不相同。**
