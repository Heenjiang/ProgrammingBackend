# JMM(Java Memory Model Aka Java内存模型)

在java的并发编程（concurrent programming）中，深入理解并且掌握JMM（java内存模型）是非常有必要的。因为在多线程运行的过程中，少不了会出现竞争、数据可见性、执行顺序等等问题，而想要理解并且正确的处理这些问题，就需要对JMM进行学习。

## Java代码运行流程

首先，我们写的java代码其实是高度抽象程度的。当我们写完代码，将代码交给JVM后，jvm和底层的CPU都是做了很多工作的。我们在代码中认为的一个简单操作，可能底层却是一个序列的子操作来支撑的。我们来看下一副图：

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220107153830406.png" alt="image-20220107153830406" style="zoom:50%;" />

这幅图其实是简单的代码执行流程图，我们其实可以将JVM当作一个java的操作系统，对上屏蔽了底层不同架构、不同操作系统的区别。所以我们可以抽象的认为字节码文件运行在JVM这个操作系统上。

当我们将代码交给JVM后，JVM对代码做了非常多的优化。这些优化非常的aggressive，比如修改源码的执行顺序（有些甚至相当于修改源码，当然前提是不改变程序的执行结果）。但是这些优化到了多线程并发的时候（多个线程运行同一份代码），就会出现问题了，而JMM正是来保证在并发条件下也能正确执行（运行结果符合开发者的逻辑）下面我们来看一个优化代码的例子：

```java
int a = 1;
a = a + 2;
a = a + 3;
//优化后
int a = 1;
a = a + 5;
```

由上面可见，只要是不影响最后程序的结果，jvm就会去进行优化。上述代码中，第一种写法需要两次的读操作（不算实例化过程），而第二种只需要一次读操作，相当于提高了程序的执行效率，而且两次执行后的结果都是相同的。当然这里只是个很简单的例子。

## 硬件层次的内存模型

每个程序无非就是数据和指令集的集合，最后都是CPU来负责每条指令的执行，而相关指令所需的数据都是在主存（RAM）中。这样就出现了一个问题，CPU的速度非常快，而RAM主存的速度相对于CPU执行指令的速度则是要慢几个数量级

<img src="https://formulusblack.com/wp-content/uploads/2019/02/Screen-Shot-2019-02-01-at-12.17.22-PM.png" alt="Compute Performance – Distance of Data as a Measure of Latency | Formulus  Black | In-Memory Storage" style="zoom:50%;" />

上图是计算机硬件等级中，各级存储结构的读取速度。所以为了解决这个问题，就提出了物理内存模型的三级结构



<img src="https://xhy3054.github.io/assets/img/data_store/%E5%A4%9A%E6%A0%B8%E5%A4%84%E7%90%86%E5%99%A8%E7%9A%84%E7%BB%84%E7%BB%87%E7%BB%93%E6%9E%84.JPG" alt="img" style="zoom:50%;" />

上图中的寄存器register是cpu的存储单元，跟cpu的计算单元是同样的速度。也有人叫它L0 cache。

以下这段话引用自：https://xhy3054.github.io/register-cache-memory/

> CPU本身只负责运算，不负责储存数据。数据一般储存在内存(memory)之中，CPU要用的时候就去内存读写数据。但是，CPU的运算速度远高于内存的读写速度，为了避免被拖慢，CPU一般都会自带一级缓存与二级缓存甚至三级缓存（cache）。基本上，**CPU缓存可以看作是读写速度较快的内存**。
>
> 但是，CPU缓存还是不够快，另外数据在缓存里面的地址是不固定的，CPU每次读写还需要做寻址操作，这会明显的拖慢速度。因此，除了缓存之外，CPU还自带了寄存器（register），寄存器不依靠地址区分数据，而是依靠名称来按位访问，速度最快，有的人称它为**零级缓存**。寄存器常常用来储存最常用的数据。也就是说，那些最频繁读写的数据（比如循环变量），都会放在寄存器里面，CPU优先读写寄存器，再由寄存器跟内存交换数据。

上面的物理内存模型，我们其实可以简化成下图

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220107170719953.png" alt="image-20220107170719953" style="zoom:60%;" />

上面的多级缓存虽然解决了CPU执行效率的问题（CPU此时不必花费大量时间等待数据），但是由于现代的主机都是多核，多个CPU，很多也支持多线程。于是在并发多线程环境下，多个cpu对主存中同一个数据可能会有读写操作。但是由于并发环境下，各个线程执行的顺序是不一定的（由操作系统调度），所以自然会带来这个问题：如何保证缓存中数据的一致性？为了解决这个问题，我们需要各个CPU在访问缓存时都遵循一些协议，在读写时要根据协议来操作。这些协议被称为缓存一致性协议，其中最为出名的莫过于intel的MESI 协议。

<img src="https://raw.githubusercontent.com/dunwu/images/dev/snap/20210102230327.png" alt="img" style="zoom:50%;" />

上面的多级缓存是为了提高CPU的效率，但是除开这个，CPU还可能对输入的指令（代码）进行乱序执行优化（out-of-order execution）。处理器会在计算之后将乱序执行的结果重组，**保证该结果与顺序执行的结果是一致的**，但不保证程序中各个语句计算的先后顺序与输入代码中的顺序一致。

稍微展开下乱序执行优化，因为在JVM执行也有相似的优化。比如CPU在执行一条load指令，此时这个指令的数据地址不在register中，我们需要计算地址，然后加载到register中，但是在加载的这个过程，cpu需要等着；如果后面紧跟的是一个add运算，而且add运算的值都已经在register中了，那我们为何不让cpu等待的时间去执行add运算呢？对的，cpu也通常是这么做的，所以cpu执行的代码顺序跟我们写的可能是不一样的。但这个是必须要有的优化。**CPU不保证执行顺序，但是保证执行的结果符合逻辑的（单线程执行下）**

## 多线程环境下面临的问题

上面的硬件模型中，CPU对指令的优化（乱序执行），多级缓存对CPU效率的提升都是有利于程序运行效率的。但是在多线程并发环境下就会出现问题了。在多线程并发下，由于程序用公用的数据内存区域，而并发时各个线程的执行调度又归于操作系统，所以如何在这种情况下保证数据的：可见性（visible），有序性（ordering），原子性（atomic）就变得及其重要。

### 可见性

**可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值**。

下面的代码中，thread1中的循环可能会一直死循环。因为线程的调度归操作系统，我们想象以下，flag在内存中，thread2将flag从内存中读取到缓存中，然后cpu修改它，但是此时thread2被阻塞了，一直挂着。那么更新后的flag就不能被flush回到主存中，而cpu彼此的缓存又是不可公用的，那么thread1中的循环就会一直循环下去。

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220107175742324.png" alt="image-20220107175742324" style="zoom:50%;" />

而且Java中还有更激进的优化，上图的代码我们如果只从thread1或者thread2的视角去看是这样的：

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220107180350960.png" alt="image-20220107180350960" style="zoom:50%;" />

thread1和thread2互相不知道对方的运行指令，所以运行的JVM会认为在threa2中，flag赋值后从来没有被读取，那么我就不执行它赋值的这一步了；同样的对于thread1, JVM回认为，flag变量从来没有被赋值过，所以我就不在从主存中去读取flag了，直接把它替换成true。那么也会造成thread1中的死循环。

### 有序性

这里的有序性是说：我们在编写业务逻辑时，需要一些步骤在前，一些步骤在后（比如初始化数据库必须在读取数据库之后），但是我们将代码交给下层执行时（比如CPU），底层会**进行out-of-order execution优化**。从而导致我们多线程情况下，获得不符合逻辑的结果。

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220107174818277.png" alt="image-20220107174818277" style="zoom:50%;" />

上图中，我们的初始意图是第一种代码写法。但是CPU会优化成第二种写法（乱序优化），进而可能优化成第三种写法。于是在多线程下，就会出现右边下方两种执行过程中foo与bar的值不符合逻辑（相对于第一种写法）。无论如何从第一种写法中，我们都无法得出何时foo==3,bar==0。

有序性的另外一个例子：

```java
public class Singleton {
    int foo;
    private Singleton() {foo = 45 }
    private volatile static Singleton instance;//如果不加volatile，会怎么样？
    public Singleton getInstance(){
        if(instance==null){
            synchronized (Singleton.class){
                if(instance==null){
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
	void method() {assert foo == 45}//如果没有在instance上加volatile，这里会不会为false？
}
```

我们虽然在写代码时，只用new就可以新建对象了。但是JVM却是把新建对象的过程分为3步，伪代码如下

```
#1 allocate(obj);
#2 initialize(obj);执行constructor
#3 obj = getRerence(obj)
```

 由于Jvm的乱序优化，所以上面三个步骤的顺序并不能得到保证，如果是单线程，无所谓，因为JVM会保证所有步骤都完成即使是乱序的。但是多线程呢？那就有可能#2#3的顺序互换，那么此时创建instance的一个线程阻塞了（正在执行#2的constructor），然后另外一个线程一检查发现instance不是null（只要执行了#1后，obj就不是null了），直接返回instance，但是此时instance并没有被初始化，而且我们也不知道多久后会初始化，那么后续使用instance就会出现问题。

### 原子性

原子性是指**一个操作是不可中断的，要么全部执行成功要么全部执行失败**。比如以下例子

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220107203109981.png" alt="image-20220107203109981" style="zoom:50%;" />

在一些架构上，64bits类型的变量在赋值操作时不是原子操作（因为jvm是32bits）,所以就会出现对long，double类型的赋值是分为两步，头32bits和后32bits。在多线程并发环境下，此时如果一个线程正在给foo变量的前32bit赋值，然后此时另外一个线程来给后32bits赋值，我们就会得到全1也就是-1这个值，但是从代码中我们却丝毫看不出这是如何得出的。

## Java Memory Model

我们经过上面的介绍后，能明白如果是单线程环境，底层（JVM和CPU）的优化几乎都不会影响到程序的执行结果，但是对于多线程并发下，由于多个线程有共享的数据区域，那么多个线程在CPU的schedule下对共享数据的操作就有可能引发各种问题。而这些问题的出现对于我们并发开发造成了极大的困扰，而此时JMM Kicks in，就是来解决这些问题的。而这三大特性，归根结底，是为了实现多线程的 **数据一致性**，使得程序在多线程并发，指令重排序优化的环境中能如预期运行。

JMM原本是一篇PhD论文， 非常的学术化，但是其实很简单,JMM就做了一个事情：**线程在读取指定变量时，能读到什么值。**

正式的定义是：将java代码分解成许多action的集合，而且对这些action赋予特定的ordering。如果读取的是一个满足happen-before guarantee的变量，那么JMM保证read操作返回一个特定的值。

那么有哪些操作呢？

java定义的主内存（所有线程公用的）和进程缓存（独属于线程的）的数据操作有：lock, unlock,read, load, use, assign, store，write共计8个操作。同时这8个操作JVM保证是原子性的（对于long，double类型在一些架构上比如ARM上不是原子的）

### **Happens-before guarantee**

首先happens-before guarantee是致力解决**共享变量**的**可见性**问题，在多线程并发时，由于

1. **程序顺序**

   **在一个线程中**，按照action的**依赖顺序**，之前的操作happens before 之后的操作。这里很多人会理解成是代码顺序。但我的理解是在单线程中，对有着依赖关系的操作，肯定是顺序发生。注意并不是代码顺序，如果代码之间没有相互依赖，那么JVM就会可能会进行重排序提高运行效率。

   ```
   int a = 1;//#1
   int c = 2;//#2
   int d = 4;//#3
   int b = a + c;//#4
   ```

   上面的代码中，按照程序规则JMM保证#1,#2肯定要发生在#4之前，但是#3是不是发生在#4之前就不一定了。同理，#1和#2的顺序也不一定，因为他们之间并没有依赖关系。这里可以理解成在单线程中，JMM给开发者一种错觉：**你的程序执行结果跟as-if-serial执行的结果一样，你不用管我在底层怎么reordering优化或者CPU怎样out-of-order execution了。**

2. **Monitor lock**

   > An unlock on a monitor lock (exiting synchronized method/block) happens-before every subsequent acquiring on the same monitor lock.
   >
   > 在monitor lock（管程锁）的unlock操作，先于任何后续对该锁的获取操作

​		JVM 并没有把 `lock` 和 `unlock` 操作直接开放给用户使用，

​		但是却提供了更高层次的字节码指令 `monitor enter` 和 `monitor exit` 来隐式地使用这两个操作。

​		这两个字节码指令反映到 Java 代码中就是同步块`synchronized`。

​		我们看下列图就会明白了

​		<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220108205714951.png" alt="image-20220108205714951" style="zoom:50%;" />

​		代码实例：				

```java
synchronized (this) { // 此处自动加锁
	if (x < 1) {
        x = 10;
    }      
} // 此处自动解锁
```

​		上面的代码中，我们假设有线程A和B都会运行这个方法，但是由于线程的执行由CPU调度，所以我们并不知	    道哪一个线程先执行。但是根据第二条happens-before guarantee,我们设B先执行这个方法，进入这个方法		时需要对this，lock操作，然后执行，那怕此时A的调度允许A来调用这个方法，A也不能执行，因为不能上一		个线程的unlock操作还没有进行，自己的lock就不能继续。verse versa

3. **Volatile，对于一个有volatile修饰的变量，对于它的write操作必须先于下一个对它的read操作**	

   volatile其实解决的是共享变量的可见性问题，也是java中最轻量级别的同步机制。那他是如何解决可将行问题的呢？

   我们知道造成共享变量并发时出现可见性问题**是由于缓存机制导致的**，于是volatile修饰的变量相当于是每次不经过缓存，每当线程需要读取volatile变量操作时，是直接从RAM中去读取的；同样的当一个线程写一个volatile变量时，也是直接从工作内存（比如register）直接写入RAM。这样一来相当于一个volatile变量无论何时被改变后，它的值都会被非常快的刷新到主存，而所有读取它的操作也是从主存读，就解决了可见性问题。（我们可以认为从工作内存写入RAM是非常快的）

   但是从上面的描述我们也发现了问题，就是**volatile并不保证原子性**！

   在解释这个之前我们先看一个名词：**内存屏障**，这也是java底层实现可见性具体操作。

   下列描述引用自：https://dunwu.github.io/javacore/concurrent/Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.html#_4-%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C

   > Java 中如何保证底层操作的有序性和可见性？可以通过内存屏障（memory barrier）。
   >
   > 内存屏障是被插入两个 CPU 指令之间的一种指令（汇编指令），用来禁止处理器指令发生重排序（像屏障一样），从而保障**有序性**的。另外也会强制刷出各种CPU的缓存数据，因此任何CPU上的线程都能读 取到这些数据的最新版本，从而保障**可见性**。
   >
   > 举个例子：
   >
   > ```text
   > Store1;
   > Store2;
   > Load1;
   > StoreLoad;  //内存屏障
   > Store3;
   > Load2;
   > Load3;
   > ```
   >
   > 对于上面的一组 CPU 指令（Store 表示写入指令，Load 表示读取指令），StoreLoad 屏障之前的 Store 指令无法与 StoreLoad 屏障之后的 Load 指令进行交换位置，即**重排序**。但是 StoreLoad 屏障之前和之后的指令是可以互换位置的，即 Store1 可以和 Store2 互换，Load2 可以和 Load3 互换。
   >
   > 常见有 4 种屏障
   >
   > - `LoadLoad` 屏障 - 对于这样的语句 `Load1; LoadLoad; Load2`，在 Load2 及后续读取操作要读取的数据被访问前，保证 Load1 要读取的数据被读取完毕。
   > - `StoreStore` 屏障 - 对于这样的语句 `Store1; StoreStore; Store2`，在 Store2 及后续写入操作执行前，保证 Store1 的写入操作对其它处理器可见。
   > - `LoadStore` 屏障 - 对于这样的语句 `Load1; LoadStore; Store2`，在 Store2 及后续写入操作被执行前，保证 Load1 要读取的数据被读取完毕。
   > - `StoreLoad` 屏障 - 对于这样的语句 `Store1; StoreLoad; Load2`，在 Load2 及后续所有读取操作执行前，保证 Store1 的写入对所有处理器可见。它的开销是四种屏障中最大的（冲刷写缓冲器，清空无效化队列）。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。
   >
   > **如果你的字段是volatile，Java内存模型将在写操作后插入一个写屏障指令，在读操作前插入一个读屏障指令**
   >
   > 以下内容引用自：https://www.zhihu.com/question/329746124
   >
   > **这样咋一看貌似可以保证线程的安全性呀，为啥不能保证呢？**
   >
   > 这样如果有一个变量= 0用volatile修饰，两个线程对其进行i++操作，如果线程1从内存中读取i=0进了缓存，然后把数据读入，之后时间片用完了，然后线程2也从内存中读取i进缓存，因为线程1还未执行写操作，内存屏障是插入在写操作之后的指令，意味着还**未触发这个指令，所以缓存行是不会失效**的。然后线程2执行完毕，内存中i=1，然后线程1又开始执行，然后将数据写回缓存再写回内存，结果还是1。

4. **传递性：如果A happens-before B，且B happens-before C，那么A happens-before C**

5. Thread.start()先于被启动线程中的任意动作

   > A call to Thread.start() on a thread happens-before every action in the started thread. Say thread A spawns a new thread B by calling threadA.start(). All actions performed in thread B's run method will see thread A's calling threadA.start() method and before that (only in thread A) happened before them.
   >
   > 在一个线程中启动另外一个线程的start()方法先于被启动线程的任何动作，也就是在被启动线程中，主线程和它的共享变量（在调用start()方法前的）对它是可见的。
   >
   > ![image-20220108220149565](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220108220149565.png)

6. **Thread join rule:一个线程中的所有action先于其他线程对它的thread.join()方法调用返回**

   > All actions in a thread happen-before any other thread successfully returns from a join on that thread. Say thread A spawns a new thread B by calling threadA.start() then calls threadA.join(). Thread A will wait at join() call until thread B's run method finishes. After join method returns, all subsequent actions in thread A will see all actions performed in thread B's run method happened before them.
   >
   > 
   >
   > <img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220108222144535.png" alt="image-20220108222144535" style="zoom:50%;" />

7. **线程中断规则** - 对线程 `interrupt()` 方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过 `Thread.interrupted()` 方法检测到是否有中断发生。

8. **对象终结规则** - 一个对象的初始化完成先行发生于它的 `finalize()` 方法的开始。

## Java当中的运用

### volatile（field scoped）

上面我们已经提到了volatile关键字的用法和volatile的底层实现是通过内存屏障来实现的。这里我们要注意的是，volatile关键字只对变量起作用，**而且只保证修饰变量在并发环境下对其他线程的可见性而不保证对该变量的原子性**

使用例子：

```java
class SharedObj
{
   // volatile keyword here makes sure that
   // the changes made in one thread are 
   // immediately reflect in other thread
   static volatile int sharedVar = 6;
}
```

如果多线程并发时，只有一个线程具有修改共享变量的权限，而剩余其他线程只能读取共享变量（或者能在不依靠当前变量值的基础上进行修改操作）的情形下。我们对这个共享变量使用volatile修饰也能达到同步的效果，也就是同时保证了可见性和原子性。

### synchronized(method scoped)

上面在happends-before guarantee的第2条，其实就是sybchronized关键字的原理。java对语言层次锁的体现就是synchronized，使用这个关键字声明的方法或者代码块任何时候只能有一个线程同步的执行。synchronized锁定的代码块或者方法只能被一个线程在某一个时间同步执行，**保证了共享资源的可见性和原子性**。在每次进入synchronized修饰的代码块或者方法之前，对当前线程能见的

synchronized关键字的底层是用monitor实现的，看下列代码（使用javap解析字节码）

**以下代码转载自：https://juejin.cn/post/6844903757709312014**

```
public class SynchronizedThis {
	public void method() {
		synchronized(this) {}
	}
}

// 反编译结果
public void method();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: aload_1
         5: monitorexit
         6: goto          14
         9: astore_2
        10: aload_1
        11: monitorexit
        12: aload_2
        13: athrow
        14: return
----------------------------------------------分界线--------------------------------------
  public class SynchronizedMethod {
	public synchronized void method() {}
  }

// 反编译结果
public synchronized void method();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 2: 0
```

我们可以看到底层是在同步代码块（syncronized修饰的代码块）最开始加了monitorenter，执行完代码块后加了monitorexit，并且要是在代码块中抛出了异常也不用死锁，一旦抛出了异常，也会执行monitorexit。同样的对于syncronized修饰的方法，jvm会加一个ACC_SYNCHRONIZED字段标记该方法，然后之后执行的时候也是利用了monitor。

那么这个monitor是什么？

monitor简单的来说是一个类，使用C++实现的。java中的每一个对象在内存中的存储形式是：**对象头**、**实例数据**和**对齐填充Padding**

> 由于Java面向对象的思想，在JVM中需要大量存储对象，存储时为了实现一些额外的功能，需要在对象中添加一些标记字段用于增强对象功能，这些标记字段组成了对象头。
>
> 对象头和monitor详解：https://www.cnblogs.com/aspirant/p/11470858.html#!comments

其中在对象头中有个mark word字段，如果对某一个对象加锁操作，那么此时在该对象的mark word就存储有该对象对应的monitor的地址。通过这个monitor对象提供的方法调用（C++实现），就可以实现对这个对象的加锁和解锁（monitor里有一个字段是存储归属线程id的）同时这也是为什么在java中任何对象都可以作为加锁的对象的原因。**任意线程对Object（Object由synchronized保护）的访问，首先要获得Object的监视器。如果获取失败，线程进入同步队列，线程状态变为BLOCKED。当访问Object的前驱（获得了锁的线程）释放**了锁，则该释放操作唤醒阻塞在同步队列中的线程，使其重新尝试对监视器的获取。

### JUC(method scoped)

之后再探究

### final（field scoped）

final关键字，在java中可以修饰类、方法、字段。我们这里只讲final修饰字段。

当使用final修饰字段时，表示这是一个在初始化后就无法修改的常量，对于基本变量那么它的值无法修改，对于引用变量，初始化后地址无法修改，但是引用变量所指向的对象可以修改。

而且使用final修饰字段时，如果这个字段是多线程并发下的共享资源，还解决了共享资源的可见性问题。

我们新建一个对象分为三步(前面提过)，如果不用final修饰，那么JVM是可能对初始化和将分配的地址赋给引用变量这一步重排序的，就会导致我们的引用变量虽然不为null（从第一步分配内存开始就不为null），但是我们的初始化并没有完成。（典型的double checked），但是使用final修饰的字段，会在#2#3步插入一个内存屏障，静止重排序，自然就保证了可见性。



## 总结

***以上内容是作者本人学习所需，如果错漏之处，请多包涵。如有讨论，请友善发言！***