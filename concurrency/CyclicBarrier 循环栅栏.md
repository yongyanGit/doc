#### CyclicBarrier 循环栅栏

CyclicBarrier是一种多线程并发控制工具并且和CountDownLatch非常类似，它让一组线程到达一个屏障时被阻塞，直到最后一个线程到达屏障时，屏障才会打开，所有被拦截的线程继续运行。当凑齐一批线程后，计数器会变成0，然后接着凑齐下一批线程。

CyclicBarrier可以接收一个Runnable对象，在一组线程中的最后一个线程到达后（在唤醒所有线程之前），会执行该对象的run方法，并且该方法只执行一次。

示例：

```java
public class CyclicBarrierDemo {


    public static class Soldier implements Runnable{

        private String soldier;
        private final CyclicBarrier cyclicBarrier;

        Soldier(CyclicBarrier cyclicBarrier,String soldierName){
            this.cyclicBarrier = cyclicBarrier;
            this.soldier = soldierName;
        }

        @Override
        public void run() {
            try {
                cyclicBarrier.await();
                doWork();
                cyclicBarrier.await();
            }catch (Exception e){
                e.printStackTrace();
            }
        }


        void doWork(){
            try {
                Thread.sleep(Math.abs(new Random().nextInt() % 10000));
            }catch (Exception e){
                e.printStackTrace();
            }
            System.out.println(soldier+"：任务完成");
        }

    }


    public static class BarrierRun implements Runnable{


        boolean flag;
        int N;

        public BarrierRun(boolean flag,int N){
            this.flag = flag;
            this.N = N;
        }

        @Override
        public void run() {

            if (flag){
                System.out.println("司令：【士兵"+N+"个任务完成]");
            }else {
                System.out.println("司令：士兵【"+N+"个，集合完毕]");
            }

        }
    }


    public static void main(String[] args){


        final int N = 10;

        Thread[] allSoldier = new Thread[10];
        boolean flag = false;
        CyclicBarrier cyclicBarrier = new CyclicBarrier(N,new BarrierRun(flag,N));
        System.out.println("集合....");
        for(int i = 0;i < N; i++){
            System.out.println("士兵"+i+"报道");
            allSoldier[i] = new Thread(new Soldier(cyclicBarrier,"士兵"+i));
            allSoldier[i].start();
        }

    }

    
}
```

##### 源码分析

* 构造方法，在构造CyclicBarrier对象时，会传入一个int型数据parties，它表示每次需要凑齐多少个线程，CyclicBarrier才会打开屏障。另外一个barrierAction参数就是在最后一个线程到达后，会执行barrierAction的run方法。

```java
 public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```

* await方法

```java
 public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }


private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();//获取独占锁
        try {
            //Generation中只有一个boolean对象，用来表示栅栏是否被破坏，默认false
            final Generation g = generation;
			//如果栅栏被破坏，抛出异常，所有线程都不执行
            if (g.broken)
                throw new BrokenBarrierException();
			//如果有线程被中断了，则将Generation的状态设置为false,然后唤醒所有的阻塞线程
            //被唤醒的线程获取到broken的状态为false，也抛出异常
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            if (index == 0) {  // tripped
                //最后一个线程到达
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();//执行barrierAction的run方法
                    ranAction = true;
                    nextGeneration();//准备下一个循环
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await();//进入阻塞
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        Thread.currentThread().interrupt();
                    }
                }
				
                //如果栅栏被破坏，抛出异常
                if (g.broken)
                    throw new BrokenBarrierException();
				//说明一个循环已经结束
                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```

* breakBarrier方法，将generation.broken的状态设置为false。循环栅栏里面的线程要么全部执行要么全部不执行。

```java
private void breakBarrier() {
	generation.broken = true;
    count = parties;
    trip.signalAll();
}
```

* nextGeneration()，唤醒所有阻塞的线程，并且开始下一个循环。

```java
 private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
```



