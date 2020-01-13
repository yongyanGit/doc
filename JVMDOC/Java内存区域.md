#### Java内存区域

数据存储区域，对于这一块区域的划分，各个虚拟机有各自的划分方式，不过它们都必须遵从JAVA虚拟机的基本规范去实现。

![jvmarea](../images/jvm/jvmarea.png)

1. 程序计数器（PC寄存器）

程序计数器存储着下一条指令的地址，处理器通过它来选取下一条要执行的字节码指令。如果执行的是native方法，则这个计数器为空。

Java虚拟机的多线程运行是通过轮流切换线程并分配处理器执行的时间来实现的，为了保证当线程挂起并唤醒后，可以在它原来正确的位置继续执行，因此它是**线程私有**的。程序计数器是唯一一个不会出现OutOfMemoryError的内存区域，它的生命周期随着线程的创建而创建，随着线程的结束而死亡。

2. Java虚拟机栈

虚拟机栈是一个类似倒立的数组数据结构的空间，越往上它内部元素的地址越大。通过使用栈指针的上下移动可以申请和释放栈空间，即栈指针向下移动，则分配新的栈空间，若向上移动，则释放那些空间。

与程序计数器一样，Java虚拟机栈也是线程私有的，并且生命周期与线程相同。每个方法执行的同时会创建一个栈帧，用于存放局部变量表，操作数栈、动态链接、方法出口等信息。在编译期，栈帧的大小就已经确定了，因为存储在它内部的数据的大小和生命周期已经确定了。

Java虚拟机规范定义了两种与栈空间相关的异常：StackoverFlowError和OutofMemoryError，如果在计算过程中，请求的栈深度大于可用的栈深度，则程序会抛出StackoverFlowError异。如果Java栈可以动态扩展，但是没有足够的内存空间来支持，则会抛出OutofMemoryError异常。

我们可以使用-Xss来设置虚拟机栈的大小(每条线程可以申请的栈大小)，栈的大小直接决定了函数调用的最大可达深度：

```java
//-Xss 1m，设置栈的大小来测试，程序会抛出StackoverFlowError异常。
public class StackDemo {
	
	private int count = 0;
	
	public void recursion(){
		count ++;
		recursion();
	}
	
	public void testStack(){
		try{
		recursion();
		}catch(Exception e){
			System.out.println("deep of stack is "+count);//打印栈益处的深度
		}
	}
	
	public static void main(String[] args){
		StackDemo stackDemo = new StackDemo();
		stackDemo.testStack();
		
	}

}
```

####  局部变量表

局部变量表存放了编译器可知的基本类型数据（boolean、byte、char、int、float、long、double）其中byte、short、char在存储前被转换成int，boolean被转换成int（0表示false，非0表示true），对象引用，returnAddress类型（返回地址，当一个方法中调用另一个方法时，被调用的方法执行完后，需要返回到原调用方法中，而这个返回地址就是用来标识程序从哪里开始执行

在局部变量表中64位的double、long占用2个局部变量空间（Slot），其余数据类型占用1个。局部变量的内存空间在编译期就确定了，在运行期不会改变它的大小。局部变量表只在当前函数中有效，当函数调用结束后，随着函数帧栈的销毁，局部变量表也会随之销毁。帧栈中的局部变量的槽位是可以重用的，如果一个局部变量过了其作用域，那么在其作用域之后声明的新的局部变量就很有可能复用过期局部变量的槽位，从而达到节省资源的目的。

下面是一个例子，局部变量表中相当于一个数组，它里面存放this指针(仅非静态方法)、所传入的参数以及字节码中的局部变量：

```java
public void foo(long l,float f){
        {
            int i = 0;
        }
        
        {
            String s = "hello world";
        }
    }
```

上面这段代码为例，由于它不是一个静态方法，因此局部变量数组的第0个单元存放者this指针；第一个参数long类型，存储在数组的1和2两个单元中；第二个参数则是float类型，存储在数组的第3个单元中。方法体中有两个代码块，分别定义了两个局部变量i和s，由于这两个局部变量的生命周期没有重合之处，Java编译器可以将它们编排至同一个单元中，也就是说局部变量数组的第4个单元将为i或者s。

![constable](../images/jvm/constable.png)

**通常存储在局部变量表的值需要加载到操作数栈中方能进行计算**，得到结果后再存储到局部变量数组中。这些加载、存储指令是区分类型的，如：int类型的加载指令为iload，存储指令为istore。加载局部变量数组时，需要指定所加载单元的下标，比如：aload0指加载第0个单元所存储的引用。

![cons](../images/jvm/constables.jpg)

在Java字节码中唯一能够直接作用于局部变量表的指令是iinc M N(M 是非负整数，N为整数)。该指令是将局部变量数组中的第M个单元中的int值增加N,常用于for循环中自增量的更新。

```java
public void foo(){
	for (int i = 100; i>= 0;i--){
    }
}

public void foo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=2, args_size=1
         0: bipush        100//直接操作操作数栈，设置值100
         2: istore_1//将100存放到局部变量数组下标为1的地方
         3: iload_1//再加载局部变量数组下标为1的元素
         4: iflt          13//如果小于，跳到字节码偏移13，即返回结果
         7: iinc          1, -1//否则直接作用于局部变量表，将值减一
        10: goto          3//跳到字节码偏移3
        13: return
```

再来看一个例子

```java
 public static int bar(int i){
 	return ((i+1)-2)*3/4;
 }

public static int bar(int);
    descriptor: (I)I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: iload_0//将局部变量表第0个元素加载到操作数栈顶
         1: iconst_1//将1加载到操作数栈顶
         2: iadd//相加
         3: iconst_2//将2加载到操作数栈顶
         4: isub//相减
         5: iconst_3//将3加载到操作数栈顶
         6: imul//相乘
         7: iconst_4//将4加载到操作数栈顶
         8: idiv//相除
         9: ireturn

```

字节码中的stack=2，locals=1代表该方法需要的操作数栈空间为2，局部变量数组空间为1。因为bar方法是一个静态方法，所以局部变量表中的第一个存的不是this指针。如下，以bar(5)为例：

![bar](../images/jvm/bar.png)

局部变量表中的变量也是重要的垃圾回收根节点，只要被局部变量表中直接或者间接引用的对象都是不会被回收的，如下示例：

```java
//配置:XX:+PrintGC可以打印gc信息
public void localGC1(){
		
	byte[] a = new byte[6*1024*1024];
	System.gc();//进行GC时，当前方法还没有退出
	System.out.println("gc");
}
//运行结果：
//[GC (System.gc())  6809K->6552K(125952K), 0.0089809 secs]
```

在localGC1中申请空间后，立即进行垃圾回收，很明显，由于byte数组被变量a引用，因此无法回收这块空间。

```java
public void localGC2(){
	byte[] a = new byte[6*1024*1024];
	a = null;
	System.gc();
	System.out.println("gc");
}
//运行结果：
//[GC (System.gc())  6809K->392K(125952K), 0.0010687 secs]
```

在localGC2方法中，在垃圾回收前，先将变量a设置为null，使得数组失去强引用，故垃圾回收可以顺利回收这块空间。

```java
public void localGC3(){
	{
		byte[] a = new byte[6*1024*1024];
	}
	System.gc();
	System.out.println("gc");
}
// 运行结果：[GC (System.gc())  6809K->6552K(125952K), 0.0053748 secs]

```

对于localGC3，在进行垃圾回收前，先使用局部变量a失效，虽然变量a已经离开了作用域，但是变量a依然存在于局部变量表中，并且也指向这块byte数组，故byte数组依然无法被回收。

```java
public void localGC4(){
	{
		byte[] a = new byte[6*1024*1024];
	}
	int c = 10;
	System.gc();
	System.out.println("gc");
}
//执行结果：[GC (System.gc())  6809K->392K(125952K), 0.0009798 secs]
```

对于localGC4，在垃圾回收前，不仅使变量a失效，更是声明了变量c，使变量c复用变量 a的字，由于变量a此时被销毁，故垃圾回收器可以顺利回收byte数组。

####操作数栈

操作数栈在解释执行过程中，每当为Java方法分配栈帧时，Java虚拟机往往需要开辟一块额外的空间作为操作数栈，**来存放计算的操作数以及返回结果**。它是一个后进先出（LIFO）栈，而它的长度也是在编译时期就写入了class文件当中，是固定的。如下foo 方法：执行每一条指令之前，Java虚拟机要求该指令的操作数压入操作数栈中，在执行指令时，Java虚拟机会将该指令所需的操作数弹出，并且将指令的结果重新压入栈中。

![add1](../images/jvm/add1.png)

以加法指令iadd为例，假设在执行该指令前，栈顶的两个元素分别是int值1和int 值2，那么iadd指令将弹出这两个int，并将求得的和int值3压入栈中。

![add2](../images/jvm/add2.png)

Java字节码中有好几条指令是直接作用在操作数栈上的，最常见的便是dup：复制栈顶元素，以及pop：舍弃栈顶元素。

dup指令常用于复制new指令生成的未经过初始化的引用。当执行new指令时，Java虚拟机将指向一块已分配的、未初始化的内存的引用压入到操作数栈中。

```java
public void foo(){
	Object o = new Object();
}
public void foo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2              // class java/lang/Object
         3: dup
         4: invokespecial #1              // Method java/lang/Object."<init>":()V
         7: astore_1
         8: return


```

如上：当new指令产生一个还没有进行初始化的引用后，接着调用dup指令复制new指令的结果，然后将这个引用作为调用者去调用其构造方法，也就是上面的invokespecial指令，当调用返回后，操作数栈上仍有原本由new指令生成的引用。

pop指令则常用于舍弃调用指令返回的结果，如下：

```java
 public void foo(){
       bar();
    }

    public static boolean bar(){
        return false;
    }

public void foo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: invokestatic  #2                  // Method bar:()Z
         3: pop
         4: return

```

如上我们在foo方法中调用了静态方法bar()，但是却不用其返回值，对应invokestatic指令，该指令仍旧会把返回值压入到foo方法的操作数栈中，因此Java虚拟机需要额外的执行pop指令，将返回值舍弃。

在Java字节码中，有一部分指令可以直接将常量加载到操作数栈上，以int型为例，Java虚拟机既可以通过iconst指令加载-1至5之间的int值，也可以通过bipush、sipush加载一个字节、两个字节所能代表的int值。

Java虚拟机还可以通过Idc加载常量池中的常量，如：Idc#18将加载常量池中的第18项。

这些常量包括int型、long型、float型、double型、String型以及Class型的常量，如下：

![loadcons](../images/jvm/loadcons.jpg)

正常情况下，操作数栈的压入弹出都是一条条指令完成的，唯一的例外情况就是在抛出异常时，Java虚拟机会清除操作数栈上的所有内容，而后将异常实例压入到操作数栈上。

Java虚拟机栈会抛出两种异常：StackOverFlowError和OutOfMemoryError

* 若Java虚拟机栈的内存大小不允许动态扩展，那么当线程请求栈的深度超过当前Java虚拟机栈的最大深度的时候，就会抛出StackOverFlowError。
* 若Java虚拟机栈的内存大小允许动态扩展，且当线程请求栈时内存用完了，无法再动态扩展了，此时抛出OutOfMemoryError异常。

3. 本地方法栈

本地方法栈(Native Method Stack)与虚拟机栈非常相似，它们之间的区别是虚拟机栈为虚拟机执行Java方法服务，而本地方法栈则为虚拟机使用到的Native方法服务。

在HotSpot虚拟机中，本地方法栈和虚拟机栈合二为一。与虚拟栈一样，本地方法栈区域也会出现 StackOverFlowError 和 OutOfMemoryError 两种异常。

4. 堆

Java虚拟机所管理的内存中最大的一块，Java堆是所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。

Java堆是垃圾收集器管理的主要区域，因此也被称为GC堆。从垃圾回收的角度，由于现在收集器基本都是采用分代垃圾收集算法，所以Java堆还可以细分为：新生代和老年代。再细致一点：Eden空间、From Survivor、To Survivor空间。

![heap](../images/jvm/heap.png)

在JDK1.8中移除整个永久代(1.8之前，方法区的实现方式是永久代)，取而代之的是一个叫做元空间（Metaspace)的区域,(永久代使用的是jvm的堆空间，而元空间使用的是物理内存，直接受到本机的物理内存的限制)

5. 方法区

和Java堆一样，方法区是一块所有线程共享的内存区域，它用于保存系统的类信息，比如类的字段、方法、常量池。方法区的大小决定了系统可以保存多少个类，如果系统定义了太多的类，导致方法区溢出，虚拟机同样会抛出内存溢出错误。

在JDK1.6、JDK1.7中，方法区可以理解为永久区，可以使用参数-XX:PermSize和-XX:MaxPermSize指定，默认情况下，-XX:MaxPermSize为64MB。

如果系统使用了一些动态代理，那么有可能会在运行时生成大量的类，如果这样，就需要设置一个合理的永久区大小，确保不发生永久区内存溢出。相对而言，垃圾收集行为在这个区域是比较少出现的，但并非数据进入方法区后就永久存在了。

在JDK1.8中，永久区已经被彻底移除了，取而代之的是元数据区，元数据区大小可以使用参数-XX:MaxMetaspaceSize指定，这是一块堆外的直接内存，与永久区不同，如果不指定大小，默认情况下，虚拟机会耗尽所有的可用系统的内存。**元数据存储类的元信息，将运行时常量池并入堆中**

#### 运行时常量池

运行时常量池是方法区的一部分(jdk1.8之前)。Class文件中除了有类、字段、方法、接口等描述信息外，还有常量池信息，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池存放。

既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法申请到内存时，会抛出OutOfMemoryError异常。

Jdk1.7之后版本的JVM将运行时常量从方法区移除了，**在Java堆中开辟了一块区域来存放运行时常量池**。

![static](../images/jvm/static.png)

7. 直接内存

直接内存并不是虚拟机运行时数据的一部分，直接内存直接向系统申请内存区间。通常访问直接内存的速度会优于Java 堆。因此出于性能考虑，读写频繁的场合可能会考虑使用直接内存。

直接内存在Java堆外，因此它的大小不会直接受限于Xmx指定的最大堆大小，但是系统内存是有限的，Java堆和直接内存的总和依然受限于操作系统能给出的最大内存。

JDK1.4中新加入的NIO类引入了一种基于通道与缓存区的i/o方式，它可以直接使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，避免了在Java堆和Native堆之间来回复制数据。
