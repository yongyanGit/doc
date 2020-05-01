#### ReadWriteLock读写锁

读写锁允许多个线程同时读，但是，考虑到数据的完整性，写写操作和读写操作之间需要相互等待。

* 读-读不互斥：读读之间不阻塞。
* 读-写互斥：读阻塞写，写也会阻塞读。
* 写写互斥：写写之间互相阻塞。

例子：从下面的例子可以看到，读操作会并行执行，写操作会阻塞等待读操作全部完成后，在依次获取锁。

```java
public class ReadWriteLockDemo {

    private static Lock lock = new ReentrantLock();

    private static ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();

    private static Lock readLock = readWriteLock.readLock();

    private static Lock writeLock = readWriteLock.writeLock();

    private int value;

    public Object handleRead(Lock lock)throws Exception{
        try {
            lock.lock();
            Thread.sleep(1000);
            return value;

        }finally {
            lock.unlock();
        }
    }

    public void handleWrite(Lock lock,int index) throws Exception{
        try {
            lock.lock();
            Thread.sleep(1000);
            value = index;
        }finally {
            lock.unlock();
        }

    }


    public static void main(String[] args){

        final ReadWriteLockDemo demo = new ReadWriteLockDemo();
        Runnable readRunnable = new Runnable() {
            @Override
            public void run() {
                try {
                    Object v = demo.handleRead(readLock);
                    System.out.println(v);

                   // demo.handleRead(lock);
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        };

        Runnable writeRunnable = new Runnable() {
            @Override
            public void run() {
                try {
                    demo.handleWrite(writeLock, new Random().nextInt());
                   // demo.handleWrite(lock,new Random().nextInt());
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        };

        for (int i =0;i<20;i++){
            new Thread(readRunnable).start();
        }

        for(int i = 0;i <20;i++){
            new Thread(writeRunnable).start();
           // new Thread(readRunnable).start();
        }

    }

}
```

#### 源码分析

在ReentrantReadWriteLock类中定义了一个Sync静态内部类，继承自AbstractQueuedSynchronizer，并实现了CLH算法。Sync有两个子类：FairSync和NonfairSync，对应公平锁和非公平锁。既然是读写锁，因此在ReentrantReadWriteLock内部分别有两个类ReadLock和WriteLockd对应着读锁，写锁。

Sync静态类里面有用一些常量来保存锁的类型如：独占锁、共享锁，已经锁的数量，如下：

```java
static final int SHARED_SHIFT   = 16;
//1左移动16位：0000 0000 0000 0000 1000 0000 0000 0000
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
//1左移动16位再减去1：0000 0000 0000 0000 0111 1111 1111 1111
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
//独占锁的标志：0000 0000 0000 0000 0111 1111 1111 1111
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
    
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
       
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

* SHARED_SHIFT：共享锁占用的位数。在读写锁中，读锁是共享锁，可以同时支持多个线程访问资源；相反的写锁的独占锁。
* SHARED_UNIT：共享锁的最小单元。每增加一个共享锁，就在原来数量的基础上增加一个SHARED_UNIT。如ReentrantReadWriteLock中通过cas函数设置锁的状态：

```java
compareAndSetState(c, c + SHARED_UNIT))
```

并且在ReentrantReadWriteLockh中，共享锁存储在int(32位)型字段的高16位。

* MAX_COUNT：锁的最大允许数量
* EXCLUSIVE_MASK：用来计算独占锁。
* sharedCount(int c)：获取共享锁的状态，共享锁占int c 的高16位，所以往右边移动16位就可以得到共享锁的数量
* exclusiveCount(int c)：获取独占锁的数量。因为独占锁占int c的低16位，所以c&EXCLUSIVE_MASK运算就可以得出独占锁的数量。

先来看读锁：

```java
//获取锁
public void lock() {
	sync.acquireShared(1);
}
//调用父类中的获取共享锁
public final void acquireShared(int arg) {
	if (tryAcquireShared(arg) < 0)//尝试获取共享锁
        doAcquireShared(arg);
}
```

* ```tryAcquireShared()```：尝试获取共享锁(读锁)。首先假如当前持有的锁是独占锁，那么会直接返回-1，因为独占锁与其它锁互斥，获取共享锁失败。如果当前持有的锁是共享锁，那么会根据当前读写锁的公平性质来判断（公平与非公平）：如果是公平锁，那么只要阻塞队列中有等待的线程，请求锁的线程就必须进入阻塞队列；如果是非公平锁，就需要根据阻塞队列中的第一个线程是获取独占锁还是公平锁来判断，如果是请求获取一个独占锁，那么根据公平性质该请求线程就必须进入阻塞队列，这样做是防止获取独占锁的线程一直获取不到锁。

```java
 protected final int tryAcquireShared(int unused) {
           
            Thread current = Thread.currentThread();//获取当前线程
            int c = getState();//获取锁的状态
     		//如果独占锁大于0，并且当前线程不等于持有锁的线程直接返回-1，获取读锁失败
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
     		//获取共享锁的数量
            int r = sharedCount(c);
     		//判断线程是否要阻塞（公平锁和非公平锁的区别）
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                //设置锁的状态
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    //当前线程是第一个获取共享锁的线程
                    firstReader = current;//记录第一个获取锁的线程
                    firstReaderHoldCount = 1;//记录获取锁的次数
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    //HoldCounter记录线程id和它获取锁的次数，该对象存放在ThreadLocal中
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        //从ThreadLocal中从新获取HoldCounter对象
                        //cachedHoldCounter缓存最后一个线程的HoldCount信息
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```

```readerShouldBlock()```：方法，公平锁与非公平锁的唯一区别，它在公平锁中的实现如下：

```java
public final boolean hasQueuedPredecessors() {
        
        Node t = tail; 
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

如上可以看到，在公平锁的实现中，只有在等待队列中有等待线程，并且该线程和请求获取锁的线程不相等，那么就返回false。

```readerShouldBlock()```在非公平锁中的实现：

```java
final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
```

如上在非公平锁中，只需要判断等待队列中的第一个线程是否是获取公平锁的线程，如果不是，那么就返回false。

* ```fullTryAcquireShared()```方法，在tryAcquireShared中如果尝试获取读锁失败，会再次调用fullTryAcquireShared()方法来获取锁。

```java
final int fullTryAcquireShared(Thread current) {
            
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    //当前持有的锁是独占锁，直接返回-1.
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                //从ThreadLocal中获取HoldCounter
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    //清空缓存
                                    readHolds.remove();
                            }
                        }
                       //返回-1,获取读锁失败
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```

线程尝试获取读锁失败后，会将线程加入到等待队列：

```java
//该方法详见"Java信号量详解"
private void doAcquireShared(int arg) {
     	//将线程保装成node节点，添加到队列的末尾
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {//只有请求获取锁线程是等待队列的第一个线程才会去尝试获取锁
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        //如果获取锁成功，重新设置头节点和唤醒下一个节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();//如果中断过，则继续自我中断
                        failed = false;
                        return;
                    }
                }
                //如果获取锁失败，则进入阻塞状态，等待前继节点唤醒
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```



写锁：

```java
public void lock() {
	sync.acquire(1);
}
public final void acquire(int arg) {
	if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
  }
```

* ```tryAcquire()```方法：首先获取读写锁的状态，如果读写锁的状态不为0 ，说明已经有线程成功获取到了锁，然后判断锁的类型，如果当前被持有的锁是读锁，则方法直接返回false，因为写锁与读锁互斥，如果锁的类型是写锁，则需要判断持有锁的线程是否与请求获取锁的线程相等，如果相等，根据可重入原则，直接修改写锁的状态，如果两个线程不相等，则直接返回false。如果读写锁的状态为0，说明还没有线程获取到锁，则根据公平原则来获取锁，如果获取锁成功，则将持有锁的线程设置为当前线程。

```java
protected final boolean tryAcquire(int acquires) {
            
            Thread current = Thread.currentThread();
            int c = getState();//锁的状态
            int w = exclusiveCount(c);//独占锁的状态（写锁）
            if (c != 0) {
                //如果写锁z等于0而读写锁的状态不为0，说明当前被持有的锁是读锁，返回false
                //如果请求线程与持有锁的线程不是同一个线程，返回false
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //重新设置锁的状态
                setState(c + acquires);
                return true;
            }
    		//线程是否需要等待其它线程（分公平锁与分公平锁）
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
    		//将持有锁的线程设置为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
```

```writerShouldBlock()```：判断是否需要等待线程，分为公平锁和非公平锁。

公平锁：如果在等待线程有其它线程在等待获取锁，并且该等下线程中的第一个线程与该请求获取线程不相等，则返回true。

```java
 final boolean writerShouldBlock() {
 	return hasQueuedPredecessors();
 }
 public final boolean hasQueuedPredecessors() {
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

非公平锁：直接返回false。

```java
final boolean writerShouldBlock() {
   return false; 
}
```

#### 释放锁

* 释放读锁

```java
 public void unlock() {
 	sync.releaseShared(1);
 }

public final boolean releaseShared(int arg) {
	if (tryReleaseShared(arg)) {//尝试释放共享锁
    	doReleaseShared();
    	return true;
 	}
    return false;
 }
```

```tryReleaseShared()```:尝试是否共享锁。首先修改缓存中的读锁的数量，如果读锁数量为1，则直接删除缓存否则将锁的数量减一。接着在修改读写锁中读锁的状态，如果读写锁中读锁的数量为0，说明读写锁中的读锁以及全部释放掉了，返回true。

```java
protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();//从ThreadLocal中获取HoldCounter
                int count = rh.count;
                if (count <= 1) {//如果读锁的数量<=1，直接删除锁缓存
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;//持有锁的数量减一
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                //设置读写锁中读锁的状态
                if (compareAndSetState(c, nextc))
                    //如果等于0，说明已经全部是否了
                    return nextc == 0;
            }
        }
```

如果锁全部释放完之后，如果等待队列中有需要唤醒的线程，那么就接着唤醒等待队列中阻塞的线程

```java
private void doReleaseShared() {
       
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                //如果头节点的状态为SIGNAL，说明有待唤醒的线程
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);//唤醒它的后续线程
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

unparkSuccess方法：

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
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);//唤醒线程
    }
```

* 释放写锁,参考ReetrantLock中的锁释放.

```java
 public void unlock() {
 	sync.release(1);
 }

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

