#### ThreadLocal源码分析

ThreadLocal是一个线程的局部变量，只有当前线程可以访问，因此是线程安全的。

示例：

```java
public class ThreadLocalDemo {

    private static ThreadLocal<SimpleDateFormat> threadLocal = new ThreadLocal<SimpleDateFormat>(){
        @Override
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        }
    };

    public static class ParseDate implements Runnable{

        int i = 0;

        public ParseDate(int i){this.i = i;}

        @Override
        public void run() {
            try {
                Date t = threadLocal.get().parse("2018-10-28 10:23:" + i % 60);
                System.out.println(i + ":" + t);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }


    public static void main(String[] args){

        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i =0; i<1000; i++){
            executorService.execute(new ParseDate(i));
        }

    }

}
```

源码分析：

* set方法

```java
public void set(T value) {
        Thread t = Thread.currentThread();//获取当前线程
        ThreadLocalMap map = getMap(t);//获取线程内的ThreadLocalMap对象，类似map集合
        if (map != null)
            map.set(this, value);//this指向调用set方法的ThreadLocal对象
        else
            createMap(t, value);//初始化线程t的ThreadLocalMap，key也是ThreadLocalMap
}

ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
 }

void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue)//this是ThreadLocal
 }
```

* ThreadLocalMap.set()方法：

```java
 private void set(ThreadLocal<?> key, Object value) {
	
			//table用来存放Entry的数组，它的索引就是ThreadLocal的哈希值
            Entry[] tab = table;
            int len = tab.length;
     		//计算哈希值
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];e != null;
             e = tab[i = nextIndex(i, len)]) { //通过nextIndex()方法往后查找空闲的位置
                ThreadLocal<?> k = e.get();

                if (k == key) {//如果key相等，说明是同一个ThreadLocal，直接覆盖原来的值
                    e.value = value;
                    return;
                }
				//替换掉失效的entry
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;//size表示tab数组中元素的数量
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();//扩容
        }
```

* replaceStaleEntry()方法

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

         	//向前扫描，查找最前面的一个无效slot
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)//获取key,如果key==null说明ThreadLocal已经被清理
                    slotToExpunge = i;

           //向后遍历table
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
				
               
                if (k == key) {//key在table中已经存在
                    e.value = value;//替换原来的值
					//tab[i]与tab[staleSlot]交换位置,如果之前的向前遍历没有发现无效的entry，那么可以断定i之前的entry都是有效的
                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;
					
                    //如果之前向前扫描没有发现无效的entry,那么slotToExpunge == staleSlot必然为true，然后我们可以继续从i处往后循环查找无效的entry.
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    //从i处继续清理无效的entry
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                //如果检测到有无效的entry，并且之前扫描没有发现无效的entry，更新slotToExpunge
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            //如果key在table中不存在，则直接在staleSlot处存放即可.
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

           //如果在检查过程中发现无效的entry，则做一次清理
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
```

* cleanSomeSlots()：清除ThreadLocalMap中无效的元素

```java
private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                //ThreadLocal为空,需要清空无效的entry
                if (e != null && e.get() == null) {
                    n = len;//
                    removed = true;
                    //清理一个连续的段，即直到下一个空的slot为止
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);//n主要是用来控制循环的次数。
            return removed;
        }
```

* expungeStaleEntry(int staleSlot)方法，分段清理无效的元素，并重新计算元素的哈希值。

```java
private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

           //清理staleSlot处无效的entry
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
    //从staleSlot往后循环,清理无效的entry
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;//直到下一个tab[i]为空，停止循环，返回i值
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    //计算哈希值
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {//如果哈希不相等，则用最新的哈希地址
                        tab[i] = null;

                        
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

* rehash()方法

````java
private void rehash() {
    		//做一次全量的清理
            expungeStaleEntries();
          
    		//因为做了一次全量清理，所以size可能会变小
    		//threadhold = len*2/3 
            if (size >= threshold - threshold / 4)//len/2
                resize();
  }
````

* resize()方法，扩大的容量为原来的2倍

```java
private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }
            setThreshold(newLen);
            size = count;
            table = newTab;
        }
```

哈希值的计算：```int i = key.threadLocalHashCode & (len-1);```，threadLocalHashCode是ThreadLocal中的一个成员变量:

```java
private final int threadLocalHashCode = nextHashCode();
private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
}

private static AtomicInteger nextHashCode = new AtomicInteger();
private static final int HASH_INCREMENT = 0x61c88647;

 	
```

上面的HASH_INCREMENT与斐波那契散列有关。斐波那契散列法本质上是一个乘法散列法，它采用一个特殊的乘数，然后与之相乘，再进行右移得出一个冲突比较少的值。

对于乘数的选择：

1. 对于16位整数而言，这个乘数是40503。
2. 对于32位整数而言，这个乘数是2654435769。
3. 对于64位整数而言，这个乘数是11400714819323198485。

例：H(key) = ( (k * 2654435769)  >> 28) << Y

再回到ThreadLocalMap中，```HASH_INCREMENT = (1L <<32)- 2654435769```，当我们将ThreadLocal为key，把值保存到ThreadLocal的ThreadLocalMap中时，ThreadLocalMap会为每个key设置一个累加的ID,然后将这个ID与ThreadLocalMap的容量进行取模，就会得到一个比较均衡的结果。

* get()方法

```java
public T get() {
        Thread t = Thread.currentThread();//获取当前线程
        ThreadLocalMap map = getMap(t);//获取Thread内部的ThreadLocalMap对象
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);//根据key获取value
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

* getEntry()方法

```java
 private Entry getEntry(ThreadLocal<?> key) {
     		//根据key的threadLocalHashCode来获取索引，即哈希值
            int i = key.threadLocalHashCode & (table.length - 1);
     		//根据下标从数组中获取entry对象
            Entry e = table[i];
     		//如果不为空，并且key相等，则返回值
            if (e != null && e.get() == key)
                return e;
            else
                //如果没有命中，则从i处开始循环查找
                return getEntryAfterMiss(key, i, e);
  }
```

* getEntryAfterMiss()方法

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)//ThreadLocal已经被回收
                	//需要清除掉无效的entry
                    expungeStaleEntry(i);
                else
                    //否则继续循环
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
}
```

* expungeStaleEntry()方法

```java
 private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
				
            //因为ThreadLocal已经为空，所以它对应的entry需要清除掉
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;//数量减一

            // Rehash until we encounter null
            Entry e;
            int i;
     		
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    //对于还有ThreadLocal没有被回收的情况，需要重新生成它的下标（哈希值）
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        //将原来地址下的元素设置为null,有助于于垃圾回收
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            //重新生成哈希值
                            h = nextIndex(h, len);
                        //根据索引设置值
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

* remove()方法：删除一个元素

```java
 private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);//计算哈希值
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {//如果key相等
                    e.clear();//显示断开弱连接
                    //进行分段清理
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```



#### ThreadLocal与内存泄漏

当ThreadLocal对象被回收后，我们存放的value不会被回收，因为它是存放在Thead对象的ThreadLocalMap中，这是一条强引用链，是可达的，value不会被回收掉。

因此如果我们在线程池中调用开启线程，一个线程的寿命很长，大对象长期不会被回收，会影响系统的运行效率与安全。如果线程不复用，用完即销毁了也不会有ThreadLocal引发内存泄露问题。

在ThreadLocal中使用remove方法可以显示的清除无用的数据，同时调用ThreadLocal的get和set方法有很高的概率会顺便清理掉无效的对象，断开value强引用，从而大对象被收集器回收掉。因此在实际运用中，我们最好手动调用remove方法将无效的对象清理掉。





