#### Java 可重入锁：ReetrantLock

可重入锁是指这种锁可以反复进入，但是这里的反复仅仅局限于同一个线程。ReetrantLock分为公平锁和非公平锁，它们的区别是获取锁的机制上是否公平，ReentrantLock通过一个队列来维护获取锁的线程，如果是公平锁，则会按照先来后到的顺序依次获取锁，非公平锁，只要锁是空闲的，不管是不是位于队列的第一个线程，都会去尝试获取锁。

ReentrantLock的几个重要方法：

* lock()：获得锁，如果锁已经被占用，则等待。
* lockInterruptibly()：获得锁，但优先响应中断。
* tryLock()：尝试获得锁，如果成功，返回true,失败返回false。该方法不等待，立即返回。
* tryLock(long time,TimeUnit unit)：在给定的时间内尝试获得锁。
* unlock()：释放锁。

一个简单的例子（公平锁），：

```java
public class FairLock implements Runnable {
	//默认是非公平锁
    public static ReentrantLock fairLock = new ReentrantLock(true);

    @Override
    public void run() {

        while (true){
            try {
                fairLock.lock();//获取锁
                System.out.println(Thread.currentThread().getName()+"get a fair lock");
                Thread.sleep(100);
            }catch (Exception e){
               e.printStackTrace();
            } finally {
                fairLock.unlock();//释放锁
            }
        }

    }

    public static void main(String[] args){

       FairLock fairLock = new FairLock();

       Thread thread1 = new Thread(fairLock,"t1");

       Thread thread2 = new Thread(fairLock,"t2");

       thread1.start();
       thread2.start();


    }
}
```

执行代码后，观察运行结果，发现thread1 和thread2轮流执行。

ReentrantLock与synchronized相比较，后者如果一个线程在等待锁，那么它就必须一直保持等待。而ReetrantLock，则提供另外一种可能，那就是线程可以被中断，也就是在等待的过程中，程序可以根据需求取消对锁的请求。

如下例子：

```java
public class IntLock implements Runnable {

    public static ReentrantLock lock1 = new ReentrantLock();
    public static ReentrantLock lock2 = new ReentrantLock();
    int lock;

    public IntLock(int lock){
        this.lock = lock;
    }

    @Override
    public void run() {

        try {

            if(lock == 1){
                lock1.lockInterruptibly();
                Thread.sleep(500);
               lock2.lockInterruptibly();

            }else {

                lock2.lockInterruptibly();
                Thread.sleep(500);
                lock1.lockInterruptibly();

            }

        }catch (InterruptedException e){
            e.printStackTrace();
        }finally {
            if(lock1.isHeldByCurrentThread()){
                lock1.unlock();
            }
            if(lock2.isHeldByCurrentThread()){
                lock2.unlock();
            }

            System.out.println(Thread.currentThread().getName()+"退出");
        }
    }


    public static void main(String[] args)throws InterruptedException{

        IntLock r1 = new IntLock(1);

        IntLock r2 = new IntLock(2);

        Thread t1 = new Thread(r1);

        Thread t2 = new Thread(r2);

        t1.start();
        t2.start();
        Thread.sleep(1000);
        t2.interrupt();

    }
}
```

线程t1和t2启动后，t1先占用了lock1，再占用lock2,t2先占用lock2在请求lock1。因此很容易形成t2和t1之间互相等待。我们在获取锁时，使用lockInterruptibly()方法，它在等待锁的过程中可以响应中断。主线程在休眠时，这两个线程处于死锁状态，然后执行t2.interrupt()方法，t2线程被中断，放弃了对lock1的申请，同时也释放了lock2。然后t1线程就可以顺利的获取lock2继续执行下去了。



#### 源码分析

**公平锁**

我们从获取锁lock()开始解析：

```java
public void lock() {
   sync.lock();
}
```

因为是公平锁，所以```sync.lock()```调用的是ReetrantLock内部类FairSync的lock()方法

```java
final void lock() {
   acquire(1);
}
```

接着进入```acquire()```方法

```java
 public final void acquire(int arg) {
   if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      selfInterrupt();
 }
```

* 首先通过tryAcquire()去获取锁，如果获取成功，直接返回，否则则将该线程加入到等待队列。
* 在尝试获取锁失败后，调用addWaiter方法，将该线程加入到队列末尾，等待获取锁。
* 线程加入到队列后，会调用acquireQueued()方法，根据公平原则获取锁。
* 如果线程在沉睡期间被中断过，会进行自我中断一次。

tryAcquire()方法：线程尝试获取锁。

```java
protected final boolean tryAcquire(int acquires) {
	final Thread current = Thread.currentThread();//获取当前线程
    int c = getState();//获取锁的状态
    if (c == 0) {//如果锁的状态是0，表示可以获取锁
        //因为是公平锁，所以需要判断当前请求锁的线程是否是等待队列中的第一个，如果不是，则不能获取
    	if (!hasQueuedPredecessors() &&
        	compareAndSetState(0, acquires)) {//通过cas函数设置锁状态
            //如果cas函数执行成功，说明获取锁成功，将锁的持有者设置为当前线程
            setExclusiveOwnerThread(current);
            return true;
         }
     }
    //如果锁已经被获取，则判断持有锁的线程是否是当前请求获取锁的线程
     else if (current == getExclusiveOwnerThread()) {
         //因为ReetranrLock是可重入锁，所以两个线程相等时，直接将锁的状态加1
         //后续释放锁时，调用一次unlock()，nexc的数量减1，直到为0，该锁就可以重新被其它线程获取
          int nextc = c + acquires;
          if (nextc < 0)
              throw new Error("Maximum lock count exceeded");
          setState(nextc);//设置锁的状态
          return true;
      }
     return false;
 }
```

```addWaiter(Node.EXCLUSIVE)```，将当前请求获取锁的线程添加到链表的末尾。```Node.EXCLUSIVE```是指锁的模式即独占锁。

线程添加到等待队列时，首先会被封装成一个Node对象，这个对象保存有节点的上一个、下一个节点，线程状态等信息。

```java
static final class Node {
       //共享锁
        static final Node SHARED = new Node();
        //独占锁
        static final Node EXCLUSIVE = null;
        //线程被取消标志
        static final int CANCELLED =  1;
       //需要唤醒它的后继节点
        static final int SIGNAL    = -1;
        //线程被阻塞，等待condition唤醒
        static final int CONDITION = -2;
        
        static final int PROPAGATE = -3;

        //新创建的节点状态为0。
        volatile int waitStatus;

        //上一个节点
        volatile Node prev;

        //下一个节点
        volatile Node next;
        //节点持有的线程
        volatile Thread thread;

        //区别队列是“独占锁”队列，还是“共享锁”队列
        Node nextWaiter;
        
       //如果是共享锁，返回true
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        //返回上一个节点
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

将Node对象添加到链表

```java
private Node addWaiter(Node mode) 
    //将Thread封装进一个Node对象中
	Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;//尾节点
   	if (pred != null) {
    	node.prev = pred;//将新增的节点的上一个节点指向原来的尾节点
        //通过cas函数将尾节点设置为node
        if (compareAndSetTail(pred, node)) {
        	pred.next = node;//如果cas函数执行成功，则将原来的尾节点的下一个节点指向node
            return node;
        }
     }
	//初始化链表
     enq(node);
     return node;
    }
```

在```addWarter()```方法中会调用```enq()```方法来初始化等待队列，它是一个链表。它除了初始化队列外，还是为了确保新增的节点会添加到链表的尾部。因为在并发环境中，cas函数有可能执行失败的。

在```enq()```方法中，通过一个死循环不断的将node节点添加到链表的尾部，直到成功为止。

```java
private Node enq(final Node node) {
	for (;;) {
    Node t = tail;
    if (t == null) { // Must initialize
        if (compareAndSetHead(new Node()))
            tail = head;//初始化链表
         } else {//将node添加到链表的末尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
          }
      }
 }
```

```acquireQueued()```尝试获取锁，如果获取锁成功，则会将头节点设置为持有锁的线程，并返回中断标志。

如果获取失败，则会调用```LockSupport.park(this);```方法，使当前线程进入睡眠状态，直到被它的上一个节点唤醒。

```java
boolean acquireQueued(final Node node, int arg) {
	boolean failed = true;
    try {
    boolean interrupted = false;//线程在睡眠期间是否有过中断
    for (;;) {
    	//“当前节点”的上一个节点("当前节点"是指在队列中等待的线程)
    	final Node p = node.predecessor();
    	//如果"当前节点"的上一个节点是头节点，说明"当前节点"是队列中的第一个节点，尝试获取锁
        if (p == head && tryAcquire(arg)) {
            setHead(node);//如果成功获取到了锁，则当前节点节点设置成头节点
            p.next = null; // help GC
            failed = false;
            return interrupted;//返回中断标志
         }
         //获取锁失败，将线程进入睡眠，等待它的上一个节点唤醒
         if (shouldParkAfterFailedAcquire(p, node) &&
             parkAndCheckInterrupt())
             interrupted = true;
       	  }
        } finally {
            if (failed)
            	cancelAcquire(node);
        }
```

```shouldParkAfterFailedAcquire```方法：它主要是有两个作用，一个是将等待队列中的状态为```CANCELLED```的节点去掉，另一个就是将新添加的节点的上一个节点的状态设置为```SIGNAL```，因为在节点释放锁时，会根据该状态来判断是否有后续节点需要自己来唤醒。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;//将被取消的线程去掉
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);//设置上一个节点的状态
    }
    return false;
}
```

```parkAndCheckInterrupt```方法，阻塞线程，返回线程的中断标志

```java
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);//阻塞线程
        return Thread.interrupted();//返回中断标志，该方法会清除掉中断标志
  }
//前面的interrupted方法会将中断标志清除掉，所以需要再次中断线程
 static void selfInterrupt() {
        Thread.currentThread().interrupt();
  }
```



**非公平锁**

因为是非公平锁，所以```sync.lock()```调用的是ReetrantLock内部类NonfairSync的lock()方法。非公平锁不管等待队列是否有线程在等待获取锁，它首先会通过cas函数去尝试获取锁，如果cas函数执行成功，接着将持有锁的线程设置为当前线程；如果获取失败则继续往后执行。

```java
final void lock() {
	if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
     else
        acquire(1);
}
```

```acquire()方法```：acquire()方法的逻辑跟公平锁是一样的，都是首先去尝试获取锁，如果获取失败，则会将线程加入到等待队列，然后等待它的上一个线程来唤醒它。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

非公平锁与公平锁的差异在于尝试获取锁的实现方式，在公平锁

```java
final boolean nonfairTryAcquire(int acquires) {
	final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
         }
     }
     else if (current == getExclusiveOwnerThread()) {
     	int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
         }
         return false;
     }
```

#### 释放锁

ReentrantLock在获取锁的时候，会通过cas函数设置锁的状态，同时使用一个字段来保存成功获取锁的次数，每成功获取一次，数量加1。因此在释放锁的时候，没调用一次```unlock()```方法，锁的状态减1，直到为0时，其它线程才可以重新获取锁。

```java
 public void unlock() {
        sync.release(1);
    }
```

```release()方法```:先通过```tryRelease()```方法释放锁，如果该线程持有的锁都已经释放完成了，则根据线程的状态来判断是否需要唤醒下一个沉睡的线程。

```java
 public final boolean release(int arg) {
 	if (tryRelease(arg)) {
       	Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

先调用```tryRelease()```方法，先尝试释放锁，将锁的状态减1，如果此时状态为0，说明该线程已经将持有的锁都释放了，然后就可以将持有锁的线程设置为null。

```java
 protected final boolean tryRelease(int releases) {
 	int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
    	throw new IllegalMonitorStateException();
     boolean free = false;
     if (c == 0) {
         free = true;
         setExclusiveOwnerThread(null);
      }
      setState(c);
      return free;
  }
```

```unparkSuccessor()```方法：唤醒下一个等待队列中的线程。如果需要唤醒的下一个节点被取消了，则会从等待队列的尾部开始循环，找出最后一个状态<=0的节点，并唤醒它。

```java
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)//从尾部开始循环
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

