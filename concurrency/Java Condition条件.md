#### Java Condition条件

Condition的作用与Object.wait()和Object.notify()方法大致相同，都是让线程进入阻塞以及唤醒休眠中的线程，但是wait()和notify()方法是synchronized关键字配合使用的，而Condition是和重入锁相关联(如：ReetrantLock)，通过newCondition()方法可以生成一个与当前重入锁绑定的Condition实例。利用Condition对象，我们就可以让线程在合适的时间等待，或者在某一个特定的时刻得到通知。

Condition接口提供的基本方法

```java
void await();
void awaitUninterruptibly();
long awaitNanos(long nanosTimeout);
boolean await(long time,TimeUnit unit);
boolean awitUntil(Date deadline);
void signal();
void signalAll();
```

* await()方法会使当前线程等待，同时释放当前锁，当其他线程中使用signal()或者signalAll()方法时，线程会重新获得锁并继续执行。当线程被中断时，也能跳出等待。
* awaitUninterruptibly()方法与await()方法基本相同，但是它不会在等待的过程中响应中断。
* singal()方法用于唤醒一个在等待中的线程。相对的singalAll()方法会唤醒所有在等待中的线程。

一个简单的Condition例子：

```java
public class ReenterLockCondition implements Runnable {

    public static ReentrantLock lock = new ReentrantLock();
    public static Condition condition = lock.newCondition();

    @Override
    public void run() {
        try {
            lock.lock();//获取锁
            condition.await();//进入阻塞
            System.out.println("Thread is going on");
        }catch (InterruptedException e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }


    public static void main(String[] args)throws InterruptedException{

        ReenterLockCondition reenterLockCondition = new ReenterLockCondition();

        Thread thread = new Thread(reenterLockCondition);

        thread.start();
        Thread.sleep(2000);
        lock.lock();//获取锁，
        condition.signal();//唤醒等待中的线程
        lock.unlock();//释放锁，以便其它被唤醒的线程能够获取到锁

    }
}
```

#### 源码解析

当```await()```方法被调用时，代码如下：

```java
public final void await() throws InterruptedException {
	if (Thread.interrupted())//如果线程被中断过，直接抛出中断异常
        throw new InterruptedException();
    //将当前线程保装成一个node对象,并加入到Condition阻塞队列
    Node node = addConditionWaiter();
    long savedState = fullyRelease(node);//释放当前线程占有的锁
    int interruptMode = 0;
    
    //释放完锁后，遍历AQS队列，看当前节点释放在队列中
    while (!isOnSyncQueue(node)) {
        //不在，说明它还没有竞争锁的资格，所以继续进入阻塞状态
           LockSupport.park(this);
           if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
               break;
    }
    //被唤醒后，重新开始竞争锁，如果竞争失败，会重新进入沉睡状态
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
     if (interruptMode != 0)
         reportInterruptAfterWait(interruptMode);
      }
```

* ```addConditionWaiter```方法：将当前线程封装成一个Node对象，并且将该节点添加到condition等待队列的末尾.

```java
 private Node addConditionWaiter() {
 	Node t = lastWaiter;//最后一个节点
    // If lastWaiter is cancelled, clean out.
     //如果最后一个节点的状态不是CONDITION，将它清出Condition等待队列
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
     }
     //新建一个node节点对象，将它的状态设置为Condition,
     Node node = new Node(Thread.currentThread(), Node.CONDITION);
     if (t == null)
          firstWaiter = node;
      else
          t.nextWaiter = node;//将它存放在链表的末尾
      lastWaiter = node;
      return node;
     }
```

* ```unlinkCancelledWaiters()```方法，将状态不是```CONDITION```的线程从condition等待队列清除掉。

```java
private void unlinkCancelledWaiters() {
	Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        //从第一个节点开始循环，如过节点状态不是Condition,清除出阻塞队列
           Node next = t.nextWaiter;
           if (t.waitStatus != Node.CONDITION) {
               t.nextWaiter = null;
               if (trail == null)
                   //trail为null。说明是第一次循环
                   firstWaiter = next;
               else
                   trail.nextWaiter = next;
                if (next == null)
                    lastWaiter = trail;
            }
            else
                trail = t;
            t = next;
      }
  }
```

* ```fullyRelease()```释放锁

```java
final long fullyRelease(Node node) {
	boolean failed = true;
    try {
    	long savedState = getState();//获取锁的状态
        if (release(savedState)) {//完全释放锁
            failed = false;
                return savedState;
         } else {
           throw new IllegalMonitorStateException();
      	 }
      } finally {
        if (failed)
           node.waitStatus = Node.CANCELLED;
       }
    }
```

* ```signal()```通知：将condition队列中的第一个线程移到同步等待队列中。

```jAVA
public final void signal() {
	if (!isHeldExclusively())//如果不是当前线程持有锁，抛出异常
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}


 private void doSignal(Node first) {
 	do {
     	if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
     } while (!transferForSignal(first) & (first = firstWaiter) != null);
 }

final boolean transferForSignal(Node node) {
    //将待唤醒的线程的状态设置为初始状态    
	if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
	//将待唤醒的线程添加到同步等待队列，并返回它的前置节点
    Node p = enq(node);
    int ws = p.waitStatus;
    //如果前置节点的状态为取消或它的节点的状态设置为SIGNAL失败，直接唤醒待唤醒的节点；
    //如果前置节点状态设置成功，则当前节点等待它的前置节点来唤醒它。
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
        return true;
    }
```

