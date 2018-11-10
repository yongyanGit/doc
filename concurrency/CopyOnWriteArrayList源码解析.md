#### CopyOnWriteArrayList源码解析

从类的名字可以看到，所谓CopyOnWrite就是在写入操作时,进行一次自我复制。即当这个List需要修改时，我们并不修改原有的内容，而是对原有的数据进行一次自我复制，将修改的内容写入到副本中。写完之后，再将修改完的副本替换原来的数据。这样就可以保证写操作不会影响读了。

* 构造方法，从构造方法中可以看到CopyOnWriteArrayList的数据存放在一个数组中。

```java
private transient volatile Object[] array;

public CopyOnWriteArrayList() {
        setArray(new Object[0]);
 }
 final void setArray(Object[] a) {
        array = a;
 }

```

* add()，添加一个元素

```java
 public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();//获取锁
        try {
            Object[] elements = getArray();//获取数组
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);//拷贝一份数组
            newElements[len] = e;//在数组的末尾添加一个元素
            //将拷贝的数组替换原来的数组，array被volatile修饰，一旦数组被替换，其它线程能够立刻知道
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();//释放锁
        }
    }
```

* get(index)，获取元素。

```java
public E get(int index) {
        return get(getArray(), index);
}
private E get(Object[] a, int index) {
        return (E) a[index];//直接根据下标，从数组中取值
}
```

* remove(index)，根据下标，删除数据

```java
public E remove(int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();//获取锁
        try {
            Object[] elements = getArray();//获取数组
            int len = elements.length;//数组的长度
            E oldValue = get(elements, index);//缓存被删除的元素的值，用来做返回值
            int numMoved = len - index - 1;
            if (numMoved == 0)//如果numMoved等于0，说明被删元素位于数组最后一位
                setArray(Arrays.copyOf(elements, len - 1));//直接拷贝数组最后一位之前的数据
            else {
                Object[] newElements = new Object[len - 1];
                //拷贝被删元素之前的的元素
                System.arraycopy(elements, 0, newElements, 0, index);
                //拷贝被删元素之后的元素
                System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
                setArray(newElements);//替换数组
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
```

