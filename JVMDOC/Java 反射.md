### Java 反射

首先我们先熟悉下反射api的调用，如下示例：

```java
public void test1()throws Exception{
		//加载字节码
        Class clazz = getClazz();
        System.out.println("完整类名称："+clazz.getName());
        System.out.println("类修饰符:"+clazz.getModifiers());
        System.out.println("父类:"+clazz.getSuperclass());
        //获取构造方法
        Constructor constructor = clazz.getConstructor(new Class[]{String.class});
        //创建一个实例，根据构造方法来传递参数
        Person person = (Person) constructor.newInstance("yan");
        System.out.println("name:"+person.getName());
        //获取方法
        Method method = clazz.getMethod("setName",new Class[]{String.class});
    	//调用方法
        method.invoke(person,"yanyong");//如果method是静态方法，invoke方法的第一个参数为null
        System.out.println("修改name"+person.getName());
        //获取属性
        Field field = clazz.getField("name");
    	//获取属性值
        Object o = field.get(person);
        System.out.println(o);

    }

private Class getClazz(){
        try {
            Class clazz = Class.forName("com.xxx.Person");
            return clazz;
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }

public class Person {


    public String name;

    public int age;

    public Person(String name){
        this.name = name;
    }

    public Person(){}
    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

#### 反射机制

查看Method.invoke源代码可以看到实际上调用目标方法委托给了MethodAccessor，MethodAccessor是一个接口，它有两个实现:DelegatingMethodAccessorImpl(委派模式)，NativeMethodAccessorImpl(通过本地方法来实现反射调用)。

```java
public Object invoke(Object obj, Object... args)
	throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, obj, modifiers);
            }
        }
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
            ma = acquireMethodAccessor();
        }
        return ma.invoke(obj, args);
    }
```



Method实例第一次反射调用都会生成委派实现，它所委派的具体实现便是一个本地实现。本地方法的实现很容易理解，即当进入虚拟机内部之后，我们已经获取到了Method实例指向的方法的具体地址，这个时候，反射的作用就是将参数准备好，然后调用进入目标方法。

```java
public static void  target(int i){
        new Exception("#"+i).printStackTrace();
    }


    public static void main(String[] args)throws Exception{

        Class clazz = Class.forName("ReflectionDemo");
        Method method = clazz.getMethod("target", int.class);
        method.invoke(null,0);
    }
```

执行结果如下：

```java
java.lang.Exception: #0
ReflectionDemo.target(ReflectionDemo.java:52)at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at ReflectionDemo.main(ReflectionDemo.java:60)

```

从下往上看，反射先调用Method.invoke方法，然后进入委派实现（DelegatingMethodAccessorImpl）,然后进入本地实现(NativeMethodAccessorImpl)，最后到达目标方法。

反射调用将委派实现作为中间层的作用是：Java的反射调用机制还设立了另外一种动态生成字节码的实现（动态实现），使用委派实现是为了能够在本地实现和动态实现中切换。

动态实现和本地实现相比，其效率要快上20倍，因为动态实现无需经过Java到c++再到Java的切换，但由于生成字节码十分耗时，仅仅调用一次，反而本地实现要快上3到4倍。

Java虚拟机设置了一个阀值15，当反射调用的次数在15之下时，采用本地实现，当达到15时，便开始动态生成字节码，并将委派实现的委派对象切换到动态实现。

```java
public  void test2()throws Exception {

        Class clazz = Class.forName("ReflectionDemo");

        Method method = clazz.getMethod("target", int.class);

        for (int i =0;i<20;i++){
            method.invoke(null,i);
        }

    }
    
 public static void  target(int i){
        new Exception("#"+i).printStackTrace();
 }    
```

执行结果如下：从16次开始，便切换到了动态实现。

```java
java.lang.Exception: #15
	at ReflectionDemo.target(ReflectionDemo.java:52)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)

java.lang.Exception: #16
	at ReflectionDemo.target(ReflectionDemo.java:52)
	at sun.reflect.GeneratedMethodAccessor1.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)

```



