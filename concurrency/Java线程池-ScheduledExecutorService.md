#### Java线程池-ScheduledExecutorService

>  用来执行定时或周期性任务。

ScheduledExecutorService的主要方法如下：

* schedule(Runnable command, long delay, TimeUnit unit)方法，在给定的时间内，对任务进行一次调度。
* scheduleAtFixedRate(Runnable command,long initialDelay,  long period,  TimeUnit unit)：以上一个任务的开始时间为起点，之后的period时间，再调度下一次任务。并且任务执行时间超过调度时间，也不会有多个任务堆叠在一起。
* public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay,                                      long delay, TimeUnit unit)：以上一个任务结束后，再经过delay时间进行任务调度。



简单示例：

```java
public class ScheduledExecutorServiceDemo {

    public static void main(String[] args){

        ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(10);
        scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                    System.out.println(System.currentTimeMillis()/1000);
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        },0,2, TimeUnit.SECONDS);

    }

}
```

#### 源码解析

* 构造方法，在构造方法中将maximumPoolSize设置为Integer.MAX_VALUE，所以corePoolSize几乎永远小于maximumPoolSize，因此核心线程（corePoolSize）将永远存在于线程池中，即使线程是空闲的，也不会被清除掉（除非设置allowCoreThreadTimeOut）。

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
}
```

* 任务调度schedule(Runnable command,long delay,TimeUnit unit)方法，

```java
    
 public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
  }
```

在ScheduledFuture中，将Runnable封装成了一个ScheduledFutureTask对象。在ScheduledFutureTask中实现了例如：比较两个任务的执行顺序、获取任务的执行周期等逻辑。

* delayedExecute()方法

```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
        if (isShutdown())//如果连接池被已经关闭，拒绝执行任务
            reject(task);
        else {
            super.getQueue().add(task);//否则将任务添加到DelayedWorkQueue队列中
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                //启动一个线程(不一定能启动)，确保任务能被执行
                ensurePrestart();
        }
    }
```

* add()方法，将任务添加进DelayedWorkQueue队列中

```java
public boolean add(Runnable e) {
            return offer(e);
}
public boolean offer(Runnable x) {
	if (x == null)
        throw new NullPointerException();
     RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
     final ReentrantLock lock = this.lock;
     lock.lock();//加锁
     try {
     	int i = size;
        if (i >= queue.length)//如果当前元素数量大于等于队列的容量，进行扩容,增加原来大小的一半
            grow();
        size = i + 1;//队列中的任务加1
         //如果当前队列还没有元素，则直接加入到头部
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0);
         } else {
            siftUp(i, e);//对任务进行排序
         }
         //如果当前任务是队列的第一个任务
         if (queue[0] == e) {
             leader = null;
             available.signal();//唤醒线程
         }
     } finally {
         lock.unlock();
     }
        return true;
 }
```

* siftUp()方法，对任务进行排序，按照任务的触发时间进行排序，把最早执行的任务排放在前面

```java
 private void siftUp(int k, RunnableScheduledFuture<?> key) {
 while (k > 0) {
 	int parent = (k - 1) >>> 1;//取中间值进行排序
    RunnableScheduledFuture<?> e = queue[parent];
    //compareTo方法，该方法在ScheduledFutureTask中进行了重写
    if (key.compareTo(e) >= 0)
        break;//如果当前任务的执行时间大于被比较的任务，则停止比较
     //否则位置对换
       queue[k] = e;
       setIndex(e, k);
       k = parent;
     }
     queue[k] = key;
     setIndex(key, k);
 }
```

* compareTo方法

```java
 public int compareTo(Delayed other) {
 	if (other == this) //如果是同一个任务，返回0
        return 0;
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        long diff = time - x.time;
        if (diff < 0)
            return -1;//触发时间小于比较对象
         else if (diff > 0)
            return 1;//触发时间大于比较对象
        //如果触发时间一样，则比较加入队列的顺序
         else if (sequenceNumber < x.sequenceNumber)
            return -1;
         else
            return 1;
     }
     long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
     return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
 }
```

* ensurePrestart()，启动一个线程，如果当前现场数已经小于corePoolSize才会启动新的线程

```java
void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
 }
```

* take()方法，任务被添加进队列后，线程池中的线程会调用队列的take()方法来获取任务，然后执行它

```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    //取出队列中的第一个元素，即最早需要执行的任务
                    RunnableScheduledFuture<?> first = queue[0];
                    //如果队列中没有任务，则线程进入阻塞，等待任务加入时唤醒
                    if (first == null)
                        available.await();
                    else {
                        //计算任务执行时间，任务触发时间-当前时间
                        long delay = first.getDelay(NANOSECONDS);
                        if (delay <= 0)
                            //到了触发时间，执行出队列操作,返回任务
                            return finishPoll(first);
                        first = null; // don't retain ref while waiting
                        //leader不等于空，表示任务已经被其它线程执行了，当前线程继续阻塞
                        if (leader != null)
                            available.await();
                        else {
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread;
                            try {
                                //如果任务还未到执行时间，线程继续阻塞相应的时间
                                available.awaitNanos(delay);
                            } finally {
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
                //队列中还有其它的任务等待线程去执行，唤醒阻塞中的线程
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }
```

* run方法，将任务从队列中取出来之后，执行ScheduledFutureTask中的run方法

```java
public void run() {
	boolean periodic = isPeriodic();//判断是否是周期性任务，1是0否
    if (!canRunInCurrentRunState(periodic))
         cancel(false);
     else if (!periodic)
         //如果不是周期性任务，直接执行run方法，然后结束任务
         ScheduledFutureTask.super.run();
     else if (ScheduledFutureTask.super.runAndReset()) {
         //计算下一次任务的执行时间
         setNextRunTime();
         //重新将任务添加进队列
         reExecutePeriodic(outerTask);
     }
 }
```

*  setNextRunTime()，重新计算任务的执行时间。

```java
private void setNextRunTime() {
	long p = period;
    if (p > 0)
        //如果任务周期大于0，则执行周期为：time+n*p,n为执行次数
        time += p;
     else
         //小于0，执行周期为：当前时间+delay
       	time = triggerTime(-p);
}
long triggerTime(long delay) {
        return now() +
            ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
 }

```

scheduleAtFixedRate和scheduleWithFixedDelay的区别：它们的区别在于period，scheduleAtFixedRate传入的是一个正数period，所以它的执行周期是：initialDelay+n*priod(n为执行次数)；与之相反，scheduleWithFixedDelay传入的是一个负数period，所以它的执行周期是：当前时间+period。

```java
 public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
    
     public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
```

