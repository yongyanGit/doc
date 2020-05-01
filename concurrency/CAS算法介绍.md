#### CAS算法介绍

CAS(V,E,N)函数包含三个参数：V表示内存地址，E表示旧的预期值，N表示新值。仅当V地址处的值等于E时，才会将V地址处的值设置为N。CAS操作是报着乐观的态度进行的，它总是认为自己可以成功完成操作(乐观锁，相反的sync是被关锁)。当多个操作同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败。失败的线程不会被挂起，尽被告知失败，并且允许再次尝试。

与锁相比，使用CAS会使程序看起来更加复杂一些，但由于其非阻塞的，它对死锁问题天生免疫，并且，线程间的相互影响也非常小。更为重要的是，使用无锁的方式完全没有锁竞争带来的系统开销，也没有线程间频繁调度带来的开销，因此，他要比基于锁的方式拥有更优越的性能。

我们以AtomicInteger的源码为例子，来看看CAS函数的原理是怎么样的：

```java
AtomicInteger atomicInteger = new AtomicInteger();
atomicInteger.getAndIncrement();
//源码
public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
//源码
public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
//源码
 public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

从上往下看，可以看到最终代码调用的是一个本地方法，即它的实现是有c/c++来实现：

compareAndSwapInt()方法在```openjdk/hotspot/src/share/vm/prims/```目录下的unsafe.cpp类中实现：

```c++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
```

接着这个方法中调用cmpxchg方法，我们可以找到在linux_x86中的实现在```openjdk/hotspot/src/os_cpu/linux_x86/vm/atomic_linux_x86.inline.cpp```类中：

```c++
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```

mp是```os::is_MP()```的返回结果，```os::is_MP()```是一个内联函数，用来判断当前系统是否为多处理器。

1. 如果当前系统是多处理器，该函数返回1。
2. 否则，返回0

LOCK_IF_MP(mp)会根据mp的值来决定是否为cmpxchg指令添加lock前缀。

1. 如果通过mp判断当前系统是多处理器（即mp值为1），则为cmpxchg指令添加lock前缀。
2. 否则，不加lock前缀。

这是一种优化手段，认为单处理器的环境没有必要添加lock前缀，只有在多核情况下才会添加lock前缀，因为lock会导致性能下降。cmpxchg是汇编指令，作用是比较并交换操作数。

**intel手册对lock前缀的说明如下：**

1. 确保对内存的读-改-写操作原子执行。在Pentium及Pentium之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存。很显然，这会带来昂贵的开销。从Pentium 4，Intel Xeon及P6处理器开始，intel在原有总线锁的基础上做了一个很有意义的优化：如果要访问的内存区域（area of memory）在lock前缀指令执行期间已经在处理器内部的缓存中被锁定（即包含该内存区域的缓存行当前处于独占或以修改状态），并且该内存区域被完全包含在单个缓存行（cache line）中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（cache locking），缓存锁定将大大降低lock前缀指令的执行开销，但是当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。
2. 禁止该指令与之前和之后的读和写指令重排序（有序性）。
3. 把写缓冲区中的所有数据刷新到内存中（可见性）。

上面的第1点保证了CAS操作是一个原子操作，第2点和第3点所具有的内存屏障效果，保证了CAS同时具有volatile读和volatile写的内存语义。

#### 缺点

1. 循环时间开销很大。
2. ABA问题。即在A的基础上修改成B然后再修改成A，整个过程，cas函数无法感知。ava并发包为了解决这个问题，提供了一个带有标记的原子引用类“AtomicStampedReference”，它可以通过控制变量值的版本来保证CAS的正确性。