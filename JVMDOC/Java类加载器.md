#### Java类加载器

一般来说，Java源程序(.java文件)在经过Java编译器编译后就被转换成Java字节码(.class文件)，类加载器否则读取Java字节代码，并转Java.lang.Class类的一个实例。每个实例用来表示一个Java类，通过此实例的newInstance()方法就可以创建该类的一个对象。

1. java.lang.ClassLoader介绍

ClassLoader类的基本职责是根据一个指定的类的名称找到或者生成对应的字节码，然后从这些字节码中定义处一个Java类，即java.lang.Class类的一个实例。

* getParent()：返回该类加载器的父类加载器
* loadClass(String name)：加载名称为name的类，返回的结果是java.lang.Class类的实例

* findClass(String name)：查找名称为name的类，返回的结果是java.lang.Class类的实例
* findLoadedClass(String name)查找名称为name的已经加载过的类，返回的结果是java.lang.Class类的实例
* defineClass(String name, byte[] b, int off, int len)：将字节数组b中的内容转换成Java类，返回的结果是java.lang.Class类的实例。这个方法被声明成final。
* resolveClass(Class<?> c)：链接指定的Java类

这里详细讲一下loadClass(String name)的处理逻辑：

当虚拟机调用loadClass()方法时，首先会调用findLoadedClass()，检查类字节码是否已经被加载，如果已经加载完成，则返回该字节码的Class对象；如果还没被虚拟机加载，则调用父接口的loadClass方法，依次类推；最后如果字节码仍然没有被加载，则调用当前类加载器的findClass方法，根据名称去加载类字节码，在findClass内部调用defineClass真正加载类字节码。

2. 类加载器的组织结构

Java中的类加载器大致分为两类：一类是系统提供的，另一类是由Java应用开发人员编写的。系统提供的类加载器主要有下面三个：

* 引导类加载器(bootstrap class loader)：它主要用来加载Java的核心库，是用原生代码来实现的，并且不继承java.lang.ClassLoader。
* 扩展类加载器(extensions class loader)：它用来加载Java的扩展库。比如存放在JRE的lib/ext目录下的jar包(以及由系统变量java.ext.dirs指定的类)。
* 系统类加载器(system class loader)：它根据Java应用的类路径(CLASSPATH)来加载Java类。一般来说，Java应用的类都是由它来完成加载的。可以通过ClassLoader.getSystemClassLoader()来获取。

除了系统提供的类加载器以外，开发人员可以通过继承java.lang.ClassLoader类来实现自己的类加载器，以满足一些特殊的需求。

除了引导类加载器之外，所有的类加载器都有一个父类加载器。对于系统提供的类加载器来说，系统类加载器的父类加载器是扩展类加载器，扩展类加载器的父类加载器是引导类加载器，开发人员编写的类加载器的父类加载器一般都是系统类加载器.

![loader](../images/jvm/loader.jpg)

例子：

```java

public class ClassLoadDemo {
    public static void main(String[] args){
        ClassLoader loader = ClassLoadDemo.class.getClassLoader();
        while (loader != null){
            System.out.println(loader.toString());
            loader = loader.getParent();

        }
    }
}
```

执行结果：

```
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@799f7e29
```

可以看到第一个输出的ClassLoadDemo的类加载器，即系统类加载器；第二个是扩展类加载器。需要注意的是，这里并没有输出引导类加载器，这是由于有些JDK的实现对于父类加载器是引导类加载器的情况，getParent()方法返回为null。

3. 类加载器的过程

类加载在尝试自己去查找某个类的字节代码时，会先代理给其父类加载器，由父类加载器先去尝试加载这个类，依次类推，这意味着。真正完成加载类的类加载器和启动这个加载过程的类加载器有可能不是同一个。真正完成类的加载工作是通过调用defineClass来实现；而启动类的加载过程是通过调用loadClass来实现的。前者称为一个类的定义加载器(defining loader)，后者称为初始加载器(initiating loader)。

方法loadClass()抛出的是java.lang.ClassNotFoundException异常，方法defineClass抛出的是java.lang.NoClassDefFoundError。

在Java虚拟机中判断两个类是相同的，不仅要看类的全名称，还要看加载此类的类加载器是否相同，只有两者都相同的情况，才认为两个类相同，否则即使是同一份字节码，被两个不同的类加载器加载之后的类，也是不同的。

例子：两个不同的类加载器加载同一份字节码。



```java
//定义一个类加载器，从文件中读取字节码文件
public class FileSystemClassLoader extends ClassLoader {

    private String rootDir;

    public FileSystemClassLoader(String rootDir){

        this.rootDir = rootDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {

        byte[] classData = getClassData(name);
        if (classData == null){
            throw new ClassNotFoundException();
        }else {
			//真正的类加载
            return defineClass(name,classData,0,classData.length);

        }

    }

    private byte[] getClassData(String className){

        String path = classNameToPath(className);
        try {
            InputStream inputStream = new FileInputStream(path);
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            int buffSize = 4096;
            byte[] buffer = new byte[buffSize];
            int index = 0;
            while ((index = inputStream.read(buffer)) != -1){

                byteArrayOutputStream.write(buffer,0,index);
            }
            return byteArrayOutputStream.toByteArray();

        }catch (IOException e){
            e.printStackTrace();
        }

        return null;
    }


    private String classNameToPath(String className){

        return rootDir + File.separatorChar+className.replace('.',File.separatorChar)+".class";

    }


}
```

```java
//测试类加载器，
public  void testClassIdentity(){
		//被加载类的路径，注意：不能与自定义的类加载器在同一个路径下，防止当前的类加载器的父类加载器找到将要被加载类。
        String classDataRootPath = "/opt";
		//先定义两个不同的类加载器
        FileSystemClassLoader f1 = new FileSystemClassLoader(classDataRootPath);
        FileSystemClassLoader f2 = new FileSystemClassLoader(classDataRootPath);
    	//被加载的类的完整名称
        String className = "com.seaeverit.reading.bean.Sample";
        try {
            Class<?> class1 = f1.loadClass(className);
            Object o1 = class1.newInstance();
            Class<?> class2 = f2.loadClass(className);
            Object o2 = class2.newInstance();
            System.out.println(o1.getClass().getClassLoader());
            System.out.println(o2.getClass().getClassLoader());
            //调用setSample，如果是两个不同的类，将会抛出cast异常
            Method setSampleMethod = class1.getMethod("setSample",java.lang.Object.class);
            setSampleMethod.invoke(o1,o2);

        }catch (Exception e){
            e.printStackTrace();
        }

    }

//被加载的类
public class Sample {

    private Sample sample;

    public void setSample(Object o){
        this.sample = (Sample) o;
    }

}
```

执行结果：

```java
Caused by: java.lang.ClassCastException: com.seaeverit.reading.bean.Sample cannot be cast to com.seaeverit.reading.bean.Sample
	at com.seaeverit.reading.bean.Sample.setSample(Sample.java:13)
```

4. 线程上下文类加载器

线程上下文类加载器随着线程的创建而设置，通过类java.lang.Thread中的getContextClassLoader()和setContextClassLoader(ClassLoader cl)来获取和设置上下文类加载器。如果没有通过setContextClassLoader方法设置的话，线程将继承其父类线程的上下文类加载器。Java应用运行的初始线程的上下文类加载器是系统类加载器。在线程中运行的代码通过此类加载器来加载类和资源。

类加载器的代理模式并不能解决Java应用开发中遇到的所有问题，如：Java提供了很多服务提供者接口(Service Provicer Interface)SPI，允许第三方为这些接口提供实现，如：JDBC。这些SPI的实现代码很可能是作为Java应用所依赖的jar包被包含进来的，如：mysql的jdbc驱动，它们可以通过类路径(CLASSPATH)来找到，即可以被系统类加载器加载。但是问题在于SPI的接口是Java核心库的一部分，是由引导类加载器加载的，而SPI的实现类由系统加载器来加载的。引导类加载器无法找到SPI的实现类，因为它只加载Java核心库。

针对这个问题，线程上下文类加载器正好可以解决这个问题，如果不做任何的设置，Java应用的线程的上下文类加载器就是系统上下文类加载器。在SPI接口的代码中使用线程上下问类加载器，就可以成功的加载到SPI实现类。

我们以jdbc为例子来分析：

```java
//加载驱动
Class.forName("com.mysql.jdbc.Driver");
//获取连接
Connection conn = java.sql.DriverManager.getConnection(url, "name", "password");
```

Class.forName：用来加载类，该方法有两种形式：Class.forName(String name, boolean initialize, ClassLoader loader)`和 `Class.forName(String className)，第一种形式的参数name表示类的全名称；initialize表示是否初始化类；loader表示加载使用的类加载器。第二种形式相当于设置了参数initialize为true,loader的值为当前类的类加载器。

在com.mysql.jdbc.Driver实现类中有一个静态类，它会将自己注册到DriverManager的CopyOnWriteArrayList集合中.

```java
static {
		try {
			java.sql.DriverManager.registerDriver(new Driver());
		} catch (SQLException E) {
			throw new RuntimeException("Can't register driver!");
		}
	}
```

DriverManager的registerDriver方法：

```java
 public static synchronized void registerDriver(java.sql.Driver driver,
            DriverAction da)
        throws SQLException {

        /* Register the driver if it has not already been added to our list */
        if(driver != null) {
            registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
        } else {
            // This is for compatibility with the original DriverManager
            throw new NullPointerException();
        }

        println("registerDriver: " + driver);

    }
```



getConnection：获取数据库连接

```java
 private static Connection getConnection(
 	String url, java.util.Properties info, Class<?> caller) throws SQLException {
    //获取类加载器，
    ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
    synchronized(DriverManager.class) {
    	// 如果为空，获取上下问类加载器
        if (callerCL == null) {
        	callerCL = Thread.currentThread().getContextClassLoader();
        }
    }
    if(url == null) {
    	throw new SQLException("The url cannot be null", "08001");
    }
    SQLException reason = null;
     
    for(DriverInfo aDriver : registeredDrivers) {
    	if(isDriverAllowed(aDriver.driver, callerCL)) {
                try {
              
                    Connection con = aDriver.driver.connect(url, info);
                    if (con != null) {
                        return (con);
                    }
                } catch (SQLException ex) {
                    if (reason == null) {
                        reason = ex;
                    }
                }

            } else {
                println("    skipping: " + aDriver.getClass().getName());
            }
        }  
    }
}



private static boolean isDriverAllowed(Driver driver, ClassLoader classLoader) {
	boolean result = false;
    if(driver != null) {
    	Class<?> aClass = null;
        try {
            //用指定的classLoader根据类名称加载
          aClass =  Class.forName(driver.getClass().getName(), true, classLoader);
            } catch (Exception ex) {
                result = false;
            }
             //两个类对象相等，返回true
        	result = ( aClass == driver.getClass() ) ? true : false;
        }
        return result;
    }
```

5. 类加载器与web容器

对于运行在JavaEE容器中的web应用来说，类加载器的实现方式与一般的ava应用有所不同。以Tomcat来说，每个web应用都有一个对应的类加载器实例，该类加载器也使用代理模式，所不同的是它首先尝试去加载某个类，如果找不到再代理给父类类加载器，这与一般类加载器的顺序是相反的。在Java Servlet规范中的推荐这样做的目的是web应用自己的类的优先级高于web容器提供的类。这种代理模式的一个例外是：Java核心库的类不在查找的范围之内，这是为了保证Java核心库的类型安全。

绝大多数情况下，web应用开发人员不需要考虑与类加载器相关的细节，下面是几条简单的原则：

* 每个web应用自己的Java类文件和使用的jar包，分别放在WEB-INF/class和WEB-INF/lib目录下面
* 多个应用共享的 Java 类文件和 jar 包，分别放在 Web 容器指定的由所有 Web 应用共享的目录下面
* 当出现找不到类的错误时，检查当前类的类加载器和当前线程的上下文类加载器是否正确

#### 链接

链接是指将创建的类合并到Java虚拟机中，使之能够执行的过程。它分为验证、准备以及解析三个阶段。

验证阶段的目的，在于确保被加载类能够满足Java虚拟机的约束条件，不会危害虚拟机自身的安全。

准备阶段则是为被加载的类的静态字段分配内存。Java代码中对静态字段的具体初始化，则会在稍后的初始化阶段中进行。除了分配内存外，部分虚拟机还会在此阶段构造跟类层次相关的数据结构，比如：用来实现虚方法的动态绑定的方法表。

在class被加载到Java虚拟机之前，这个类无法知道其他类及其方法、字段所对应的具体地址，甚至不知道自己的方法、字段的地址。因此每当需要引用这些成员时，Java编译器会生成一个符合引用，然后在运行阶段，这个符号引用一般都能够定位到具体的目标上。比如：对于一个方法调用，编译器会生成一个包含目标方法所在类的名字、目标方法的名字、接收参数类型以及返回值类型额符号引用，这个引用就是用来指代指要调用的方法。

解析阶段的目的正是将这些符号引用解析成实际引用。如果一个符号引用指向一个未被加载的类或者未被加载类的字段或方法，那么解析将触发这个类的加载（但未必触发这个类的链接和初始化）。

实际引用与符合引用的区别在于：比如："xxx的房子“这种说法，不管它存在不存在，我们都可以用它来指代xxx的房子。实际引用则好比实际的通讯地址。

Java虚拟机规范没有要求在链接过程中完成解析，它规定了：如果某些字节码使用看符号引用，那么在执行这些字段之前，需要完成这些符号引用的解析。

#### 初始化

类加载的最后一步是初始化，便是为常量值的字段赋值，执行<clinit>方法的过程。Java虚拟机会通过加锁来确保类的<clinit>方法仅被执行一次。

如果直接赋值的静态字段被final所修饰，并且它的类型是基本类型或者字符串，那么该字段便会被Java编译器标记为常量，其初始化直接由Java虚拟机完成。除此之外的直接赋值操作、以及所有静态代码块中的代码，则会被Java编译器置于同一个方法中，命名为<clinit>。

类的初始化触发条件：

* 当虚拟机启动时，初始化用户指定的主类。
* 当遇到用以新建目标实例的new指令时，初始化new指令的目标类
* 当遇到调用静态方法的指令时，初始化该静态方法所在的类
* 当遇到访问静态字段指令时，初始化该静态字段所在的类
* 子类的初始化会触发父类的初始化
* 如果一个接口定义了default方法，那么直接实现或者间接实现该接口的类的初始化，会触发该接口的初始化。
* 使用反射API对某个类进行反射调用时，初始化这个类
* 当初次调用MethodHandle实例时，初始化该MethodHandle指向的方法所在的类

