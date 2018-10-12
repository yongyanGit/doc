#### Java信号量（Semaphore）

型号量用来指定多个线程同时访问某一个资源。在构造信号量时，需要指定信号量的准入数，即同时能申请多少个许可，每个线程每次只申请一个许可，这就相当与指定同时有多少个线程可以访问某一个资源。它的常用方法如下：

```java
public void acquire();
public void acquireUninterruptibly();
public boolean tryAcquire();
public boolean tryAcquire(long timeout, TimeUnit unit);
public void release();
```

* acquire()：尝试获取一个准入许可，如果无法获得，则线程会等待，直到有线程释放一个许可或当前线程被中断。
* acquireUninterruptibly()：与acquire()方法类似，但是它不响应中断。
* tryAcquire()：尝试获取一个许可，如果成功返回true,如果失败返回false,它不会进行等待。
* release()：线程访问资源结束后，释放一个许可，使其它的等待许可的线程可以进行资源访问。

例子：

```java
public class SemapDemo implements Runnable{

    final Semaphore semaphore = new Semaphore(5);

    @Override
    public void run() {
        try {
            semaphore.acquire();//获取许可
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getId()+":done");
            semaphore.release();//释放许可,如果不释放，其它等待线程无法获取许可来访问资源。
        }catch (Exception e){
            e.printStackTrace();
        }

    }


    public static void main(String[] args){

        ExecutorService executorService = Executors.newFixedThreadPool(20);
        final SemapDemo semapDemo = new SemapDemo();

        for(int i =0;i<20;i++){
            executorService.submit(semapDemo);
        }


    }
}
```

#### 源码分析

Semaphore构造方法：Semaphore有两种模式，公平模式和非公平模式。公平模式就是获取许可证按照先到先获取的原则，而非公平模式则是抢占式的。在构造Semaphore对象时，必须指定信号量，并且如果不指定是否公平，默认创建非公平模式。对于信量的获取和释放委托给了它的内部类Sync，对应公平与非公Sync有两个子类：FairSync、NonfairSync。

```java
public Semaphore(int permits) {
	sync = new NonfairSync(permits);
}

 public Semaphore(int permits, boolean fair) {
 	sync = fair ? new FairSync(permits) : new NonfairSync(permits);
 }
```

* ```acquire()```：获取许可。

```java
public void acquire() throws InterruptedException {
	sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
	if (Thread.interrupted())//如果线程被中断过，抛出异常
    	throw new InterruptedException();
 	if (tryAcquireShared(arg) < 0)//尝试获取许可
        doAcquireSharedInterruptibly(arg);//如果获取许可失败，将线程加入到等待队列
}
```

```tryAcquireShared()```在公平模式和非公平模式中的实现是不同的，先从非公平模式分析：

```java
protected int tryAcquireShared(int acquires) {
	return nonfairTryAcquireShared(acquires);
}
//非公平模式
final int nonfairTryAcquireShared(int acquires) {
	for (;;) {
    	int available = getState();//获取信号量
        //剩余的信号量
        int remaining = available - acquires;
        //信号量小于0说明信号量已经全部被用完了
        //通过compareAndSetState方法更新信号量，如果返回true，表示获取信号量成功.
        if (remaining < 0 ||compareAndSetState(available, remaining))
            return remaining;//返回剩余的信号量
     }
 }
```

公平模式下获取许可：

```java
protected int tryAcquireShared(int acquires) {
for (;;) {
	if (hasQueuedPredecessors())//判断等待队列中是否线程在等待获取信号量
        return -1;
    int available = getState();//剩余的信号量
    int remaining = available - acquires;
    if (remaining < 0 ||compareAndSetState(available, remaining))
        return remaining;
    }
}
```

公平模式与非公平模式的区别在于，公平模式下，线程获取信号量要先判断是否有线程在等待着获取信号，如果有，那么按照先到先获取的原则，该线程要进入等待队列。

尝试获取许可失败后，线程会进入等待队列中:

```java
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);//将线程加入到等待队列
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();//获取上一个节点
                if (p == head) {//如果它的上一个节点是头节点，则尝试去获取许可
                    int r = tryAcquireShared(arg);//尝试获取许可，并返回剩余的信号量
                    if (r >= 0) {//如果返回的许可大于等于0，说明获取许可成功
                        setHeadAndPropagate(node, r);//设置头节点
                        p.next = null; // help GC//将原来的头节点的指针置为空
                        failed = false;
                        r如果返回的许可大于等于0，说明获取许可成功eturn;
                    }
                }
                
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())//将线程阻塞掉
                    throw new InterruptedException();//如果线程在阻塞期间被中断，抛出异常
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

```addWaiter()```:将线程加入到等待队列的末尾。

```java
 private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);//包装成node节点
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {//将尾节点设置为node
                pred.next = node;
                return node;
            }
        }
        enq(node);//初始化链表
        return node;
    }
```

```setHeadAndPropagate()```：设置头节点和唤醒后续节点

```java
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; 
        setHead(node);// 设置头节点
       //如果还要信号量，则唤醒后续的节点
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();//唤醒节点
        }
    }
```

```doReleaseShared()```：唤醒节点。

```java
private void doReleaseShared() {
        
	for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                //如果头节点的状态是signal表示可以唤醒后续的节点
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);//唤醒头节点的下一个节点
                }
                //如果信号量已经被用完，将节点状态设置为PROPAGATE
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
 }



//唤醒节点
private void unparkSuccessor(Node node) {
       
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;//唤醒头节点的下一个节点
        if (s == null || s.waitStatus > 0) {
            s = null;
            //如果节点已经被取消，则从后往前循环查找最后一个状态小于0的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);//唤醒节点
    }
```

```shouldParkAfterFailedAcquire()```：设置当前节点的前置节点的状态设置为SIGNAL。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            
            return true;
        if (ws > 0) {//ws大于0 说明前置节点已经被取消了
            do {
                node.prev = pred = pred.prev;//重新设置当前节点的前置节点
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //将当前节点的前置节点设置为SIGNAL，表示有后置节点需要它来唤醒
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```



* ```release()```:释放许可。

```java
 public void release() {
        sync.releaseShared(1);
    }

public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {//先尝试是否许可
            doReleaseShared();//唤醒下一个节点
            return true;
        }
        return false;
  }
```

```tryReleaseShared()```：是否许可

```java
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();//当前信号量
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))//通过cas函数设置信号量
                    return true;
            }
}
```

