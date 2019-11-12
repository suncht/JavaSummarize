## jvm参数类型
标准参数：
* -version  -showversion
* -help
* -server  -client
* -cp -classpath
X参数：
* -Xint：解释执行
* -Xcomp：第一次使用就编译成本地代码
* -Xmixed：混合模式，JVM自己来决定是否编译成本地代码
XX参数：
* Boolean类型
> -XX:+UseConcMarkSweepGC  -XX:+UseG1GC
* 非Boolean类型（KV类型）
> -XX:MaxGCPauseMillis=500   -XX:GCTimeRatio=19
* -Xms 等价于 -XX:InitialHeapSize
* -Xmx 等价于 -XX:MaxHeapSize
* -Xss 等价于 -XX:ThreadStackSize

## 查看jvm运行时参数 jinfo
jinfo -flags 进程ID  ==> 列举所有jvm运行时参数
jinfo -flag 参数名 进程ID ==> 列举指定参数的jvm运行时参数
例子：jinfo -flag MaxHeapSize 进程ID
     jinfo -flag UseG1GC
     jinfo -flag UseParallelGC
     jinfo -flag UseConcMarkSweepGC

## 查看jvm统计信息 jstat
动态查看垃圾回收情况、类加载信息、编译信息等
options: -class -compiler -gc -printcompilation
* 查看类加载信息
例子：jstat -class 进程ID 1000(每隔)
* 查看垃圾回收情况  -gc -gcutil
例子：jstat -gc 进程ID
S0C、S1C、S0U、S1U：S0和S1的总量和使用量
EC、EU：Eden区总量和使用量
OC、OU：Old区总量和使用量
MC、MU：Metaspace区总量和使用量
CCSC、CCSU：压缩类空间总量和使用量
YGC、YGCT：Young GC的次数和时间
FGC、FGCT：Full GC的次数和时间
GCT：总的GC时间

##如何导出内存映像文件 jmap
自动导出hprof文件
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=./
手动导出
jmap -dump:format=b,file=heap.hprof 进程ID

##如何定位死循环和死锁 jstack
死循环，CPU飚高
ps -ef  显示PID是十进制
printf '%x'  将PID转为16进制
jstack 进程ID > 文件名
在文件中，用16进制的PID搜索对应的行

死锁
ps -ef | grep java  获取进程ID
jstack 进程ID > 文件名

##远程监控java进程
监控远程tomcat
修改Catalina.sh参数
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9004 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.net.preferlPv4Stack=true -Djava.rmi.server.hostname=10.110.3.62"

jvisual添加远程-远程JMX


监控远程jar
运行时参数
java -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9004 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.net.preferlPv4Stack=true -Djava.rmi.server.hostname=10.110.3.62 -jar xxxxx.jar

## BTrace


## psi-probe监控tomcat
psi-probe部署tomcat，可以监控

## tomcat优化
1. 内存优化

2. 线程优化
maxConnection: 最大连接数，默认10000
acceptCount: 请求队列最大个数
maxThreads: 工作线程数，默认200
minSpareThreads：最小空闲工作线程，不要太小

3. 配置优化
autoDeploy：是否周期型检查新部署的应用，默认是false
enableLookups：是否启用DNS lookup，默认是false
reloadable：是否检查应用是否发生变化，默认是false
protocol：bio(http/1.1)、nio(Http11NioProtocol)、aio(tomcat8以上)、apr

性能比较：
参考：https://blog.csdn.net/mrleeapple/article/details/80420395

4. session优化
如果使用分布式session，可以禁用tomcat的session

## nginx优化
