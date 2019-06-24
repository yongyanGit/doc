#### ConcurrentLinkedQueue源码解析

ConcurrentLinkedQueue是一个基于链表的无界安全队列，采用FIFO的规则对节点进行排序，当我们添加一个元素的时候，会追加在链表的末尾，当我们获取一个元素时，它会返回队列头部的元素。

作为一个链表，自然需要定义有关链表内的节点，在ConcurrentLinkedQueue中，定义的节点Node核心如下：

```java
static final class Node<E> {
       volatile E item;
       volatile Node<E> next;
       }      
```

其中item存放的是目标元素，next表示当前节点的下一个节点，这样每个node就能环环相扣，串在一起了。

* 构造方法，在构造方法中，将链表的头节点和尾节点初始化成一个没有元素的空节点。

```java
public ConcurrentLinkedQueue() {
        head = tail = new Node<E>(null);
    }
```

* add()方法，在链表的末尾添加一个节点。

```java
public boolean add(E e) {
	return offer(e);
}
public boolean offer(E e) {
        checkNotNull(e);
    	//新建一个node节点
        final Node<E> newNode = new Node<E>(e);
		
        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            if (q == null) {
                //p是最后一个节点
                //通过cas设置p的next为newNode
                if (p.casNext(null, newNode)) {
                    //tail的更新会滞后，每次更新会跳跃2个节点
                    if (p != t) //假设第一次添加元素，p 是等于t的，不会执行casTail方法
                        casTail(t, newNode);  
                    return true;
                }
               
            }
            else if (p == q)
                //2 从头开始循环
                p = (t != (t = tail)) ? t : head;
            else
               //1
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
```

offer方法没有任何锁，线程安全完全由CAS操作和队列的算法来保证。整个方法的核心是for循环，这个循环没有出口，直到尝试成功。

当第一次加入元素时，由于队列为空，p.next为空，程序将P的next设置为newNode，因为是第一次添加，所以此时p 等于t，因此不会更新tail尾节点。

当第二次添加元素时，tail的next已经不为空了，它指向的是队列中的第一个元素，所以会进入代码“1”处然后执行下面的代码：```p = (p != t && t != (t = tail)) ? t : q;```,执行"!="时，程序会先取得t的值，然后执行t=tail，并取得新的t的值，在单线程中，t!=t显然不会成立，但是在并发环境中，有可能先获得左边的t值，然后右边的t值被其它线程修改，这样t!=t就可能成立。因此在比较的过程中，tail被其它线程修改，当它再次赋值给t时，就会导致等待左边的t和右边的t不同。如果两个t不相同，表示tail在中途被其它线程修改过，这时我们就可以用新的tail作为链表的末尾，也就是等式右边的t。

至于代码“2”处的 ```p ==q```，主要表示要删除的节点或者节点为空。

* poll方法，获取队列的第一个元素，并删除它。

```java
public E poll() {
        restartFromHead:
        for (;;) {
            for (Node<E> h = head, p = h, q;;) {
                E item = p.item;
				//将头节点持有的元素置为空
                if (item != null && p.casItem(item, null)) {//2
                    
                    if (p != h) 
                        updateHead(h, ((q = p.next) != null) ? q : p);
                    return item;
                }
                
                else if ((q = p.next) == null) {//将p的next赋值给q
                    updateHead(h, p);
                    return null;
                }
                
                else if (p == q)
                    continue restartFromHead;
                else
                    p = q;//1
            }
        }
    }
```

假设队列只添加了一个元素，根据前面的描述，此时tail并没有更新，而是指向和head相同的位置，此时head本身的item为null,它的next为列表第一个元素。因此在第一次循环时，代码直接计入1处，将q的值赋值给p,也就是列表中的第一个元素。接着在第二次循环中，p.item显然不为NULL，因此代码会进入代码2处，如果cas操作成功，则会将p的item的值设置为null。同时，此时p与h是不相等的，故执行updateHead方法：

```java
final void updateHead(Node<E> h, Node<E> p) {
        if (h != p && casHead(h, p))//将p设置为头节点
            h.lazySetNext(h);//原来的头节点的next指向它自己
}
```

lazySetNext()方法操作成功后，会将原来的头节点的next指向自己，而此时head与tail实际上是同一个元素，所以offer插入元素是，就会遇到这个tail，也就是offer代码中判断p==q的意义。