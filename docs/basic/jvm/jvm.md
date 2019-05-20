## 1. Class文件格式
Class文件包含：虚拟机指令、符号表、其他辅助信息

Class文件格式如下：
格式 | 魔数 | 次版本号 | 主版本号 | 常量池计数器 | 常量池 | 访问标志 | 类索引 | 父类索引 | 接口计数器 | 接口索引集合 | 字段计数器 | 字段表集合 | 方法计数器 | 方法表集合 | 属性计数器 | 属性表集合
---------|---------|----------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------
 字节个数 | u4 | u2 | u2 | u2 | cp_info | u2 | u2 | u2 | u2 | u2 | u2 | field_info | u2 | method_info | u2 | attribute_info
 英文名 | magic | minor_version | major_version | constant_pool_count | constant_pool | access_flags | this_class | super_class | interfaces_count | interfaces | fields_count | fields | methods_count | methods | attributes_count | attributes
 个数 | 1 | 1 | 1 | 1 | constant_pool_count-1 | 1 | 1 | 1 | 1 |  interfaces_count | 1 | fields_count | 1 | methods_count | 1 | attributes_count

> 说明：
u1：1个字节
u2：2个字节
u3：3个字节
u4：4个字节

### 1.1 魔数
每个Class文件的头4个字节为模式（Magic Number），作用是判断文件是否是一个能被虚拟机接受的Class文件，值为：0XCAFEBABE(咖啡宝贝)

CA | FE | BA | BE
---------|----------|---------|---------

### 1.2 版本号
紧接魔数的4个字节是Class文件的版本号：
第5-6个字节是次版本号（Minor Version）
第7-8个字节是主版本号（Major Version）

JDK1.7： 0x0033  --> 对应十进制51
JDK1.8： 0x0034  --> 对应十进制52

### 1.3 常量池
常量池===>22 (0x16)

字面量：相当于常量概念，比如文本字符串、常量值等，例子：`int a = 3`，其中3是字面量
符号引用：类和接口的全限定名、字段的名称和描述符、方法的名称和描述符

#### 1.3.1 常量池结构表

