#### Java Future

Future 模式是多线程开发中常见的一种设计模式，它的核心思想是异步调用。当我们调用一个函数方法时，如果这个函数执行很慢，那么我们就要进行等待。但有时候我们可能并不着急要结果，可以让被调者立即返回，让它在后台慢慢处理这个请求，对于调用者来说，可以先处理一些其它的任务，在真正需要数据的场合再尝试去获取需要的数据。

例子：

```java
public class RealData implements Callable<String> {

    private String data;

    public RealData(String data){
        this.data = data;
    }

    @Override
    public String call() {
		//call方法执行具体的业务逻辑
        StringBuffer sb = new StringBuffer();
        for (int i = 0;i<10;i++){
            sb.append(data);
            try{
                Thread.sleep(100);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
        return sb.toString();
    }
}

public class FutureMain {

    public static void main(String[] args) throws Exception{
		//创建一个FutureTask对象，它的构造函数参数类型是Callable类型对象
        FutureTask<String> futureTask = new FutureTask<>(new RealData("a"));
		
        ExecutorService executorService = Executors.newFixedThreadPool(1);
		//执行任务
        executorService.submit(futureTask);
        
        try {
            //这里依然可以额外的数据操作
            Thread.sleep(2000);
        }catch (Exception e){

        }
		//调用get方法，获取业务执行结果
        System.out.println("数据"+futureTask.get());

    }

}
```



源码解析：

* Callable接口，Callable接口与Runnable类似，区别在于前者可以返回执行结果和抛出检查异常。

```java
public interface Callable<V> {//泛型V表示方法返回的类型
   
    V call() throws Exception;
}
```

* Future接口

```java
public interface Future<V> {

    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();
    
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

1. cancel()方法：如果异步任务已经完成或者已经被取消，或者由于某些原因不能取消，则返回false。如果任务还没被执行，则会返回true并且任务不会被执行。如果任务已经开始执行了，但是还没有执行完，如果参数mayInterruptIfRunning为true则会立即中断任务线程并返回true，否则会返回true，但是不会中断线程。
2. isCancelled()：判断在任务完成前，任务是否已经被取消了。
3. isDone()：判断任务是否已经完成，如果已经完成则返回true，否则返回false。任务执行过程中，发生异常、任务被取消也属于任务已经被完成，也会返回true。
4. get()：获取任务执行结果，如果任务还没有执行完成，线程进入阻塞。如果任务被取消，则抛出CancellationException异常，如果执行过程中发生异常，则抛出ExecutionException，如果阻塞过程中线程被中断，抛出InterruptedException异常。

* FutureTask

1. FutureTask构造方法

```java
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;//callable对象封装具体的业务供线程调用
        this.state = NEW;       // 任务状态，
  }
```

FurureTask的状态一共有1种：

```java
/**	状态变化过程
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```

* NEW ：表示FutureTask是一个新的任务或者还没有执行完的任务，这个是初始状态。
* COMPLETING：表示任务执行完成或者执行任务的时候发生异常，但是任务执行结果或者异常原因还没有保存到outcome字段，这个时候状态会从NEW变到COMPLETING。
* NORMAL：任务已经执行完成，并且执行结果已经保存到outcome字段，状态会从COMPLETING转换到NORMAL。
* EXCEPTIONAL：任务执行发生异常时并且异常原因保存到outcome字段后，状态会从COMPLETING转换到EXCEPTIONAL。
* CANCELLED：任务还没有开始执行或者已经开始执行了但是还没有执行完成，用户调用了cancle(false)方法取消任务并且不中断任务线程，这个时候状态就会从NEW变成CANCELLED。
* INTERRUPTING：任务还没有开始执行或者已经开始执行，但是还没有执行完的时候，用户调用了cancle(true)方法，状态从NEW变成INTERRUPTING。
* INTERRUPTED：调用interrupt()中断任务执行线程之后状态就会从INTERRUPTING转换成INTERRUPTED。

2. run方法：

```java
public void run() {
    //检查任务状态，如果它的状态不是NEW，表示该任务已经被取消
    //通过cas函数来设置当前线程
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();//执行业务逻辑
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);//发生异常，将异常保存到outcome字段上
                }
                if (ran)//正常执行完
                    set(result);//执行完成将结果保存到outcome字段上
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }


protected void set(V v) {
    	//将任务状态从NEW变成COMPLETING
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;//将结果保存到outcome对象上
            //将任务的状态从COMPLETING变成NORMAL
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
 }

 protected void setException(Throwable t) {
    //将当前任务的状态从NEW变成COMPLETING
  	if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;//将异常保存到outcome对象上
        	//将当前任务的状态从COMPLETING变成EXCEPTIONAL
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
     }
 }

```

3. get()方法：

```java
 public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)//如果任务还没有执行完成，线程进入阻塞状态
            s = awaitDone(false, 0L);
        return report(s);//返回结果
   }
```

awaitDone()方法

```java
 private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
     	//计算等待截止时间
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                //如果线程被中断，则从等待队列中移除节点并抛出异常
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                //状态大于COMPLETING，说明任务已经结束（正常结束、异常结束、被取消）
                if (q != null)
                    q.thread = null;//help gc
                return s;
            }
            else if (s == COMPLETING) 
                //表示任务已经完成，但是结果还没有赋值到oucome字段
                //让其它线程优先执行
                Thread.yield();
            else if (q == null)
                //创建一个node节点
                q = new WaitNode();
            else if (!queued)
                //通过cas函数将上一个循环创建的WaitNode节点放在waiters的首节点
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {//如果已经超时，清除等待队列中的节点，返回任务状态
                    removeWaiter(q);
                    return state;
                }
                //阻塞线程直到超时
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);//阻塞线程，直到被其它线程唤醒
        }
    }
```

report()方法：

```java
 private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;//如果正常结束，直接返回执行结果
        if (s >= CANCELLED)
            throw new CancellationException();//如果任务被取消，抛出异常
        throw new ExecutionException((Throwable)x);//抛出异常
    }
```

4. cancle(boolean)取消任务

```java
public boolean cancel(boolean mayInterruptIfRunning为false，将任务状态从NEW变成) {
    	//如果mayInterruptIfRunning为true，将任务状态从NEW 变成INTERRUPTING
    	//如果mayInterruptIfRunning为false，将任务状态从NEW变成CANCELLED
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();//给线程设置中断标志
                } finally { 
                    //修改状态为INTERRUPTED
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```

finishCompletion方法：循环唤醒阻塞中的线程

```java
 private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);//唤醒线程
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```

