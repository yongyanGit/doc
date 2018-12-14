### Java 语法糖与Java编译器

#### 自动装箱与自动拆箱

Java语言拥有8个基本类型，每个基本类型都有对应的包装类型，之所以需要包装类型，是因为许多Java核心类库的API都是面上对象。

```java
public class Demo3 {

    public static void main(String[] args){

        ArrayList<Integer> list = new ArrayList<>();

        list.add(0);

        int result = list.get(0);
        System.out.println(result);

    }

}
```



```java
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: new           #2                  // class java/util/ArrayList
         3: dup
         4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
         7: astore_1
         8: aload_1
         9: iconst_0
        10: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        13: invokevirtual #5                  // Method java/util/ArrayList.add:(Ljava/lang/Object;)Z
        16: pop
        17: aload_1
        18: iconst_0
        19: invokevirtual #6                  // Method java/util/ArrayList.get:(I)Ljava/lang/Object;
        22: checkcast     #7                  // class java/lang/Integer
        25: invokevirtual #8                  // Method java/lang/Integer.intValue:()I

```

当我们向Interger的ArrayList添加int值时，便需要用到自动装箱，上面字节码偏移量为10的指令中，便调用Integer.valueOf方法，将int 型值转换成Integer型，再存储到容器中。

```java
 public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
 }
```

上面是Integer.valueOf的源代码，可以看到，当请求的int值在某个范围时(默认-128~127)，我们会返回缓存了的Integer对象，而当请求的int值不在范围之内时，我们会新建一个Integer对象。

当从泛型为Integer的ArrayList取出元素时，我们得到的实际上也是Integer对象，如果应用程序期待的是一个int值，那么就会发生自动拆箱。如字节码偏移量为25的指令，该指令调用Integer.intValue方法，这是一个实例方法，直接返回Integer对象所存储的int值。

```java
public int intValue() {
        return value;
 }
```

#### 泛型与泛型擦除

在前面例子生成的字节码中，往ArrayList中添加元素的add方法，所接收的参数类型是Object;而从ArrayList中获取元素的get方法，其返回的类型同样是Object。

```java
//添加
13: invokevirtual #5   // Method java/util/ArrayList.add:(Ljava/lang/Object;)Z
//获取
19: invokevirtual #6  // Method java/util/ArrayList.get:(I)Ljava/lang/Object;
```

之所以会出现这种情况是因为Java泛型的类型擦除。而之所以这么做主要是为了兼容引入泛型之前的代码。

当然并不是每一个泛型被擦除后都会变成Object，对于限定继承类的泛型参数，经过类型擦除后，所有的泛型参数都将变成所限定的继承类。

```
public class GenericTest <T extends Number> {
    T foo(T t){
        return t;
    }

}
```

```java
T foo(T);
    descriptor: (Ljava/lang/Number;)Ljava/lang/Number;
    flags:
    Code:
      stack=1, locals=2, args_size=2
         0: aload_1
         1: areturn
    Signature: #23                          // (TT;)TT;

```

如上：我定义了一个T extends Number的泛型参数，可以看到foo方法的方法描述符所接收的参数类型和返回参数类型都为Number。方法描述符是Java虚拟机识别、调用目标方法的关键。不过字节码中仍然存在泛型参数的信息，如：方法声明里的T foo(T)。

#### 桥接方法

泛型的擦除带来了不少问题，方法的重写就是其中一个。

```java
public class Merchant2<T extends Customer> {

    public double actionPrice(T customer){
        return 0.0d;
    }
}

public class VIPOnlyMerchant extends Merchant2<VIPCustomer> {
    @Override
    public double actionPrice(VIPCustomer customer) {
        return 0.0d;
    }
    public static void main(String[] args){
        System.out.println("hello");
    }
}

```



```java
public class VIPOnlyMerchant extends Merchant2<VIPCustomer>
 ...
  public double actionPrice(VIPCustomer);
    descriptor: (LVIPCustomer;)D
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: dconst_0
         1: dreturn
     

  public double actionPrice(Customer);
    descriptor: (LCustomer;)D
    flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: checkcast     #5                  // class VIPCustomer
         5: invokevirtual #6                  // Method actionPrice:(LVIPCustomer;)D
       

```

如上字节码，在VIPOnlyMerchant类中重写了父类Merchant2的actionPrice(Customer)方法，该桥接方法将传入的Customer参数强制转换成VIP类型再调用原本的 actionPrice(VIPCustomer)方法。

当一个声明类型为Merchant，实际类型为VIPOnlyMerchant的对象，调用actionPrice方法时，字节码的符合引用之下的是Merchant.actionPrice(Customer)方法。Java虚拟机将动态绑至VIPOnlyMerchant的桥接方法之中，并且调用其actionPrice(VIP)方法。

除了泛型重写会造成桥接方法之外，如果子类定义了一个与父类参数类型相同的方法，其返回类型为父类返回类型的子类，那么Java编译器也会为其生成桥接方法。

```java
public class Merchant {
    public Number actionPrice(Customer customer){
        return 0;
    }
}

public class NaiveMerchant extends Merchant {
    @Override
    public Double actionPrice(Customer customer) {
        return 0.0d;
    }
}
```

```java
public class NaiveMerchant extends Merchant
 
  public java.lang.Double actionPrice(Customer);
    descriptor: (LCustomer;)Ljava/lang/Double;
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: dconst_0
         1: invokestatic  #2                  // Method java/lang/Double.valueOf:(D)Ljava/lang/Double;
         4: areturn
     
  public java.lang.Number actionPrice(Customer);
    descriptor: (LCustomer;)Ljava/lang/Number;
    flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: invokevirtual #6                  // Method actionPrice:(LCustomer;)Ljava/lang/Double;
         5: areturn
}

```

class文件中允许出现两个同名、同参数类型但是不同返回类型的方法，由于该桥接方法中，我们同样会调用原方法。

#### 其它语法糖

foreach循环允许Java程序在for循环里遍历数组或者Iterable对象，对于数组来说，foreach循环将从0开始逐一访问数组中的元素，直至数组的末尾。

```java
public class ForeachDemo {
   static List<Integer> list = new ArrayList<>();
   static {
       list.add(1);
       list.add(2);
   }
    public static void main(String[] args){
        for (Integer i : list){
            System.out.println(i);
        }
    }
}
```

```java
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: getstatic     #2                  // Field list:Ljava/util/List;
         3: invokeinterface #3,  1            // InterfaceMethod java/util/List.iterator:()Ljava/util/Iterator;
         8: astore_1
         9: aload_1
        10: invokeinterface #4,  1            // InterfaceMethod java/util/Iterator.hasNext:()Z
        15: ifeq          38
        18: aload_1
        19: invokeinterface #5,  1            // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
        24: checkcast     #6                  // class java/lang/Integer
        27: astore_2
        28: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
        31: aload_2
        32: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        35: goto          9
        38: return

```

如上字节码偏移量为3处，foreach循环内部其实调用的是Iterator方法，然后偏移量为10通过hasNext来判断集合中是否有元素，偏移量19通过next()方法取出元素。



字符串switch编译而成的字节码实际上是一个哈希桶，由于每个case对应的字符串都是常量，因此Java编译器会将原来的字符串switch转换成int值switch，比较所输入的字符串的哈希值。由于字符串哈希值很容易发生碰撞，因此还需要String,equals逐个比较相同哈希值的字符串。