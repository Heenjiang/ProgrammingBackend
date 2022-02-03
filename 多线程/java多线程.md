# Java多线程机制

## 简述

现在的计算机一般都有多个core，而且由于CPU的计算速度其实非常快，所以非常适合用于多任务并发。而并发的方式一般分为两种，多线程和多进程。线程是计算机最小的任务单位，同时一个进程可以包括多个线程。进程是拥有自己独立的地址空间，各个进程的地址空间相互独立。而线程同属于一个进程，有共享的地址空间。

Java中采用多线程来实现多任务并发。

## 创建线程的方式

1. 通过继承Thread类，并且重写run方法，实现一个Thread的子类。然后就可以通过创建该类的实例来创建一个线程，调用start()方法启动线程

2. 通过实现Runnable接口写一个类，重写run方法，实现一个Runnable接口的类。然后通过创建该类的实例作为target传入Thread的构造方法来创建一个线程，调用start()方法启动线程

3. 通过实现callable接口写一个类，重写call()方法（有返回值），然后将这个类的实例传入FutureTask 的构造函数，创建一个FutureTask 的实例，再将FutureTask的实例作为target传入Thread的构造方法来创建一个线程，可以通过Future类的get()方法来获取回调值(阻塞获取)

4. 通过实现Runnable结构写一个类，重写run方法，创建一个线程池，通过ExecutorService实例的execute方法传入实现Runnable接口类的实例，执行线程

   ```java
   package wayforcreatethreads;
   
   import java.util.Random;
   import java.util.concurrent.Callable;
   import java.util.concurrent.ExecutionException;
   import java.util.concurrent.ExecutorService;
   import java.util.concurrent.Executors;
   import java.util.concurrent.Future;
   import java.util.concurrent.FutureTask;
   
   public class HelloThread{
   	
   	 public static void main(String[] args) {
   		 //通过实现Thread的子类，重写run方法来实现线程
   		 Thread t = new MyThread();
   		 t.start();//自动调用t.run()
   		 
   		 //通过向Thread的构造方法中传入一个Runable类作为target
   		 Thread t1 = new Thread(new MyRunnable());
   		 t1.start();
   		 
   		 //通过实现callable接口
            //FutureTask是一个包装器，
   		 //它通过接受Callable来创建，它同时实现了Future和Runnable接口。
   		 FutureTask<Integer> futureTask = new FutureTask<>(new MyCallableThread());
   		
   		 Thread t2 = new Thread(futureTask);
   		 t2.start();
   		 try {
   			System.out.println("blocking execute");
   			System.out.print(futureTask.get() + "\n");
   		} catch (InterruptedException e) {
   			e.printStackTrace();
   		} catch (ExecutionException e) {
   			e.printStackTrace();
   		}
   		 
   		//通过线程池创建
   		 ExecutorService executorService = Executors.newFixedThreadPool(10);
   		 for (int i = 0; i < 10; i++) {
   			executorService.execute(new MyRunnableInExecutorService());
   		}
   		 
   	 }
   }
   
   class MyThread extends Thread {
   	@Override
   	public void run() {
   		System.out.println(Thread.currentThread().getName()
                   +" This is a new thread echoing!\nI am created by extending "
   				+ "the Thread class and override the run method\n");
   	}
   }
   
   class MyRunnable implements Runnable{
   	@Override
   	public void run() {
   		System.out.println(Thread.currentThread().getName()
                   +" This is a new thread echoing\n"
   				+ "I am created by implementing the Runnable interface and"
   				+ "override the run method and as a parameter passed to the "
   				+ "constructor of Thread class \n");
   	}
   }
   
   class MyCallableThread implements Callable<Integer>{
   	@Override
   	public Integer call() throws Exception{
   		Thread.sleep(2000);
   		System.out.println(Thread.currentThread().getName()
                   +" This is a new thread echoing\n"
   				+ "I am created by implementing the Callable interface");
   		return new Random().nextInt(200);
   	}
   }
   
   class MyRunnableInExecutorService implements Runnable{
   	@Override
   	public void run() {
   		System.out.println(Thread.currentThread().getName()
                   +" This is a new thread echoing\n"
   				+ "I am created by implementing the Runnable interface and "
   				+ "executing by Thread pool \n");
   	}
   }
   
   ```

   执行结果：

   ```asciiarmor
   Thread-0 This is a new thread echoing!
   I am created by extending the Thread class and override the run method
   
   Thread-1 This is a new thread echoing
   I am created by implementing the Runnable interface andoverride the run method and as a parameter passed to the constructor of Thread class 
   
   blocking execute
   Thread-2 This is a new thread echoing
   I am created by implementing the Callable interface
   82
   pool-1-thread-1 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   pool-1-thread-4 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   pool-1-thread-3 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   pool-1-thread-2 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   pool-1-thread-5 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   pool-1-thread-6 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   pool-1-thread-7 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   pool-1-thread-9 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   pool-1-thread-8 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   pool-1-thread-10 This is a new thread echoing
   I am created by implementing the Runnable interface and executing by Thread pool 
   
   ```

   但是上述四种方法其实本质都是实现Thread类，具体解释看：

   https://blog.csdn.net/m0_37840000/article/details/79756932

**直接调用`run()`方法，相当于调用了一个普通的Java方法**， **必须调用`Thread`实例的`start()`方法才能启动新线程**，如果我们查看`Thread`类的源代码，会看到`start()`方法内部调用了一个`private native void start0()`方法，`native`修饰符表示这个方法是由JVM虚拟机内部的C代码实现的，不是由Java代码实现的。

## 线程的优先级

可以对线程设定优先级，设定优先级的方法是：

```
Thread.setPriority(int n) // 1~10, 默认值5
```

优先级高的线程被操作系统调度的优先级较高，**操作系统对高优先级线程可能调度更频繁**，**但我们决不能通过设置优先级来确保高优先级的线程一定会先执行。**

## 线程状态

```ascii
         ┌─────────────┐
         │     New     │
         └─────────────┘
                │
                ▼
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
 ┌─────────────┐ ┌─────────────┐
││  Runnable   │ │   Blocked   ││
 └─────────────┘ └─────────────┘
│┌─────────────┐ ┌─────────────┐│
 │   Waiting   │ │Timed Waiting│
│└─────────────┘ └─────────────┘│
 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                │
                ▼
         ┌─────────────┐
         │ Terminated  │
         └─────────────┘
```

- New：新创建的线程，**尚未执行**；
- Runnable：运行中的线程，**正在执行`run()`方法的Java代码**；（但是也有可能在等待资源）
- Blocked：运行中的线程，**因为某些操作被阻塞而挂起**；
- Waiting：运行中的线程，因为某些操作在等待中；
- Timed Waiting：运行中的线程，因为执行`sleep()`方法正在计时等待；
- Terminated：线程已终止，因为`run()`方法执行完毕。

当线程启动后，它可以在`Runnable`、`Blocked`、`Waiting`和`Timed Waiting`这几个状态之间切换，直到最后变成`Terminated`状态，线程终止。

## 线程终止的原因有：

- 线程正常终止：`run()`方法执行到`return`语句返回；
- 线程意外终止：`run()`方法因为未捕获的异常导致线程终止；
- 对某个线程的`Thread`实例调用`stop()`方法强制终止（强烈不推荐使用）。

## 线程中断

如果线程需要执行一个长时间任务，就可能需要中断该线程。**中断线程就是其他线程给该线程发一个信号**，该线程收到信号后结束执行`run()`方法，使得自身线程能立刻结束运行。

1. 在其他线程中对目标线程调用`interrupt()`方法，目标线程需要反复检测自身状态是否是interrupted状态，如果是，就立刻结束运行。

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new MyThread();
        t.start();
        Thread.sleep(1); // 暂停1毫秒
        t.interrupt(); // 中断t线程
        t.join(); // 等待t线程结束
        System.out.println("end");
    }
}

class MyThread extends Thread {
    public void run() {
        int n = 0;
        while (! isInterrupted()) {
            n ++;
            System.out.println(n + " hello!");
        }
    }
}
```

**注意，`interrupt()`方法仅仅向`t`线程发出了“中断请求”，至于`t`线程是否能立刻响应，要看具体代码**

2. 被中断的线程处于等待状态，此时其他的线程对它调用interrupt()方法会使得等待状态的线程处于等待的那个方法（Thread.sleep(), join()等）抛出InterruptException，此时只需要在被中断的线程中捕获这个异常，然后处理就行了（通常是直接结束线程）

   ```java
   public class Main {
       public static void main(String[] args) throws InterruptedException {
           Thread t = new MyThread();
           t.start();
           Thread.sleep(1000);
           t.interrupt(); // 中断t线程
           t.join(); // 等待t线程结束
           System.out.println("end");
       }
   }
   
   class MyThread extends Thread {
       public void run() {
           Thread hello = new HelloThread();
           hello.start(); // 启动hello线程
           try {
               hello.join(); // 等待hello线程结束
           } catch (InterruptedException e) {
               System.out.println("interrupted!");
           }
           hello.interrupt();
       }
   }
   
   class HelloThread extends Thread {
       public void run() {
           int n = 0;
           while (!isInterrupted()) {
               n++;
               System.out.println(n + " hello!");
               try {
                   Thread.sleep(100);
               } catch (InterruptedException e) {
                   break;
               }
           }
     
   ```

   

   上面代码需要注意的是：如果去掉MyThread类的最后一行代码，如果去掉这一行代码，可以发现`hello`线程仍然会继续运行，且JVM不会退出

   3. 设置标志位。我们通常会用一个`running`标志位（该字段要保证可见性，用volatile修饰）来标识线程是否应该继续运行，在外部线程中，通过把`HelloThread.running`置为`false`，就可以让线程结束：

      ```
      public class Main {
          public static void main(String[] args)  throws InterruptedException {
              HelloThread t = new HelloThread();
              t.start();
              Thread.sleep(1);
              t.running = false; // 标志位置为false
          }
      }
      
      class HelloThread extends Thread {
          public volatile boolean running = true;
          public void run() {
              int n = 0;
              while (running) {
                  n ++;
                  System.out.println(n + " hello!");
              }
              System.out.println("end!");
          }
      }
      
      ```

      **但是一定要注意修改volatile变量的最好只有一个线程**

## 守护线程（Damon Thread）

JVM退出的条件是：JVM中运行的全是守护线程时。

> The Java Virtual Machine exits when the only threads running are all daemon threads.

Java的线程类型有两种：用户线程和守护线程。用户线程的优先级高，而守护线程一般来说是为用户线程提供服务的（比如GC的线程）。当JVM中运行的只剩下守护线程时，JVM就会退出（自动退出，虽然不是马上自动退出）。

以下图片来自于：https://www.cnblogs.com/quanxiaoha/p/10731361.html

![设置该线程为守护线程](https://exception-image-bucket.oss-cn-hangzhou.aliyuncs.com/155550846091464)

![第二次运行示例代码](https://exception-image-bucket.oss-cn-hangzhou.aliyuncs.com/155550859267171)

如何创建守护线程呢？方法和普通线程一样，只是在调用`start()`方法前，调用`setDaemon(true)`把该线程标记为守护线程，在守护线程中，编写代码要注意：**守护线程不能持有任何需要关闭的资源**，例如打开文件等，因为虚拟机退出时，守护线程没有任何机会来关闭文件，这会导致数据丢失。

## Synchronized

保证一段代码的原子性就是通过加锁和解锁实现的。Java程序使用`synchronized`关键字对一个对象进行加锁：

```java
synchronized(lock) {
    n = n + 1;
}
```

如何使用`synchronized`：

1. 找出修改共享变量的线程代码块；
2. 选择一个共享实例作为锁；
3. 使用`synchronized(lockObject) { ... }`。

在使用`synchronized`的时候，不必担心抛出异常。**因为无论是否有异常，都会在`synchronized`结束处正确释放锁：**

Synchronized是通过对象header中的monitor（管程来实现的）【每个对象header都有一个monitor对象，monitor对象的一个字段表明这个对象是否被锁】

## 不需要synchronized的操作

JVM规范定义了几种原子操作：

- 基本类型（`long`和`double`除外）赋值，例如：`int n = m`；
- 引用类型赋值，例如：`List<String> list = anotherList`。

`long`和`double`是64位数据，JVM没有明确规定64位赋值操作是不是一个原子操作，不过在x64平台的JVM是把`long`和`double`的赋值作为原子操作实现的。

单条原子操作的语句不需要同步。例如：

```java
public void set(int m) {
    synchronized(lock) {
        this.value = m;
    }
}
```

有些时候，通过一些巧妙的转换，可以把非原子操作变为原子操作（将多个同类型的赋值，变成数组赋值）

## 线程安全

**如果一个类被设计为允许多线程正确访问，我们就说这个类就是“线程安全”的（thread-safe）**

例子：我们知道Java程序依靠`synchronized`对线程进行同步，使用`synchronized`的时候，锁住的是哪个对象非常重要。

**让线程自己选择锁对象往往会使得代码逻辑混乱，也不利于封装**。更好的方法是**把`synchronized`逻辑封装起来**。例如，我们编写一个计数器如下：

```java
public class Counter {
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

java.lang.StringBuffer（很多方法都是用Synchronized修饰，跟HashTable类似），还有一些不变类，例如`String`，`Integer`，`LocalDate`类似`Math`这些只提供静态方法，没有成员变量的类，也是线程安全的

除了上述几种少数情况，**大部分类**，例如`ArrayList`，**都是非线程安全的类**，我们不能在多线程中修改它们。但是，**如果所有线程都只读取，不写入**，那么`ArrayList`**是可以安全地**在线程间共享的。

## wait和notify

`synchronized`解决了多线程竞争的问题。但是`synchronized`并没有解决多线程协调的问题。

```java
class TaskQueue {
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s) {
        this.queue.add(s);
    }

    public synchronized String getTask() {
        while (queue.isEmpty()) {
        }
        return queue.remove();
    }
}
```

上面的代码是一个任务队列的实现，在并发条件下，getTask()是存在问题的：如果一个线程调用getTask()方法时，会首先对this对象加锁，然后判断queue是否为空，但是此时如果为空了，就会一直循环，而其他线程因为无法获取当前实例this的锁，所以也无法addTask()，那么这里的while循环就会一直循环占用100%的CPU。所以这也是为什么说synchronized解决了竞争问题（加锁），但是并没有解决多线程协调的问题。

java中也提供了多线程协调的机制，多线程协调的原则是：当条件不满足时，线程进入等待状态；当条件满足时，线程被唤醒，继续执行任务。

wait和notify就是提供使线程等待或者唤醒线程的，我们可以改造上面的代码

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        var q = new TaskQueue();
        var ts = new ArrayList<Thread>();
        for (int i=0; i<5; i++) {
            var t = new Thread() {
                public void run() {
                    // 执行task:
                    while (true) {
                        try {
                            String s = q.getTask();
                            System.out.println("execute task: " + s);
                        } catch (InterruptedException e) {
                            return;
                        }
                    }
                }
            };
            t.start();
            ts.add(t);
        }
        var add = new Thread(() -> {
            for (int i=0; i<10; i++) {
                // 放入task:
                String s = "t-" + Math.random();
                System.out.println("add task: " + s);
                q.addTask(s);
                try { 
                    Thread.sleep(100); 
                } 
                catch(InterruptedException e) {}
            }
        });
        add.start();
        add.join();
        Thread.sleep(100);
        for (var t : ts) {
            t.interrupt();
        }
    }
}

class TaskQueue {
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s) {
        this.queue.add(s);
        this.notifyAll();
    }

    public synchronized String getTask() throws InterruptedException {
        while (queue.isEmpty()) {
            this.wait();
        }
        return queue.remove();
    }
}

```

**wait()方法**：线程调用该方法会使得线程进入等待状态，wait()方法会一直不返回直到将来的某个时刻被其他线程唤醒后，wait()方法才会返回，然后执行下一条语句。同时wait()方法的底层是JVM调用底层C实现的（是object中被native修饰的方法），线程调用wait后会**释放**当前synchronized块所持有的锁，wait返回后又会**试图**重新获得锁。所以wait必须在锁对象上调用，而且必须在synchronized块中才能调用。

**notify**：线程对锁对象调用notify()方法，会唤醒一个正在等待锁对象的线程，从而使得等待的线程从wait()方法中返回。（注意只是唤醒等待的一个线程，而且不一定是哪一个）。

**notifyAll：**同上，只不过会唤醒所有等待锁对象的线程，然后被唤醒的线程会去竞争锁对象。

注意我们在getTask中用了while循环来检查任务队列，而不是用的if。这是因为，notifyAll会唤醒所有等待的线程，但是等待的线程从wait中返回时，需要获得this锁对象，而this锁对象只有一个，所以在addTask中唤醒所有线程，并且addTask方法执行完后，只有一个被唤醒的线程能够获得this锁。所以剩下的被唤醒的线程没有竞争到this锁，自然还会等待。当前获得this锁的线程会从队列中取出一个任务，之后执行完getTask方法后会释放this锁，让给剩下的线程，那么下一个得到this锁的线程自然还是需要检查队列的状态（可以理解为，**每次获得this锁后都需要检查队列，因为不知道队列是不是被change过**）

## 死锁

一个线程可以获取一个锁后，再继续获取**另一个锁**。在获取多个锁的时候，**不同线程**获取**多个不同对象**的锁可能导致死锁。

```java
public void add(int m) {
    synchronized(lockA) { // 获得lockA的锁
        this.value += m;
        synchronized(lockB) { // 获得lockB的锁
            this.another += m;
        } // 释放lockB的锁
    } // 释放lockA的锁
}

public void dec(int m) {
    synchronized(lockB) { // 获得lockB的锁
        this.another -= m;
        synchronized(lockA) { // 获得lockA的锁
            this.value -= m;
        } // 释放lockA的锁
    } // 释放lockB的锁
}
```

两个线程各自持有不同的锁，然后各自试图获取对方手里的锁，造成了双方无限等待下去，这就是死锁。

死锁发生后，没有任何机制能解除死锁，**只能强制结束JVM进程**。

那么我们应该如何避免死锁呢？答案是：**线程获取锁的顺序要一致**。即严格按照先获取`lockA`，再获取`lockB`的顺序

## 可重入锁(ReentrantLock)

JVM允许**同一个线程**重复获取同一个锁，这种能被同一个线程反复获取的锁，就叫做**可重入锁**。

**`ReentrantLock`保证了只有一个线程可以执行临界区代码**

由于Java的线程锁是可重入锁，所以，获取锁的时候，不但要判断是否是第一次获取，还要记录这是第几次获取。每获取一次锁，记录+1，每退出`synchronized`块，记录-1，**减到0**的时候，才会**真正释放锁**。

```java
public class Counter {
    private final Lock lock = new ReentrantLock();
    private int count;

    public void add(int n) {
        lock.lock();
        try {
            count += n;
        } finally {
            lock.unlock();
        }
    }
}
```

因为`synchronized`是Java语言层面提供的语法，所以我们不需要考虑异常，而`ReentrantLock`是Java代码实现的锁，我们就必须先获取锁，然后在`finally`中正确释放锁。

`ReentrantLock`可以尝试获取锁

```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        ...
    } finally {
        lock.unlock();
    }
}
```

所以，使用`ReentrantLock`比直接使用`synchronized`更安全，线程在`tryLock()`失败的时候不会导致死锁

## ReentrantLock和Condition

用`ReentrantLock`我们怎么编写`wait`和`notify`的功能呢？答案是使用`Condition`对象来实现`wait`和`notify`的功能。

Condition类是基于Lock的，它的实例必须绑定一个Lock对象。它提供的await，signal,  signalAll跟wait。notify，notifyAll是一样的作用。其中await可以像RenentrantLock的tryLock()方法一样指定等待时间，如果超过指定的等待时间后，该线程会自动唤醒。

```java
class TaskQueue {
    private final Lock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    private Queue<String> queue = new LinkedList<>();

    public void addTask(String s) {
        lock.lock();
        try {
            queue.add(s);
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public String getTask() {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                condition.await();
            }
            return queue.remove();
        } finally {
            lock.unlock();
        }
    }
}
```

## ReadWriteLock

上面的ReentranLock或者synchronized机制都是保证临界区代码只能有一个线程执行。但是针对一些多数线程只读，少数线程写的情况，这种对临界资源的保护就过头了。造成效率低下（哪怕是一个读的线程都需要获得锁）。我们更想要的是：在所有线程都是读取操作的时候，我们允许多个线程同时读取临界资源，但是哪怕只有一个线程在写，我们必须只允许写的线程执行临界区代码。

而ReadWriteLock就是来解决这个问题的，它保证：

- 只允许一个线程写入（其他线程既不能写入也不能读取）；
- 没有写入时，多个线程允许同时读（提高性能）。

```java
public class Counter {
    private final ReadWriteLock rwlock = new ReentrantReadWriteLock();
    private final Lock rlock = rwlock.readLock();
    private final Lock wlock = rwlock.writeLock();
    private int[] counts = new int[10];

    public void inc(int index) {
        wlock.lock(); // 加写锁
        try {
            counts[index] += 1;
        } finally {
            wlock.unlock(); // 释放写锁
        }
    }

    public int[] get() {
        rlock.lock(); // 加读锁
        try {
            return Arrays.copyOf(counts, counts.length);
        } finally {
            rlock.unlock(); // 释放读锁
        }
    }
}
```

读锁和写锁必须从ReadWriteLock对象获取

把读写操作分别用读锁和写锁来加锁，在读取时，多个线程可以同时获得读锁，这样就大大提高了并发读的执行效率。

使用`ReadWriteLock`时，适用条件是同一个数据，有大量线程读取，但仅有少数线程修改。

例如，一个论坛的帖子，回复可以看做写入操作，它是不频繁的，但是，浏览可以看做读取操作，是非常频繁的，这种情况就可以使用`ReadWriteLock`。

## StampedLock

如果我们深入分析`ReadWriteLock`，会发现它有个潜在的问题：如果有线程正在读，写线程需要等待读线程释放锁后才能获取写锁，即读的过程中不允许写，这是一种悲观的读锁。

StampedLock: 读的过程中也允许获取写锁后写入！这样一来，我们读的数据就可能不一致，所以，需要一点额外的代码来判断读的过程中是否有写入，这种读锁是一种乐观锁。

乐观锁的意思就是**乐观地估计读的过程中大概率不会有写入**，因此被称为乐观锁。反过来，**悲观锁则是读的过程中拒绝有写入，也就是写入必须等待**。显然乐观锁的并发效率更高，但一旦有小概率的写入导致读取的数据不一致（脏读），需要能检测出来，再读一遍就行。

```
public class Point {
    private final StampedLock stampedLock = new StampedLock();

    private double x;
    private double y;

    public void move(double deltaX, double deltaY) {
        long stamp = stampedLock.writeLock(); // 获取写锁
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            stampedLock.unlockWrite(stamp); // 释放写锁
        }
    }

    public double distanceFromOrigin() {
        long stamp = stampedLock.tryOptimisticRead(); // 获得一个乐观读锁
        // 注意下面两行代码不是原子操作
        // 假设x,y = (100,200)
        double currentX = x;
        // 此处已读取到x=100，但x,y可能被写线程修改为(300,400)
        double currentY = y;
        // 此处已读取到y，如果没有写入，读取是正确的(100,200)
        // 如果有写入，读取是错误的(100,400)
        if (!stampedLock.validate(stamp)) { // 检查乐观读锁后是否有其他写锁发生
            stamp = stampedLock.readLock(); // 获取一个悲观读锁
            try {
                currentX = x;
                currentY = y;
            } finally {
                stampedLock.unlockRead(stamp); // 释放悲观读锁
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

和`ReadWriteLock`相比，写入的加锁是完全一样的，不同的是读取。注意到首先我们通过`tryOptimisticRead()`获取一个乐观读锁，**并返回版本号**。接着进行读取，读取完成后，我们通过`validate()`去验证版本号，如果在读取过程中没有写入，版本号不变，验证成功，我们就可以放心地继续后续操作。如果在读取过程中有写入，版本号会发生变化，验证将失败。在失败的时候，我们再通过获取**悲观读锁**再次读取。由于写入的概率不高，程序在绝大部分情况下可以通过乐观读锁获取数据，**极少数情况下使用悲观读锁获取数据**。

可见，`StampedLock`把读锁细分为乐观读和悲观读，能进一步提升并发效率。但这也是有代价的：一是代码更加复杂，二**是`StampedLock`是不可重入锁**，不能在一个线程中反复获取同一个锁。