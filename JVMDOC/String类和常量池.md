#### String类和常量池

1. 创建String对象的两种方式：

```java
String str1 = "abcd";
String str2 = new String("abcd");
System.out.println(str1==str2);//false
```

第一种方式是直接从常量池中获取对象，在编译期，就会将该字符串添加到运行区常量池；第二种是在堆内创建一个对象，该种方式也会在编译期将"abcd"添加到常量池，然后在运行期在堆上创建一个"abcd"对象。

**字符串拼接**

```java
String str1 = "str";

        String str2 = "ing";

        String str3 = "str"+"ing";//在常量池中创建一个string字符串

        String str4 = str1 + str2;//在堆上创建新的对象

        String str5 = "string";//常量池中的对象

        System.out.println(str3 == str4);//false

        System.out.println(str3 == str5);//true

        System.out.println(str4 == str5);//false
```

**8种基本类型的包装类**

Java基本类型的包装类大部分都实现了常量池，即Byte,Short,Integer,Character,Boolean,这5种包装类型默认创建了数值[-128,127]的相应类型的缓存数据，但是超出这个范围仍然会去创建对象。

两种浮点型包装类Float，Double并没有实现常量池技术。

```java
public void test5(){

        Integer i1 = 33;

        Integer i2 = 33;

        System.out.println(i1 == i2);//true

        Integer i11 = 333;

        Integer i22 = 333;

        System.out.println(i11 == i22);//false

        Double i3 = 1.2;

        Double i4 = 1.2;

        System.out.println(i3 == i4);//false

    }
```

Integer缓存代码：

```java
public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
}
```

Integer i1 = 30;Java在编译阶段会将代码封装成Integer i1 = Integer.valueOf(30);从而使用缓存池中的对象。

Integer i2 = new Integer(40)这种情况下会创建新的对象。