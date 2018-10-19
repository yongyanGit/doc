#### 倒计时器：CountDownLatch

这个工具通常用来控制线程等待，它可以让某一个线程等待，直到倒计时结束，再执行。

下面是一个例子：

```java
public class CountDownLatchDemo implements Runnable {

    static final CountDownLatch end = new CountDownLatch(10);
    static final CountDownLatchDemo demo = new CountDownLatchDemo();

    @Override
    public void run() {
        try {
            Thread.sleep(new Random().nextInt(10) * 1000);
            System.out.println("check complete");
            end.countDown();//倒计时减一
        }catch (Exception e){
            e.printStackTrace();
        }

    }

    public static void main(String[] args) throws Exception{

        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for(int i = 0;i<10;i++){
            executorService.submit(demo);

        }
        end.await();//开始倒计时
        System.out.println("fire!");
        executorService.shutdown();
    }

}
```

上面的程序执行await方法后，要求主线程等待所有10个检查任务全部完成后，它才能够继续执行下去。

源码分析：

首先可以看到它的构造方法，在创建一个CoutDownLatch对象时会传入一个整形数字，这个数字就是用来倒计时的。

```java
 public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
 }
```

Sync继承自AbstractQueuedSynchronizer，主要实现了一些对锁的操作，如：获取锁、释放锁等。在sync的构造方法中会对锁的状态进行设置，并且只有当锁的状态为0时，CountDownLatch所在的线程才可以继续执行

```java
Sync(int count) {
     setState(count);//根据CountDownLatch传过来的参数进行设置锁的状态
 }

 int getCount() {
     return getState();//获取锁的状态
  }
```

* await()：主线程进行阻塞状态。

```java
 public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
  }
  
  public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())//如果线程被中断过，抛出中断异常
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)//获取锁
            doAcquireSharedInterruptibly(arg);//线程进入阻塞
   }
```

* tryAcquireShared()：尝试获取锁。在CountDownLoatch中，当它的锁的状态为0时返回true.

```java
protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
 }
```

如果尝试获取锁失败后，当前线程会进入阻塞

* doAcquireSharedInterruptibly

```java
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);//将线程加入到等待队列
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);//尝试获取锁
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&//获取失败，进入阻塞
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

* countDown()

```java
	public void countDown() {
        sync.releaseShared(1);//是否锁
    }
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {//尝试释放锁
            doReleaseShared();//如果锁全部释放完，唤醒等待线程
            return true;
        }
        return false;
    }
```

* tryReleaseShared()：释放锁.

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();//获取锁的状态
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))//cas函数设置锁的状态
            return nextc == 0;//如果锁的状态为0，即倒时器减为0，返回true
    }
}
```

* doReleaseShared()

```java
 private void doReleaseShared() {
       
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);//唤醒线程
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }

 private void unparkSuccessor(Node node) {
       
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);//唤醒线程
    }
```

