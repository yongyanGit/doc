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





