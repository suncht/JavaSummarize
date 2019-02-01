# HashCode一点思考
1. String字符串的hashcode
    ```java
    String str1 = "aaa";
    String str2 = new String("aaa");
    System.out.println(str1.hashCode() == str2.hashCode());
    ```
    结果：true
    结论：根据String类包含的字符串的内容，根据一种特殊算法返回哈希码，只要字符串的内容相同，返回的哈希码也相同。
    String的hashCode方法：
    ```java
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
    从源码中可见，计算公式s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
    字符串中每个字符值都参与计算

2. Integer等包装类的hashCode
    ```java
    Integer int1 = new Integer(100);
    System.out.println(int1.hashCode());
    ```
    结果： 100
    结论：返回的哈希码就是Integer对象里所包含的那个整数的数值，例如Integer i1=new Integer(100)，i1.hashCode的值就是100 。由此可见，2个一样大小的Integer对象，返回的哈希码也一样。
3. Object类的hashCode
    返回对象的经过处理后的内存地址，由于每个对象的内存地址都不一样，所以哈希码也不一样。这个是native方法，取决于JVM的内部设计，一般是某种C地址的偏移。