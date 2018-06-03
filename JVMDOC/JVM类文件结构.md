# JAVA 类文件结构

 ## 1. 类文件的整体结构



Java文件经过编译后，以16进制的形式严格按照JVM定义的规范，存放在class文件中。

class文件格式：

| 类型           | 名称                | 数量                  |
| -------------- | ------------------- | --------------------- |
| u4             | magic               | 1                     |
| u2             | minor_version       | 1                     |
| u2             | major_version       | 1                     |
| u2             | constant_pool_count | 1                     |
| cp_info        | constant_pool       | constant_pool_count-1 |
| u2             | access_flags        | 1                     |
| u2             | this_class          | 1                     |
| u2             | super_class         | 1                     |
| u2             | interfaces_count    | 1                     |
| u2             | interfaces          | interfaces_count      |
| u2             | fields_count        | 1                     |
| field_info     | fields              | field_count           |
| u2             | methods_count       | 1                     |
| method_info    | methods             | methods_count         |
| u2             | attributes_count    | 1                     |
| attribute_info | attributes          | attributes_count      |

* class文件格式中的**类型**叫做**无符号数**，用来记录数据项的大小。它是一种基本数据类型，以u1、u2、u4、u8分别代表1个字节、2个字节、4个字节、8个字节。如：magic对应的类型u4，表示在class文件中占用4个字节(即文件的开头4个字节就是magic的内容)。
* 在class文件中，当同一类型的数据项有多个时，会前置一个容量计数器来表示数据项的个数，然后在该计数器的后面连续排列具体的数据项。如constant_pool(常量池)就是一个相同数据项的集合，在它前面有constant_pool_count来表示常量池的大小。
* 每个类文件都是严格按照上述表格形式、顺序来存放编译后的数据的。



## 2. mgic和class文件版本



每个文件的开头4个字节称为魔数。它唯一的作用是确定这个文件是否是虚拟机能够接受的class文件。

















