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



