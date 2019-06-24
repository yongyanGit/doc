#### Java 引用

Java引用分为强引用、软引用、弱引用、虚引用。

1. 强引用(StrongReference)

强引用是使用最普遍的引用。如果一个对象具有强引用，那垃圾回收器绝不会回收它。当内存不足时，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会随意回收具有强引用的对象来解决内存不足的问题。(强引用也就是我们平时A a = new A())

2. 软引用(SoftRefrence)

如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足，就会把这些对象回收。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可以用来实现内存敏感的高速缓存。软引用可以和一个引用队列(ReferenceQueue)联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把软引用加入到与之关联的引用队列中。

```java
//-Xmx10M -XX:+UseSerialGC
public class SoftRef {
	
	public static class User{
		
		public User(int id,String name){
			this.id = id;
			this.name = name;
		}
		
		public int id;
		
		public String name;
		
		@Override
		public String toString() {
			
			return "id="+id+";name="+name;
		}
		
	}
	
	public static void main(String[] args){
		
		User u = new User(1,"yanyong");
		SoftReference<User> reference = new SoftReference<SoftRef.User>(u);
		u= null;
		
		System.out.println(reference.get());
		System.gc();
		System.out.println("after gc");
		System.out.println(reference.get());
		byte[] b = new byte[1024*939*7];
		System.gc();
		System.out.println(reference.get());
	}

}
```

执行结果：

```
id=1;name=yanyong
after gc
id=1;name=yanyong
null
```

上面的代码中创建了一个user对象，并且将强引用设置为null，如果堆内存还没有满，就可以通过软引用来获取user对象，后面当创建字节数组后，堆内存不够，然后执行gc就会回收软引用。

每一个软引用都可以附带一个引用队列，当对象的可达性状态发生改变时，软引用对象就会进入引用队列。通过这个引用队列，可以跟踪对象的回收情况：

```java
//-Xmx10M -XX:+UseSerialGC
public class SoftRefQ {
	
public static class User{
		
		public User(int id,String name){
			this.id = id;
			this.name = name;
		}
		
		public int id;
		
		public String name;
		
		@Override
		public String toString() {
			
			return "id="+id+";name="+name;
		}
		
	}

	static ReferenceQueue<User> softQueue = null;
	
	public static class CheckRefQueue extends Thread{
		
		@Override
		public void run() {
			while(true){
				
				if(softQueue != null){
					UserSoftReference obj = null;
					try{
						obj = (UserSoftReference)softQueue.remove();
					}catch(Exception e){
						e.printStackTrace();
					}
					
					if(obj != null){
						System.out.println("id"+obj.uid+"is delete");
					}
				}
			}	
		}
	}
	
	
	public static class UserSoftReference extends SoftReference<User>{
		int uid;
		public UserSoftReference(User referent,ReferenceQueue<? super User> q){
			super(referent,q);
			this.uid = referent.id;
		}
	}
	
	public static void main(String[] args) throws Exception{
		
		Thread t = new CheckRefQueue();
		t.setDaemon(true);
		t.start();
		User u = new User(1,"yanyong");
		softQueue = new ReferenceQueue<>();
		UserSoftReference userSoftReference = new UserSoftReference(u, softQueue);
		u = null;
		System.out.println(userSoftReference.get());
		System.gc();
		System.out.println("After gc");
		System.out.println(userSoftReference.get());
		byte[] b = new byte[1024*939*7];
		System.gc();
		System.out.println(userSoftReference.get());
		Thread.sleep(1000);
	}
}
```

执行结果：

```
id=1;name=yanyong
After gc
id=1;name=yanyong
id1is delete
null
```

3. 弱引用(WeekReference)

弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收线程扫描它所管辖的内存区域，一旦发现只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收线程是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。

```java
public class WeakRef {
	
	public static class User{
		
		public User(int id,String name){
			this.id = id;
			this.name = name;
		}
		
		public int id;
		
		public String name;
		
		@Override
		public String toString() {
			
			return "id="+id+";name="+name;
		}
		
	}
	
	public static void main(String[] args){
		
		User u = new User(1,"yan");
		WeakReference<User>   userWeak = new WeakReference<WeakRef.User>(u);
		u = null;
		System.out.println(userWeak.get());
		System.gc();
		System.out.println("after gc");
		System.out.println(userWeak.get());
		
	}
	
}
```

执行结果：

```
id=1;name=yan
after gc
null
```

弱引用和软引用一样，在构造弱引用时，也可以指定一个引用队列，当弱引用对象被回收时，就会被加入到指定的引用队列，通过这个队列可以追踪对象的回收情况。

4. 虚引用（PhantomReference）

虚引用与其它几种引用不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可以被垃圾回收器回收。

虚引用主要用来跟踪对象被垃圾回收器回收的活动。虚引用与软引用和弱引用的一个区别在于：虚引用必须与引用队列联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收内存之前，把这个虚引用加入到与之关联的引用队列中。

