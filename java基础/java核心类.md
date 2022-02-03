# String, StringBuilder, StringBuffer

在java中，对字符的操作非常频繁也非常普遍，以至于在历来的java版本中对字符串操作都有很多符号优化。其中我们在操作字符串时，String, StringBuilder, StringBuffer三个类是经常会用到的java核心类。我们可以来看下，为什么java对于字符串的操作会有这三个核心类。

## String

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence,
               Constable, ConstantDesc {

    /**
     * The value is used for character storage.
     *
     * @implNote This field is trusted by the VM, and is a subject to
     * constant folding if String instance is constant. Overwriting this
     * field after construction will cause problems.
     *
     * Additionally, it is marked with {@link Stable} to trust the contents
     * of the array. No other facility in JDK provides this functionality (yet).
     * {@link Stable} is safe here, because value is never null.
     */
    @Stable
    private final byte[] value;

    /**
     * The identifier of the encoding used to encode the bytes in
     * {@code value}. The supported values in this implementation are
     *
     * LATIN1
     * UTF16
     *
     * @implNote This field is trusted by the VM, and is a subject to
     * constant folding if String instance is constant. Overwriting this
     * field after construction will cause problems.
     */
    private final byte coder;

    /** Cache the hash code for the string */
    private int hash; // Default to 0

    /**
     * Cache if the hash has been calculated as actually being zero, enabling
     * us to avoid recalculating this.
     */
    private boolean hashIsZero; // Default to false;

    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    @java.io.Serial
    private static final long serialVersionUID = -6849794470754667710L;
```

1. 从上面的string源码来看，从开始的注释就说得很明确了，string类型的实例在创建之后就是不可改变的。这也在代码中的private final byte[] value的修饰符是final也说明了这个承载string字符数组的实例变量在构造方法的初始化后就不可更改。同时因为string是immutable的，所以也**是线程安全的（**实例创建后只可读，不可更改）。

2. java的complier对涉及到String的操作都做了优化。在代码中，任何字符串常量其实都是等价于String实例,例如：

   ```java
   String str = "abc";//等价于下面的写法
   
   char data[] = {'a', 'b', 'c'};
   
   String str = new String(data);
   ```

   对于字符串常量的+也是有优化，但是这个根据不同jvm有所不同，比如在hotspot上用的就是StringBuilder	来实现

```java
 sb += "java";//编译的时候， 等价于下面的写法

 StringBuilder sb = new StringBuilder(s);
 sb.append("java");
 s=sb.toString();
```

## StringBuilder

那有了String了，为什么还需要StringBuilder呢？我们先来看个例子

```java
public class Main {
         
    public static void main(String[] args) {
        String string = "";
        for(int i=0;i<10000;i++){
            string += "hello";
        }
    }
}
```

上述的代码中，就是频繁修改或者concoct字符串的场景。我们知道String不可变，所以上述场景在编译后会等效于以下代码

```java
public static void main(String[] args) {
    String string = "";
    for(int i=0;i<10000;i++){
       	 StringBuilder sb = new StringBuilder(string);
         sb.append("java");
         string=sb.toString();
    }
}
```
循环10000次就会在运行时的heap区，新建10000个实例，而且在这个循环后，这些实例都是垃圾，需要gc来清理的。所以这无疑就挤占了本就宝贵的heap区域，而且拖慢了gc的效率。

此时就是StringBuilder kicks in 了。我们还是先来看源码

```java
public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, Comparable<StringBuilder>, CharSequence
{

    /** use serialVersionUID for interoperability */
    @java.io.Serial
    static final long serialVersionUID = 4383685877147921099L;

    /**
     * Constructs a string builder with no characters in it and an
     * initial capacity of 16 characters.
     */
    @HotSpotIntrinsicCandidate
    public StringBuilder() {
        super(16);
    }

    /**
     * Constructs a string builder with no characters in it and an
     * initial capacity specified by the {@code capacity} argument.
     *
     * @param      capacity  the initial capacity.
     * @throws     NegativeArraySizeException  if the {@code capacity}
     *               argument is less than {@code 0}.
     */
    @HotSpotIntrinsicCandidate
    public StringBuilder(int capacity) {
        super(capacity);
    }
```

发现上述代码中并没有承载字符串的数组，其实是在它的父类中。

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    /**
     * The value is used for character storage.
     */
    byte[] value;

    /**
     * The id of the encoding used to encode the bytes in {@code value}.
     */
    byte coder;

    /**
     * The count is the number of characters used.
     */
    int count;

    private static final byte[] EMPTYVALUE = new byte[0];

    /**
     * This no-arg constructor is necessary for serialization of subclasses.
     */
    AbstractStringBuilder() {
        value = EMPTYVALUE;
    }

    /**
     * Creates an AbstractStringBuilder of the specified capacity.
     */
    AbstractStringBuilder(int capacity) {
        if (COMPACT_STRINGS) {
            value = new byte[capacity];
            coder = LATIN1;
        } else {
            value = StringUTF16.newBytesFor(capacity);
            coder = UTF16;
        }
    }
```

结合两个代码可以看出，StringBuilder底层采用的依旧是数组存储字符串，只不过此时的数组并没有像在String中的final修饰符。而且这里还多了一个capacity的变量，用来指定数组的容量，所以这是一个动态的数组。相当于预分配了字符串的容量，初始的容量是16，后面根据数组填充的大小来进行扩容。

```java
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = value.length >> coder;
        int newCapacity = (oldCapacity << 1) + 2;
        if (newCapacity - minCapacity < 0) {
            newCapacity = minCapacity;
        }
        int SAFE_BOUND = MAX_ARRAY_SIZE >> coder;
        return (newCapacity <= 0 || SAFE_BOUND - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity;
    }
```

但是正是由于StringBuilder是可变的，那么在多线程环境下，就会造成数据操作的不可预见性，也就是线程不安全。这时就是StringBuffer Kicks in了。

## StringBuffer

```java
 public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, Comparable<StringBuffer>, CharSequence
{

    /**
     * A cache of the last value returned by toString. Cleared
     * whenever the StringBuffer is modified.
     */
    private transient String toStringCache;

    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    @java.io.Serial
    static final long serialVersionUID = 3388685877147921107L;

    /**
     * Constructs a string buffer with no characters in it and an
     * initial capacity of 16 characters.
     */
    @HotSpotIntrinsicCandidate
    public StringBuffer() {
        super(16);
    }

    /**
     * Constructs a string buffer with no characters in it and
     * the specified initial capacity.
     *
     * @param      capacity  the initial capacity.
     * @throws     NegativeArraySizeException  if the {@code capacity}
     *             argument is less than {@code 0}.
     */
    @HotSpotIntrinsicCandidate
    public StringBuffer(int capacity) {
        super(capacity);
    }
```

上述代码中，我们不难看出, StringBuffer也是继承了AbstractStringBuilder这个抽象类，也就是说StringBuffer和StringBuilder的行为是类似的。

```java
    @HotSpotIntrinsicCandidate
    public synchronized StringBuffer append(int i) {
        toStringCache = null;
        super.append(i);
        return this;
    }

    /**
     * @since 1.5
     */
    @Override
    public synchronized StringBuffer appendCodePoint(int codePoint) {
        toStringCache = null;
        super.appendCodePoint(codePoint);
        return this;
    }

    @Override
    public synchronized StringBuffer append(long lng) {
        toStringCache = null;
        super.append(lng);
        return this;
    }

```

但是，不同于StringBuilder的线程不安全，StringBuffer是线程安全的，因为在数据操作的方法中都用**synchronized**来修饰。

## 小结

因此，这三个类是各有利弊，应当根据不同的情况来进行选择使用：　

当字符串相加操作或者改动较少的情况下，建议使用 String str="hello"这种形式；

当字符串相加操作较多的情况下，建议使用StringBuilder，如果采用了多线程，则使用StringBuffer。

## 常见试题考点

**以下内容引用自：http://www.cnblogs.com/dolphin0520/p/3778589.html**

1. 下面这段代码的输出结果是什么？

```java
String a = "hello2"; 　　
String b = "hello" + 2; 　　
System.out.println((a == b));
```

输出结果为：true。原因很简单，"hello"+2在编译期间就已经被优化成"hello2"，因此在运行期间，变量a和变量b指向的是同一个对象。

2. 下面这段代码的输出结果是什么？

```java
String a = "hello2"; 　 
String b = "hello";    
String c = b + 2;    
System.out.println((a == c));
```

输出结果为:false。由于有符号引用的存在，所以 String c = b + 2;不会在编译期间被优化，不会把b+2当做字面常量来处理的，因此这种方式生成的对象事实上是保存在堆上的。因此a和c指向的并不是同一个对象。javap -c得到的内容：

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220106181902531.png" alt="image-20220106181902531" style="zoom:50%;" />

3. 下面这段代码的输出结果是什么？

```java
String a = "hello2";  　 
final String b = "hello";    
String c = b + 2;    
System.out.println((a == c));
```

输出结果为：true。对于被final修饰的变量，会在class文件常量池中保存一个副本，对final变量的访问在编译期间都会直接被替代为真实的值（可以看成是宏）。那么String c = b + 2;在编译期间就会被优化成：String c = "hello" + 2; 下图是javap -c的内容：



<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220106184101323.png" alt="image-20220106184101323" style="zoom:50%;" />

4. 下面这段代码输出结果为：

```java
public class Main {
    public static void main(String[] args) {
        String a = "hello2";
        final String b = getHello();
        String c = b + 2;
        System.out.println((a == c));
    }
     
    public static String getHello() {
        return "hello";
    }
}
```

　　输出结果为false。这里面虽然将b用final修饰了，但是由于其赋值是通过方法调用返回的，**那么它的值只能在运行期间确定**，因此a和c指向的不是同一个对象。

5. 下面这段代码的输出结果是什么？

```java
public class Main {
    public static void main(String[] args) {
        String a = "hello";
        String b =  new String("hello");
        String c =  new String("hello");
        String d = b.intern();
         
        System.out.println(a==b);
        System.out.println(b==c);
        System.out.println(b==d);
        System.out.println(a==d);
    }
}
```

　　输出结果为（JDK版本 JDK6)：

　　![img](https://images0.cnblogs.com/i/288799/201406/091536144056475.jpg)

　　这里面涉及到的是String.intern方法的使用。在String类中，**intern方法是一个本地方法**，在JAVA SE6之前，intern方法会**在运行时常量池中查找是否存在内容相同的字符串，如果存在则返回指向该字符串的引用，如果不存在，则会将该字符串入池，并返回一个指向该字符串的引用**。因此，a和d指向的是同一个对象。

6. String str = new String("abc")创建了多少个对象？

　　这个问题在很多书籍上都有说到比如《Java程序员面试宝典》，包括很多国内大公司笔试面试题都会遇到，大部分网上流传的以及一些面试书籍上都说是2个对象，这种说法是片面的。

　　如果有不懂得地方可以参考这篇帖子：

　　http://rednaxelafx.iteye.com/blog/774673/

　　首先**必须弄清楚创建对象的含义**，创建是什么时候创建的？这段代码在运行期间会创建2个对象么？毫无疑问不可能，用javap -c反编译即可得到JVM执行的字节码内容：

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220106184738395.png" alt="image-20220106184738395" style="zoom:50%;" />　　

很显然，new只调用了一次，也就是说只创建了一个对象。

　　而这道题目让人混淆的地方就是这里，这段代码在运行期间确实只创建了一个对象，即在**堆上创建了"abc"对象**。而为什么大家都在说是2个对象呢，这里面要澄清一个概念该段代码执行过程和**类的加载过程**是有区别的。在类加载的过程中，确实**在运行时常量池中创建了一个"abc"对象**，而在代码执行过程中确实只创建了一个String对象。

　　因此，这个问题如果换成 String str = new String("abc")**涉及**到几个String对象？合理的解释是2个。

　　个人觉得在面试的时候如果遇到这个问题，可以向面试官询问清楚”是这段代码执行过程中创建了多少个对象还是涉及到多少个对象“再根据具体的来进行回答。

7. 下面这段代码1）和2）的区别是什么？

```java
public class Main {
    public static void main(String[] args) {
        String str1 = "I";
        //str1 += "love"+"java";        //1)
        str1 = str1+"love"+"java";      //2)
         
    }
}
```

　　1）的效率比2）的效率要高。1）中的"love"+"java"在编译期间会被优化成"lovejava"，而2）中的不会被优化。下面是两种方式的字节码：

　　1）的字节码：

　　	<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220106185323632.png" alt="image-20220106185323632" style="zoom:50%;" />

　　2）的字节码：

​			<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220106185619997.png" alt="image-20220106185619997" style="zoom:50%;" />

　　可以看出，在1）中只进行了一次append操作，而在2）中进行了两次append操作。