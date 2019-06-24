#### Java 多线程

#### 1. 线程状态

```java
/**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
```

* NEW：表示刚刚创建的线程，这种线程还没有开始执行，等待线程的start()方法调用，才表示线程开始执行。
* RUNNABLE：表示线程所需要的资源都已经准备好了，等待处理器的调用，如果线程在执行过程中遇到了synchronized同步块，就会进入BLOCKED阻塞状态，这时线程就会暂停执行，直到获得请求的锁。
* BLOCKED：线程获取synchronized同步锁失败，线程就会暂停，只有重新获取到锁，才会被唤醒。
* WAITING和TIME_WAITING 都表示等待状态，它们的区别是WAITING会进入一个无时间限制的等待，TIME_WAITING会进入一个有时限的等待。比如：wait()方法等待的线程在等待notify()方法，而通过join()方法等待的线程会等待目标线程的终止。而一旦到了期望的事件，线程就会再次进入执行，进入RUNNABLE状态。
* TERMINATED：当线程执行完毕，则进入TERMINATED状态，表示结束。



![threadstate](../images/thread/threadstate.jpeg)



#### 2. 创建线程

* 继承Thread类

```java
public class MyThread extends Thread {

	private int num = 10;
	private int i = 1;
	public MyThread(String name){
		super(name);
	}
	
	public void run() {
		
		while(true){
			if(++i > num) break;
			System.out.println(this.getName()+"---卖出了" +i+"张票");
		}
	}
	
	public static void main(String[] args) {
		
		MyThread thread1 = new MyThread("thread1");
		MyThread thread2 = new MyThread("thread2");
		MyThread thread3 = new MyThread("thread3");
		thread1.start();
		thread2.start();
		thread3.start();
	}
	
	

}
```

* 实现Runnable接口

```java
public class MyRunnable implements Runnable {
	
	int num = 10;
	private int i = 0;
	
	public void run() {
		while(true){
			if(++i > num) break;
			System.out.println(Thread.currentThread().getName()+"卖出了" +i+"张票");
		}
		
	}
	
	public static void main(String[] args) {
		
		MyRunnable runnable = new MyRunnable();
		Thread thread1 = new Thread(runnable);
		Thread thread2 = new Thread(runnable);
		Thread thread3 = new Thread(runnable);
		thread1.start();
		thread2.start();
		thread3.start();
		
	}

}
```

两种方式，推荐第二种方式，因为Java是单继承，如果继承了Thread类，就不能再继承其它的类了。

**run()方法与start方法的区别**：

如果直接调用Thread.run()方法，不行单独开一个线程去执行run()方法里面的逻辑代码，跟执行一个普通的方法是一样的。如下示例代码：

```java
public class Thread2 extends Thread {

	
	public Thread2(String name){
		super(name);
	}
	
	public void run() {
		System.out.println(Thread.currentThread().getName());
		
	}
	
	public static void main(String[] args) {
		
		Thread2 thread1 = new Thread2("thread1");
		System.out.println("start method ");
		thread1.start();
		System.out.println("run method");
		thread1.run();
	}

}
```

执行结果：

```
start method 
run method
main
thread1
```

我们可以从源码中对比下start()方法与run()方法的区别：

```java
public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)//如果线程不是就绪状态，抛出异常
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);//将线程添加到ThreadGAroup中

        boolean started = false;
        try {
            start0();//start0是一个本地方法，它实际是新建一个线程去执行run方法
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
```

```java
public void run() {
        if (target != null) {
            target.run();//target对象是一个runnable对象，线程调用run方法其实是调用runnable对象的run方法，并且不会新建线程。
        }
    }
```

#### 3. 终止线程

查看JDK会发现Thread提供了一个stop()方法，如果你使用了stop方法，就立即将一个线程终止。但是在JDK文档中，它是一个被废弃的方法，因为stop()方法太过于暴力，强制把执行到一半的线程终止，可能会引起一些数据的不一致的问题。查看JDK源码，它的底层实现好像是直接抛出一个异常来终止线程。

```java
 @Deprecated
    public final synchronized void stop(Throwable obj) {
        throw new UnsupportedOperationException();
    }
```

一个简单的例子：

```java
public class StopThread {
    private static User u = new User();
    static class InnerCLass extends Thread{
        @Override
        public void run() {
            while (true){
                synchronized (u){
                    int value = (int)(System.currentTimeMillis()/1000);
                    u.setId(value);
                    try {
                        Thread.sleep(100);
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                    u.setName(value);
                }
                Thread.yield();
            }
        }
    }

    static class ReadThread extends Thread{

        @Override
        public void run() {

            while (true){

                synchronized (u){
                    if (u.getId() != u.getName()){
                        System.out.println(u.getId()+":"+u.getName());
                    }
                }
                Thread.yield();
            }
        }
    }

    public static void main(String[] args)throws Exception{

        new ReadThread().start();
        while (true){

            Thread t = new InnerCLass();
            t.start();
            Thread.sleep(150);
            t.stop();
        }
    }
}

class User{

    private int id;

    private int name;

    public int getId() {
        return id;
    }

    public int getName() {
        return name;
    }

    public void setName(int name) {
        this.name = name;
    }

    public void setId(int id) {
        this.id = id;
    }
}
```

代码执行后的输出结果：

```
1537495908:1537495907
1537495911:1537495910
```

可以看到虽然对id和name的赋值是在一个同步模块中执行，但是它们的值却会存在不一样的值。

其实我们可以通过设置一个标志来停止线程，通过设置一个标志的true或false来中断线程，类似如下的代码：

```java
public void run() {
  while (true){
       if (flag){
           System.out.println("exit......");
           break;
       }
       synchronized (o){
           System.out.print(System.currentTimeMillis());
               ....
      }
 }
 
 public void setStopFlag(boolean flag){
        this.flag = flag;
 }
```



#### 4. 线程中断

线程中断并不会使线程立即退出，而是给线程发送一个通知，告知目标线程，有人希望它退出。在目标线程接到通知后如何处理，则完全有目标线程自行决定。

与线程中断有关的方法：

```java
 public void interrupt()
```

通知目标线程中断，也就是设置中断标志位。如果此时线程已经（调用```wait()、sleep()、join()```）进入阻塞状态时，调用interrupt()方法，会抛出一个InterruptedException异常。

如果线程阻塞在java.nio.channels.InterruptibleChannel的IO上，Channel将会被关闭，线程被设置为中断状态，并抛出ClosedByInterruptException异常。

如果线程阻塞在java.nio.channels.Selector上，线程会被设置为中断状态，select方法会马上返回，类似调用wake up状态。

```java
public static boolean interrupted()//判断是否被中断，但是会清除中断标志
private native boolean isInterrupted()//判断是否被中断
```

#### 5. Java中wait()与notify()

> wait()与notify()必须在同步模块中执行，并且执行该方法的线程持有锁。

* wait()：让当前持有锁的线程进入Object对象的等待队列，并且释放它持有的锁，在这个等待队列中，可能会有多个线程，因为系统运行多个线程同时等待某一个对象。
* notify()：当调用notify()方法时，会从等待队列中随机的选择一个线程，并将其唤醒。注意这个选择不是公平的，并不是先等待的线程会优先被选择。当然当线程被唤醒后（Runnable)，并不是马上执行后续的代码，它还要去获取监视器（锁），只有成功获取后，才会继续往后执行。
* notifyAll()：唤醒所有的线程

一个简单的例子：

```java
public class ThreadDemo extends Thread{
	
	Object object = null;
	private String name ;
	public ThreadDemo(Object object,String name){
		this.object = object;
		this.name = name;
	}
	public void run()  {
		
		while(true){
			synchronized(object){
				try {
					Thread.sleep(1000);
					object.notify();//唤醒锁上的一个线程，被唤醒的线程处于就绪状态，等待被处理器调用
					System.out.println(name +"-----run");
					object.wait();//阻塞,释放锁
				}catch (Exception e) {
					e.printStackTrace();
				}
			} 
		}
	}
	
	
	public static void main(String[] args) {
		Object o = new Object();
		ThreadDemo thread1 = new ThreadDemo(o, "thread1");
		ThreadDemo thread2 = new ThreadDemo(o, "thread2");
		thread1.start();
		thread2.start();
		
	}
```

#### 6. Java Thread.sleep()方法

程序执行sleep方法后，线程由运行状态进入阻塞状态，阻塞的时间由传入的时间决定。时间过期后，线程程变成就绪状态，等待cpu的调度。sleep()方法如果在同步模块中执行，不会释放锁资源。

#### 7. Java Join()方法

让主线程等待它的子线程死亡，即当一个线程中启动新的线程后，如果子线程调用了join()方法，主线程会等待子线程死亡后才会继续往下执行。

一个简单的例子：

```java
public class ThreadJoin extends Thread{
	
	public void run() {
		try{
			System.out.println("join is start");
			for(int i = 0;i<10;i++){
				
				Thread.sleep(1000);
				System.out.println("thread is run:"+i);
			}
			System.out.println("join is end");
		}catch(Exception e){
			e.printStackTrace();
		}
	}

	public static void main(String[] args) throws Exception {
		
		ThreadJoin threadJoin = new ThreadJoin();
		threadJoin.start();
		threadJoin.join();
		System.out.println("main is end");
	}

}
```

执行结果：

```
join is start
thread is run:0
thread is run:1
thread is run:2
thread is run:3
thread is run:4
thread is run:5
thread is run:6
thread is run:7
thread is run:8
thread is run:9
join is end
main is end
```

可以从源码中看到其实join方法调用了wait方法，将当前拥有锁的线程（主线程）阻塞掉了。当子线程执行完之后，会调用notifyAll()方法，唤醒等待的线程。因此有一点需要注意：不要在应用程序中，在Thread对象实例上使用类似wait()或者notify()等方法，这很有可能会影响系统API工作，或者被系统API所影响。

```java
public final synchronized void join(long millis)
    throws InterruptedException {
        if (millis == 0) {
            while (isAlive()) {//如果子线程还活着
                wait(0);//主线程进入阻塞
            }
        } 
    }
```

#### 8. Java Thread.yield()方法

让线程从运行状态进入到就绪状态，让其它相同优先级的线程获取到执行权限，但是执行yield()方法后，其它线程不一定会获取到执行权限。如果yield()方法在同步模块中执行，不会释放锁。

#### 9. 线程组

如果在一个系统中线程数量很多，并且功能分配比较明确，就可以将相同的功能的线程放在一个线程组里面。

```java
public class ThreadGroupDemo implements Runnable {

    public static void main(String[] args){

        ThreadGroup threadGroup = new ThreadGroup("printGroup");
        Thread t1 = new Thread(threadGroup,new ThreadGroupDemo(),"t1");
        Thread t2 = new Thread(threadGroup,new ThreadGroupDemo(),"t2");
        t1.start();
        t2.start();
        System.out.println(threadGroup.activeCount());
        threadGroup.list();

    }

    @Override
    public void run() {

        String groupName = Thread.currentThread().getThreadGroup().getName()
                +"-"+Thread.currentThread().getName();
        System.out.println(groupName);

    }
}
```

#### 10. 守护线程

Java线程分为两种线程：用户线程(User Thread)、守护线程(Daemon Thread)。

用户线程可以认为是系统的工作线程，它会完成这个程序应该要完成的业务操作。如果用户线程全部结束，意味着这个程序实际上无事可做了，守护线程要守护的对象已经不存在了，那么整个程序就自然应该结束。因此，当一个Java应用内只有守护线程时，Java虚拟机就会自然退出。很好理解，守护线程是用来辅助用户线程的，如公司的保安和员工，各司其职，当员工下班离开后，保安自然下班了。

守护线程不适合用于输入输出或者计算机等操作，因为用户线程执行完毕，程序就dead了，适用于辅助用户线程的场景，如JVM的垃圾回收，内存管理都是守护线程。

```java
public class DaemonDemo {

    public static class DaemonT extends Thread{


        @Override
        public void run() {

            while (true){

                System.out.println("i am alive");
                try{
                    Thread.sleep(1000);

                }catch (Exception e){
                    e.printStackTrace();
                }

            }

        }
    }


    public static void main(String[] args)throws Exception{

        Thread thread = new DaemonT();
        thread.setDaemon(true);
        thread.start();
        Thread.sleep(2000);

    }


}
```

在上面的例子中，thread被设置为守护线程，系统中只有主线程main为用户线程，在主线程退出之前，守护线程会一直打印```i am alive```,当主线程休眠结束后，整个应用没有了用户线程，随之jvm会退出，守护线程也就退出了。

另外如果通过Java线程池创建的线程，默认是用户线程。

#### 11. 线程优先级

Java线程中，优先级高的线程在竞争资源时会更有优势，更可能抢占资源，当然优先级高的线程也有可能会抢占失败。

在Java中国使用1到10表示线程的优先级，数字越大则优先级越高。一般可以使用内置的三个静态变量来表示：

```java
/**
     * The minimum priority that a thread can have.
     */
    public final static int MIN_PRIORITY = 1;

   /**
     * The default priority that is assigned to a thread.
     */
    public final static int NORM_PRIORITY = 5;

    /**
     * The maximum priority that a thread can have.
     */
    public final static int MAX_PRIORITY = 10;
```

在Java中我们可以通过调用Thread的```setPriority```方法来设置线程的优先级。



