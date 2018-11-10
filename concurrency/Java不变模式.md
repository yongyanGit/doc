#### Java不变模式

多线程对同一个对象进行读写操作时，为了保证对象数据的一致性和正确性，有必要对对象进行同步。而同步操作对系统性能是有相当的损耗。为了能尽可能地去除这些同步操作提高并行程序的性能，可以使用一种不可变的对象，依靠对象的不可变性，确保在没有同步操作的多线程环境中依然始终保持内部状态的一致性和正确性

不变模式的核心思想是，一旦对象被创建，它的内部状态将永远不会发生改变，没有一个线程可以改变其内部状态和数据。同时内部状态也绝不会自行发生改变。

不变模式的主要使用场景满足条件：

* 当对象创建后，其内部状态和数据不再发生任何变化。
* 对象被共享，被多线程频繁访问。

在Java中为了确保对象被创建后不发生任何改变，需要注意如下4点：

* 去除setter方法以及所有修改自身属性的方法。
* 将所有的属性设置为私有，并用final标记，确保其不可修改。
* 确保没有子类可以重载修改它的行为。
* 有一个可以创建完整对象的构造函数。

```java
public final class Product {
    
    private final String no;
    
    private final String name;
    
    private final double price;
    
    public Product(String no,String name,double price){
        
        this.name = name;
        
        this.no = no;
        
        this.price = price;
    }

    public String getName() {
        return name;
    }

    public double getPrice() {
        return price;
    }

    public String getNo() {
        return no;
    }
}
```

