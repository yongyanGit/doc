#### ArrayBlockingQueue源码解析

ArrayBlockingQueue是基于数组实现的有界队列，队列可以容纳的最大元素数量需要在创建队列时指定。BlockingQueue适合作为数据共享的通道，当队列为空时，它可以让服务线程进行等待，当有消息进入队列后，自动唤醒线程。

* 构造方法：初始化ArrayBlockingQueue时，会从构造方法中传入一个capacity参数，它用来指定数组的初始化大小，即容器大小。在容器中各个线程获取锁的策略默认是非公平的，可以通过构造函数来指定策略。

```java
public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
}

public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
    	//根据构造方法传入进来的容量大小初始化数组
        this.items = new Object[capacity];
    	//初始化独占锁ReentrantLock,锁的性质默认是非公平的
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
 }
```

* take()方法，从队列中获取元素，如果队列为空，则线程进入阻塞状态。

```java
 public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();//获取锁，可中断
        try {
            while (count == 0)//如果队列为空，则进入阻塞状态
                notEmpty.await();
            return dequeue();//出队列
        } finally {
            lock.unlock();
        }
    }

private E dequeue() {
        //存放数据的数组
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];//通过下标从数组中获取元素
        items[takeIndex] = null;//将takeIndex处置为null
        if (++takeIndex == items.length)//数组中的元素已经全部取完
            takeIndex = 0;
        count--;//数组中元素的数量减一
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();//唤醒生产消息的线程，
        return x;
    }
```

* poll()：从队列中获取元素，如果队列为空，线程不会进入阻塞状态，而是直接返回null。

```java
public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //线程为空直接返回null，否则调用dequeue出队列。
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
```

* put()方法：

```java
public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();//获取锁，可中断
        try {
            while (count == items.length)
                notFull.await();//如果队列已经满了，则线程进入阻塞状态
            enqueue(e);//往队列中添加元素
        } finally {
            lock.unlock();
        }
    }

 private void enqueue(E x) {
       //存放元素的数组
        final Object[] items = this.items;
     	//将元素存放到putIndex下标处
        items[putIndex] = x;
        if (++putIndex == items.length)//true？队列已经添加满
            putIndex = 0;
        count++;//记录队列中的元素数量
        notEmpty.signal();//唤醒消费元素的线程
    }
```

