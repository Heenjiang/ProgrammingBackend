# Java中的CAS机制

## **简述**

CAS机制其实是基于汇编语言的指令提供的，全程是Compare And Swap

> You can use the Machine Interface's (MI) Compare and Swap (CMPSWP) instruction to access data in a multithreaded program.
>
> The CMPSWP instruction compares the value of a first compare operand to the value of a second compare operand. If the two values are equal, the swap operand is stored in the location of the second compare operand. If the two values are unequal, the second compare operand is stored into the location of the first compare operand.
>
> When an equal comparison occurs, the CMPSWP instruction assures that **no access by another CMPSWP instruction will occur at the location of the second compare operand** between the moment that the second compare operand is fetched for comparison and the moment that the swap operand is stored at the location of the second compare operand.
>
> When an unequal comparison occurs, no atomicity guarantees are made regarding the store to the first compare operand location and other CMPSWP instruction access. Thus only the second compare operand should be a variable shared for concurrent processing control.

上面是引用自https://www.ibm.com/docs/en/i/7.1?topic=threads-compare-swap-instruction，描述了IBM平台对CAS的支持，该命令是在多线程并发时，使用**非锁的方式**对共享数据进行访问的方式。从上面的描述，CMPSWP命令一共有三个操作数：（**first compare operand, second compare operand, swap operand）**。在执行该命令时，如果first compare operand == second compare operand, 那么就将swap operand存入second compare operand的内存位置；否则，就存入first compare operand的位置。

而且如果是相等的情况下，CMPSWP命令保证在读取second compare operand和将swap operand存入second compare operand内存位置的过程中，不会有其他CMPSWP命令对second compare operand位置有访问权利（保证相等情况时，swap操作是原子性的）

而当等式不成立时，swap operand 存入first compare operand的过程的原子性不被保证（其他CMPSWP命令依然可以访问first compare operand的内存位置），所以**应该将second compare operand作为并发时的共享变量**。

上面的只是IBM对CAS的实现，其他平台的实现虽然有区别，也大致相似。

下面内容取自；https://blog.csdn.net/qq_32998153/article/details/79529704?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_default&utm_relevant_index=1

在Java层面可以，换一个通俗易懂的说法就是，**一个汇编命令，有3个操作数：内存地址V，旧的预期值A，要修改的新值B。当操作执行时，比较V的值和A是否相等，相等就将B放进V，如果不相等就自旋。**（线程1重新获取内存地址V的当前值，并重新计算想要修改的值。这个重新尝试的过程被称为自旋。）

我们来看CAS在java中的运用

```java
public class CounterWithSynchronized {
	 private int count = 0;

	    public void add(int n) {
	        synchronized(this) {
	            count += n;
	        }
	    }

	    public void dec(int n) {
	        synchronized(this) {
	            count -= n;
	        }
	    }

	    public int get() {
	        return count;
	    }
}

```

上面的代码我们是通过synchronized来保证count变量在并发时能保证自增和自减的原子性的。但是这样往往会带来性能问题（Java在1.6后对synchronized已经进行了优化），所以java在JUC（java.until.cocurrent）包中提供了一些并发环境下，能够保证原子性，可见性的类。然后上面的代码就可以修改成：

```java
public class CounterCAS {
    private AtomicInteger count = new AtomicInteger(0);

    public void add(int n) {
        count.addAndGet(n);
    }

    public void dec(int n) {
        count.addAndGet(-n);
    }

    public int get() {
        return count.get();
    }
}}
```

## Java中AtomicInteger类的底层CAS实现

```java
  public final int getAndAdd(int delta) {
        return U.getAndAddInt(this, VALUE, delta);
    }

    /**
     * Atomically increments the current value,
     * with memory effects as specified by {@link VarHandle#getAndAdd}.
     *
     * <p>Equivalent to {@code addAndGet(1)}.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        return U.getAndAddInt(this, VALUE, 1) + 1;
    }
```

可以看到是通过U.getAndAddInt()方法实现的

```java
private static final Unsafe U = Unsafe.getUnsafe();
```

而这个U就是Unsafe类的一个实例，它是JVM中调用底层操作系统的接口（利用C++和C）

```java
    /**
     * Atomically adds the given value to the current value of a field
     * or array element within the given object {@code o}
     * at the given {@code offset}.
     *
     * @param o object/array to update the field/element in
     * @param offset field/element offset
     * @param delta the value to add
     * @return the previous value
     * @since 1.8
     */
    @HotSpotIntrinsicCandidate
    public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!weakCompareAndSetInt(o, offset, v, v + delta));
        return v;
    }
```

这是Unsafe类中getAndAddInt的实现，可以看到我们是通过循环（自旋方式）来改变变量的，如果weakCompareAndSetInt()返回true，那么就结束循环，**返回原来的v。** getIntVolatile方法负责获取到变量的最新值（Volatile方式）。

```java
	@HotSpotIntrinsicCandidate
    public final boolean weakCompareAndSetInt(Object o, long offset,
                                              int expected,
                                              int x) {
        return compareAndSetInt(o, offset, expected, x);
    }
```

weakCompareAndSetInt()方法中就简单调用了compareAndSetInt()

```java
 	/**
     * Atomically updates Java variable to {@code x} if it is currently
     * holding {@code expected}.
     *
     * <p>This operation has memory semantics of a {@code volatile} read
     * and write.  Corresponds to C11 atomic_compare_exchange_strong.
     *
     * @return {@code true} if successful
     */
    @HotSpotIntrinsicCandidate
    public final native boolean compareAndSetInt(Object o, long offset,
                                                 int expected,
                                                 int x);

```

注释已经说得很清楚了，原子性的将java变量update成x，如果满足expected。等价于c++的汇编底层调用atomic_compare_exchange_strong（就是用C++实现的）.成功就返回true

## **CAS的缺点**

1） CPU开销过大

在并发量比较高的情况下，如果许多线程反复尝试更新某一个变量，却又一直更新不成功，循环往复，会给CPU带来很到的压力。

2） 不能保证代码块的原子性

CAS机制所保证的知识一个变量的原子性操作，而不能保证整个代码块的原子性。

3） ABA问题

CAS需要检查操作值有没有发生改变，如果没有发生改变则更新。但是存在这样一种情况：如果一个值原来是A，变成了B，然后又变成了A，那么在CAS检查的时候会发现没有改变，但是实质上它已经发生了改变，只是又回到了原来的值而已，这就是所谓的ABA问题。

具体例子看：https://blog.csdn.net/qq_42370146/article/details/105559575#:~:text=CAS%20%EF%BC%88compareAndSwap%EF%BC%89%EF%BC%8C%E4%B8%AD%E6%96%87%E5%8F%AB,%E7%9A%84%EF%BC%8C%E4%BB%8E%E8%80%8C%E5%9C%A8%E7%A1%AC%E4%BB%B6%E5%B1%82%E9%9D%A2

对于ABA问题其解决方案是加上版本号，即在每个变量都加上一个版本号，每次改变时加1，即A —> B —> A，变成1A —> 2B —> 3A，采用AtomicStampedRdference类可以实现这个方案。