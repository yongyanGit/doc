# JAVA 类文件结构

 ## 1. 类文件的整体结构



Java文件经过编译后，以16进制的形，严格按照JVM定义的规范，存放在class文件中。一个完整的类文件包含有魔数、java版本信息、常量池、父类、接口、方法、成员变量等信息。如下图构成类文件的数据项：

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

* **类型**叫做**无符号数**，它是一种基本数据类型，以u1、u2、u4、u8分别代表1个字节、2个字节、4个字节、8个字节。它用来记录数据项在class文件中占用多少个字节。如：magic对应的类型u4，表示在class文件中占用4个字节(即文件的开头4个字节就是magic的内容)。
* 在class文件中，当同一类型的数据项有多个时，会前置一个容量计数器来表示数据项的个数，然后在该计数器的后面连续排列具体的数据项。如constant_pool(常量池)就是一个相同数据项的集合，在它前面有constant_pool_count来表示常量池的大小。
* 每个类文件都是严格按照上述表格形式、顺序来存放编译后的数据的。




下面就一一来解释类文件中的各个数据项。

示例代码：

```
package zookeeperdemo;
public class TestClass {
	
	private int m;
	
	public int inc(){
		return m+1;
	}

}
```

编译后的字节码

 ```
cafe babe 0000 0034 0033 0700 0201 0017
7a6f 6f6b 6565 7065 7264 656d 6f2f 5465
7374 436c 6173 7307 0004 0100 106a 6176
612f 6c61 6e67 2f4f 626a 6563 7401 0001
6d01 0001 4901 0006 3c69 6e69 743e 0100
0328 2956 0100 0443 6f64 650a 0003 000b
0c00 0700 0801 000f 4c69 6e65 4e75 6d62
6572 5461 626c 6501 0012 4c6f 6361 6c56
6172 6961 626c 6554 6162 6c65 0100 0474
6869 7301 0019 4c7a 6f6f 6b65 6570 6572
6465 6d6f 2f54 6573 7443 6c61 7373 3b01
0003 696e 6301 0003 2829 4909 0001 0013
0c00 0500 0601 0004 6d61 696e 0100 1628
5b4c 6a61 7661 2f6c 616e 672f 5374 7269
6e67 3b29 560a 0001 000b 0900 1800 1a07
0019 0100 106a 6176 612f 6c61 6e67 2f53
7973 7465 6d0c 001b 001c 0100 036f 7574
0100 154c 6a61 7661 2f69 6f2f 5072 696e
7453 7472 6561 6d3b 0a00 0100 1e0c 001f
0011 0100 0469 6e63 320a 0021 0023 0700
2201 0013 6a61 7661 2f69 6f2f 5072 696e
7453 7472 6561 6d0c 0024 0025 0100 0770
7269 6e74 6c6e 0100 0428 4929 5601 0004
6172 6773 0100 135b 4c6a 6176 612f 6c61
6e67 2f53 7472 696e 673b 0100 0663 6c61
7373 3107 002a 0100 136a 6176 612f 6c61
6e67 2f45 7863 6570 7469 6f6e 0100 0178
0100 0165 0100 154c 6a61 7661 2f6c 616e
672f 4578 6365 7074 696f 6e3b 0100 0d53
7461 636b 4d61 7054 6162 6c65 0700 3001
0013 6a61 7661 2f6c 616e 672f 5468 726f
7761 626c 6501 000a 536f 7572 6365 4669
6c65 0100 0e54 6573 7443 6c61 7373 2e6a
6176 6100 2100 0100 0300 0000 0100 0200
0500 0600 0000 0400 0100 0700 0800 0100
0900 0000 2f00 0100 0100 0000 052a b700
0ab1 0000 0002 000c 0000 0006 0001 0000
0003 000d 0000 000c 0001 0000 0005 000e
000f 0000 0002 0010 0011 0001 0009 0000
0031 0002 0001 0000 0007 2ab4 0012 0460
ac00 0000 0200 0c00 0000 0600 0100 0000
0800 0d00 0000 0c00 0100 0000 0700 0e00
0f00 0000 0900 1400 1500 0100 0900 0000
4f00 0200 0200 0000 13bb 0001 59b7 0016
4cb2 0017 2bb6 001d b600 20b1 0000 0002
000c 0000 000e 0003 0000 000d 0008 000e
0012 0010 000d 0000 0016 0002 0000 0013
0026 0027 0000 0008 000b 0028 000f 0001
0001 001f 0011 0001 0009 0000 00c0 0001
0005 0000 001a 043c 1b36 0406 3c15 04ac
4d05 3c1b 3604 063c 1504 ac4e 063c 2dbf
0003 0000 0005 000a 0029 0000 0005 0015
0000 000a 0010 0015 0000 0003 000c 0000
0032 000c 0000 0018 0002 001a 0005 0020
0007 001a 000a 001c 000b 001d 000d 001e
0010 0020 0012 001e 0015 001f 0016 0020
0018 0021 000d 0000 0034 0005 0000 001a
000e 000f 0000 0002 0008 002b 0006 0001
000d 0008 002b 0006 0001 0018 0002 002b
0006 0001 000b 000a 002c 002d 0002 002e
0000 000a 0002 4a07 0029 4a07 002f 0001
0031 0000 0002 0032 
 ```




## 2. magic和class文件版本

每个文件的开头4个字节称为魔数。它唯一的作用是确定这个文件是否是虚拟机能够接受的class文件。很多文件存储标准中也是使用魔数来进行身份验证，如：jpeg或者gif的文件头中也存在魔数。使用魔数而不是通过扩展名来验证文件类型是处于安全考虑，因为文件扩展名是可以更改的。class文件中的魔数是：0xCAFEBABE(咖啡宝贝?)

紧接着魔数后的第5、6个字节是次版本，第7、8个字节是主版本。JDK可以向下兼容版本，但不能运行超过它超过它版本号的class文件。如：上述的字节码JDK1.8环境下编译的，主版本号是：0x0034，转换成10进制后就是52，而JDK1.7可以生成的最高版本号是51,所以如果将该字节码放在1.7的JDK上，虚拟机将拒接执行。

## 3. 常量池

紧接着主次版本后面就是常量池入口，常量池主要存放两大类常量：字面量和符号引用。字面量比较接近于JAVA中的常量概念，如：字符串、final的常量值。而符号引用包括下面三类常量：

* 类和接口的全限名称(含有包名的类或者接口名称)
* 字段的名称和描述符
* 方法的名称和描述符


常量池中的每一个常量都是一个表，每一张表都有一个特点，表开始的第一位是一个u1类型的标志位，代表当前这个常量是属于那种常量类型。如下图14种常量类型：

| 类型                             | 标志 | 描述                     |
| -------------------------------- | ---- | ------------------------ |
| CONSTANT_Utf8_info               | 1    | UTF-8编码的字符串        |
| CONSTANT_Integer_info            | 3    | 整形字面量               |
| CONSTANT_Float_info              | 4    | 浮点型字面量             |
| CONSTANT_Long_info               | 5    | 长整形字面量             |
| CONSTANT_Double_info             | 6    | 双精度浮点型字面量       |
| CONSTANT _Class_info             | 7    | 类或接口的符号引用       |
| CONSTANT_String_info             | 8    | 字符串类型字面量         |
| CONSTANT_Fieldref_info           | 9    | 字段的符号引用           |
| CONSTANT_Methodref_info          | 10   | 类中方法的符号引用       |
| CONSTANT_InterfaceMethodref_info | 11   | 接口中方法的符号引用     |
| CONSTANT_NameAndType_info        | 12   | 字段或方法的部分符号引用 |
| CONSTANT_MethodHand_info         | 15   | 表示方法句柄             |
| CONSTANT_MethodType_info         | 16   | 标志方法类型             |
| CONSTANT_InvokeDynamic           | 18   | 表示一个动态方法调用点   |

我们再来看示例代码编译后的字节码，常量池的第一项常量，它的标志位是：cafe babe 0000 0034 0033 **`07`**00 0201，查询类型表，07是一个类或接口的引用。前面提到过，常量池的每一个常量都是一张表，CONSTANT _Class_info的表结构如下：

| 类型 | 名称       | 数量 |
| ---- | ---------- | ---- |
| u1   | tag        | 1    |
| u2   | name_index | 1    |

tag是标志位，区分常量类型。name_index是一个索引值，它指向常量池中的一个CONSTANT_Utf8_info类型常量，此常量代表这个类(或接口)的全限定名。name_index的值是：cafe babe 0000 0034 0033 07**`00 02`**01，0x0002指向常量池的第2项常量。

继续查找常量池中的第二项常量，它的标志位是：0700 02 **`01`**   0017，0x01是一个CONSTANT_Utf8_info常量，结构类型如下：

| 类型 | 名称   | 数量   |
| ---- | ------ | ------ |
| u1   | tag    | 1      |
| u2   | length | 1      |
| u1   | bytes  | length |

length说明这个utf-8编码的字符串长度是多少个字节。它后面紧跟着的就是长度为length字节的连续数据。连续数据采用的UTF-8缩略编码表示字符串，从```\u0001```到```\u007f```之间的字符串（相当于1~127的ASCII码），用一个字节表示，从```/u0080```到```\u7ff```之间用两个字节表示，从```\u0800```到```\uffff```之间按照普通的utf-8编码规则使用三个字节。

class文件中的类、方法、字段名称等都是用CONSTANT_Utf8_info来描述的，CONSTANT_Utf8_info能描述的最大长度是u2的最大值(65535)，如果我们在定义方法名或者成员变量时，如果方法名称长度超过了64kb，将会编译不通过。

我们接着往下解析，常量的length值等于：0700 0201 **`0017`**，0x0017从length后面连续23个字节：```7a6f 6f6b 6565 7065 7264 656d 6f2f 5465 7374 436c 6173 73```,转换后刚好是：```zookeeperdemo/TestClass``。

到此为止，我们已经解析了常量池的两项常量，剩余常量也是通过类似的方法来解析。根据标志位确定常量类型，如果是一个CONSTANT_Utf8_info类型常量，根据length解析utf8字符串，否则根据索引值跳转到索引值指向的常量，继续解析。

在JDK的bin目录下，有一个用于分析class文件的工具：javap。通过:```javap -v ```＋class类名，可以将类文件中的常量池解析出来，如下图：

```
Compiled from "TestClass.java"
public class zookeeperdemo.TestClass
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Class              #2             // zookeeperdemo/TestClass
   #2 = Utf8               zookeeperdemo/TestClass
   #3 = Class              #4             // java/lang/Object
   #4 = Utf8               java/lang/Object
   #5 = Utf8               m
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Methodref          #3.#11         // java/lang/Object."<init>":()V
  #11 = NameAndType        #7:#8          // "<init>":()V
  #12 = Utf8               LineNumberTable
  #13 = Utf8               LocalVariableTable
  #14 = Utf8               this
  #15 = Utf8               Lzookeeperdemo/TestClass;
  #16 = Utf8               inc
  #17 = Utf8               ()I
  #18 = Fieldref           #1.#19         // zookeeperdemo/TestClass.m:I
  #19 = NameAndType        #5:#6          // m:I
  #20 = Utf8               SourceFile
  #21 = Utf8               TestClass.java


```

## 4、访问标志

常量池结束后，紧接着两个字节是访问标志，这个标志用于识别一些类或者接口层次的访问信息，包括：这个class是类还是接口；是否定义为public类型，是否定义为abstract;如果是类，是否声明为final等。

| 标志名称       | 标志值 | 含义                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| ACC_PUBLIC     | 0X0001 | 是否为public类型                                             |
| ACC_FINAL      | 0x0010 | 是否被声明为final,只有类可以声明                             |
| ACC_SUPER      | 0X0020 | 是否允许使用invokespecial字节码的新语义，invokespecial指令在JDK1.0.2发生过改变，为了区别这条指令使用哪种语义，JDK1.0.2之后编译出来的类的这个标志都必须为真 |
| ACC_INTERFACE  | 0x0200 | 标识这是一个接口                                             |
| ACC_ABSTRACT   | 0x0400 | 是否是abstract类型，对于接口和抽象类，此标志值为真，其它值为假 |
| ACC_SYNTHENTIC | 0x1000 | 标识这个类不是由用户代码产生的                               |
| ACC_ANNOTATION | 0x2000 | 标识这是一个注解                                             |
| ACC_ENUM       | 0X4000 | 标识这是一个枚举                                             |

如本文提供的示例代码TestClass,它是一个public类型的类，因此ACC_PUBLIC、ACC_SUPER为真，其它为假，因此它的access_flags的值应该是0x0001|0x0020=0x0021,参照编译后的class 文件：6176 61**`00 21`**00 0100 0300，结果也是一致的。

## 5. 类索引、父类索引与接口索引集合

类索引和父类索引都是一个u2类型的数据。类索引用于确定这个类的全限定名称，父类索引用于确定这个类的父类的全限定名称，由于Java不允许类多继承，所以每个类只有一个父类。除Object外，每个类都有父类。接口索引集合是一组u2型的数据，因为类可以多继承，所以一个类可以有多个接口。接口索引集合的入口第一项(u2型数据)是接口计数器，表示索引表的数量，如果这个类没有实现任何接口则计数器为0。如示例代码的类索引、父类索引、接口索引集合：```0001 0003 0000```：前两个u2型数据分别指向常量池中的第一个和第二个常量，第三个u2型数据为0x0000表示TestClass没有实现任何接口。

部分常量池内容：

```
   #1 = Class              #2 // zookeeperdemo/TestClass
   #2 = Utf8               zookeeperdemo/TestClass
   #3 = Class              #4  // java/lang/Object
   #4 = Utf8               java/lang/Object
```

## 6. 字段表集合

字段表用来描述接口或者类中声明的变量。字段包括类级别变量以及实例级变量，但是不包括方法内部声明的局部变量。

描述一个字段可以包括的信息有：字段的作用域(public、private、protected修饰符)，是实例变量还是类变量(static修饰符)，可变性(final修饰符)，并发可见性(volatitle修饰符)，是否可被序列化(transient修饰符)，字段数据类型，字段名称等。上述的信息中，修饰符都是boolean型，要么有，要么没有，很适合用标志位来表示，而字段名称、类型无法确定字段名称、类型什么，所以需要通过常量池来保存常量名称、类型。

字段表结构

| 类型           | 名称             | 数量             |
| -------------- | ---------------- | ---------------- |
| u2             | access_flags     | 1                |
| u2             | name_index       | 1                |
| u2             | descriptor_index | 1                |
| u2             | attributes_count | 1                |
| attribute_info | attributes       | attributes_count |

字段修饰符放在access_flags中，它与类中的access_flags非常的类似。

字段访问标志

| 标志名称      | 标志值 | 含义                     |
| ------------- | ------ | ------------------------ |
| ACC_PUBLIC    | 0X0001 | 字段是否public           |
| ACC_PRIVATE   | 0x0002 | 字段是否private          |
| ACC_PROTECTED | 0X0004 | 字段是否protected        |
| ACC_STATIC    | 0X0008 | 字段是否static           |
| ACC_FINAL     | 0X0010 | 字段是否final            |
| ACC_VOLATILE  | 0X0040 | 字段是否volatile         |
| ACC_TRANSIENT | 0X0080 | 字段是否transient        |
| ACC_SYNTHETIC | 0X1000 | 字段是否有编译器自动生成 |
| ACC_ENUM      | 0X4000 | 字段是否enum             |

在实际情况中，ACC_PUBLIC、ACC_PRIVATE、ACC_PROTECTED只能选一，ACC_FINAL和ACC_VOLATILE不能同时选择。在接口中，字段必须有：ACC_PUBLIC、ACC_STATIC、ACC_FINAL。

name_index和descriptor_index是对常量池的引用，分别对应着字段的简单名称和字段描述。字段的简单名称是指没有字段类型(方法的简单名称是指没有返回类型和参数)。如这个类中的字段```m```和方法```int inc()```，它们的简单名称分别是：

```m```和```inc```。

描述符 的作用是用来描述字段的数据类型、方法的参数列表(数量、类型、顺序)、和返回值。根据描述规则，基本数据类型、void都用一个大写字符来描述，而对象类型则用字符L加对象的全限定名来表示。如下是描述符标识字符含义

| 标识字符 | 含义   | 标识字符 | 含义                          |
| -------- | ------ | -------- | ----------------------------- |
| B        | byte   | J        | long                          |
| C        | char   | S        | short                         |
| D        | double | Z        | boolean                       |
| F        | fload  | V        | void                          |
| I        | int    | L        | 对象类型如：Ljava/lang/Object |

对于数组类型，每一个维度将使用一个前置的```[```来描述，如：```java.lang.String[][]```类型的数组，将被记录为：```[[Ljava/lang/String```，```int[]```将被记为```[I```。

用描述符来描述方法时，按照先参数列表后返回值的顺序来描述，参数列表按照参数的顺序放在一组小括号```()```中。如方法void inc()的描述符是()V，方法java.lang.String.toString()的描述符是```()Ljava/lang/String```，方法```int indexOf(char[]source,int sourceOffset,int sourceCount,char[]target,int targetOffsetmint targetCount,int fromIndex)```的描述符是```([CII[CIII])I```。

在给出的示例代码中，第一个u2型数据是容量计数器，它的值是```0X0001```，说明这个类中只有一个字段表数据。接着就是字段访问标志```access_flags```,它的值为```0x0002```，说明这个字段是私有的，接着是字段简单名称```name_index```，它的值是0x0005，从常量表中查询得出，位于第五号位置的常量是```m```，然后是字段描述```descriptor_index```，它的值```0x0006```，查看常量池中第六号位置是```I```，根据这些信息可以推断出源代码定义的字段是：```private int m```。

在字段描述之后，还有一个属性集合，用来存储一些额外的信息，这个后面在讲。

部分常量池内容：

```
 #4 = Utf8               java/lang/Object
 #5 = Utf8               m
 #6 = Utf8               I
 #16 = Utf8               inc
 #17 = Utf8               ()I
```

## 7. 方法表集合

class文件中方法的存储结构和字段一致，表结构也和字段一样，依次包括了：访问标志(access_flags)、名称索引(name_index)、描述符索引(descroptor_index)、属性表集合几项，表结构如下：

| 类型           | 名称             | 数量             |
| -------------- | ---------------- | ---------------- |
| u2             | access_flags     | 1                |
| u2             | name_index       | 1                |
| u2             | descriptor_index | 1                |
| u2             | attributes_count | 1                |
| attribute_info | attributes       | attributes_count |

方法表access_flags与字段表有点区别，volatile关键字和transient关键字不能修饰方法，与之相对应的synchronized、native、strictfp、abstract关键字可以修饰方法、如下标志位表：

| 标志名称         | 标志值 | 含义                             |
| ---------------- | ------ | -------------------------------- |
| ACC_PUBLIC       | 0x0001 | 方法是否为public                 |
| ACC_PRIVATE      | 0x0002 | 方法是否为private                |
| ACC_PROTECTED    | 0x0004 | 方法是否为protected              |
| ACC_STATIC       | 0x0008 | 方法是否为static                 |
| ACC_FINAL        | 0x0010 | 方法是否为final                  |
| ACC_SYNCHRONIZED | 0x0020 | 方法是否为synchronized           |
| ACC_BRIDGE       | 0x0040 | 方法是否是由编译器产生的桥接方法 |
| ACC_VARARGS      | 0x0080 | 方法是否接受不定参数             |
| ACC_NATIVE       | 0x0100 | 方法是否为native                 |
| ACC_ABSTRACT     | 0x0400 | 方法是否为abstract               |
| ACC_STRICTEP     | 0x0800 | 方法是否为strictfp               |
| ACC_SYNTHETIC    | 0x1000 | 方法是否由编译器自动产生         |
|                  |        |                                  |

方法的定义通过访问标志、名称索引、描述符索引表达清楚，方法里面的代码存放在方法属性表集合中一个名为```code```的属性里面。

方法表集合中的第一个u2类型数据是```0x0002```，表示集合中有2个方法，一个是编译器添加的实例构造器```<init>```另一个就是源码中的```inc()```。第一个方法的访问标志的值是```0x0001```，代表这个方法是public型，名称索引值为```0x0007```，查看常量池的方法名称是```<init>```，描述符索引是```0x0008```，对应常量是```()V```，属性表计数器attributes_count的值为0x0001，表示此方法属性表集合有一项属性，属性表名称索引为```0x0009```，对应常量为```code```，说明此属性是方法的字节码描述。

## 8. 属性表集合

在Class文件、字段表、方法表都可以携带自己的属性表集合，以用来描述某些专有的信息，下面就方法表中的code属性来详细解释。

java程序方法中的代码经过javac编译器处理后，最终变为字节码指令存储在Code属性内。

code属性表结构

| 类型           | 名称                   | 数量                   |
| -------------- | ---------------------- | ---------------------- |
| u2             | attribute_name_index   | 1                      |
| u4             | attribute_length       | 1                      |
| u2             | max_stack              | 1                      |
| u2             | max_locals             | 1                      |
| u4             | code_length            | 1                      |
| u1             | code                   | code_length            |
| u2             | exception_table_length | 1                      |
| exception_info | exception_table        | exception_table_length |
| u2             | attributes_count       | 1                      |
| attribute_info | attributes             | attributes_count       |

```attribute_name_index```是一项指向```CONSTANT_Utf8_info```型常量的索引，常量值固定为```code```，```attribute_length```是指属性值的长度。

```max_stack```操作数栈深度的最大值，在方法执行的任意时刻，操作数栈都不会超过这个深度。虚拟机运行的时候会根据这个值来分配栈帧中的操作栈深度。

```max_locals```代表了局部变量所需的存储空间，max_locals的单位是Slot，Slot是虚拟机为局部变量分配内存所需要的最小单位。对于byte、char、float、int、short、int、short、boolean 和returnAddress等长度不超过32的数据类型，每个局部变量占用1个Slot来存放，而double和long这两种64位数据则需要两个Slot来存放。方法参数(包括this)、显示异常处理器的参数（try-catch语句中catch块所定义的异常）、方法体中定义的局部都需要局部变量表来存放。

```code_length```代表字节码长度，它虽然是u4型数据，但是实际上只使用了u2的长度，所以一个方法如果超过65535条字节码，java编译器会拒绝编译。

```code```是用于存储java源程序编译后生成的字节码指令，每个字节是一个u1类型的单字节，当虚拟机读取到code中的一个字节码时，就可以对应查出字节码代表的是什么指令。

前面提到过```<init>```方法的```code```属性，它的操作数栈和本地变量表的容量都为```0x0001```，字节码区域所占的空间长度为```0x0005```，根据字节码指令表翻译出来对应的字节码指令，字节码是```2a b7000ab1```，翻译过程如下：

1. 读入2a，查得表0x2a对应的指令是aload_0，这个指令是将第0个 Slot中reference类型的本地变量推送到操作数栈顶。
2. 读入b7，0xb7的指令为invokespecial，这条指令的作用是以栈顶的reference类型的数据所指向的对象作为方法接收者，调用此对象的实例构造器方法，private方法或者它的父类方法。
3. 读入00 0a，这个是invokespecial，查询常量池，0x000A对应的常量是




























