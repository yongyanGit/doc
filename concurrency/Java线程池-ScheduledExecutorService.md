#### Java线程池-ScheduledExecutorService

>  用来执行定时或周期性任务。

ScheduledExecutorService的主要方法如下：

* schedule(Runnable command, long delay, TimeUnit unit)方法，在给定的时间内，对任务进行一次调度。
* scheduleAtFixedRate(Runnable command,long initialDelay,  long period,  TimeUnit unit)：以上一个任务的开始时间为起点，之后的period时间，再调度下一次任务。并且任务执行时间超过调度时间，也不会有多个任务堆叠在一起。
* `public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay,                                      long delay, TimeUnit unit)：以上一个任务结束后，再经过delay时间进行任务调度。



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

* 任务调度

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


public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<V> t = decorateTask(callable,
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
}
```

