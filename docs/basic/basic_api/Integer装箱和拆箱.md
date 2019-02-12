# Integer的装箱和拆箱
## 什么是装箱？什么是拆箱？
从Java SE5开始就提供了自动装箱的特性，如果要生成一个数值为10的Integer对象，只需要这样就可以了：
```java
Integer i = 10;
```
这个过程中会自动根据数值创建对应的 Integer对象，这就是装箱。

那什么是拆箱呢？顾名思义，跟装箱对应，就是自动将包装器类型转换为基本数据类型：
```java
Integer i = 10;  //装箱
int n = i;   //拆箱
```
简单一点说，装箱就是  自动将基本数据类型转换为包装器类型；拆箱就是  自动将包装器类型转换为基本数据类型。

下表是基本数据类型对应的包装器类型：

基本类型 | 类
---------|----------
 int（4字节） | Integer 
 byte（1字节） | Byte  
 short（2字节） | Short 
 long（8字节） | Long 
 float（4字节） | Float
 double（8字节）| Double
 char（2字节）| Character
 boolean（1字节）| Boolean

## 装箱和拆箱是如何实现的
以Interger类为例，下面看一段代码：
```java
public class Main {
    public static void main(String[] args) {
        Integer i = 10;
        int n = i;
    }
}
```
反编译class文件之后得到如下内容：
![](https://github.com/suncht/JavaSummarize/raw/master/docs/basic/basic_api/images/integer.jpg)

从反编译得到的字节码内容可以看出，在装箱的时候自动调用的是Integer的valueOf(int)方法。而在拆箱的时候自动调用的是Integer的intValue方法。

## Integer\Long\Short\Char
先从一个代码入手：
```java
public class Main {
    public static void main(String[] args) {
         
        Integer i1 = 100;
        Integer i2 = 100;
        Integer i3 = 200;
        Integer i4 = 200;
         
        System.out.println(i1==i2); //true
        System.out.println(i3==i4); //false
    }
}
```
事实上输出结果是：
true
false

为什么会出现这样的结果？输出结果表明i1和i2指向的是同一个对象，而i3和i4指向的是不同的对象。此时只需一看源码便知究竟，下面这段代码是Integer的valueOf方法的具体实现：
```java
public static Integer valueOf(int i) {
    if(i >= -128 && i <= IntegerCache.high)
        return IntegerCache.cache[i + 128];
    else
        return new Integer(i);
}
```
而其中IntegerCache类的实现为：
```java
private static class IntegerCache {
    static final int high;
    static final Integer cache[];

    static {
        final int low = -128;

        // high value may be configured by property
        int h = 127;
        if (integerCacheHighPropValue != null) {
            // Use Long.decode here to avoid invoking methods that
            // require Integer's autoboxing cache to be initialized
            int i = Long.decode(integerCacheHighPropValue).intValue();
            i = Math.max(i, 127);
            // Maximum array size is Integer.MAX_VALUE
            h = Math.min(i, Integer.MAX_VALUE - -low);
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);
    }

    private IntegerCache() {}
}
```
从这2段代码可以看出，在通过valueOf方法创建Integer对象的时候，如果数值在[-128,127]之间，便返回指向IntegerCache.cache中已经存在的对象的引用；否则创建一个新的Integer对象。

　　上面的代码中i1和i2的数值为100，因此会直接从cache中取已经存在的对象，所以i1和i2指向的是同一个对象，而i3和i4则是分别指向不同的对象。

Integer、Short、Byte、Character、Long这几个类的valueOf方法的实现是类似的。

## Double\Float
先从一段代码开始：
```java
public class Main {
    public static void main(String[] args) {
         
        Double i1 = 100.0;
        Double i2 = 100.0;
        Double i3 = 200.0;
        Double i4 = 200.0;
         
        System.out.println(i1==i2); //false
        System.out.println(i3==i4); //false
    }
}
```
实际输出结果为：
false
false

至于具体为什么，Double类的valueOf的实现:
```java
public static Double valueOf(double d) {
    return new Double(d);
}
```
从源码看出，Double没有缓存的，Float也类似。

