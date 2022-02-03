# Java 异常处理机制

**一个健壮的程序必须处理各种各样的错误**，这些错误可能是来自用户的输入，外界资源的突然中断，，当出现这些错误时，不能总是就终止程序的运行。所谓的错误就是**程序在执行某个方法时，如果失败了，就表示出错**。java中的异常机制大部分是来自于C++的，调用方如何知道调用某个函数的时候失败了呢？

一般来说有两种办法

1. **根据返回值**

   这种实现方式在C语言的底层函数应用得非常广泛，调用方和被调用方实现约定好错误码代表的含义，还有就是在我们http请求代码中也有大量的应用。

   但是这种实现方式，处理错误代码就会非常麻烦，因为错误代码的约定很多，一个个来处理的话，会造成代码的臃肿和冗余。

2. **在语言层次上提供一个异常处理机制**，java中实现了这种异常处理机制，**异常本身也是一种class**，满足class的特性。异常可以在任何地方抛出， 我们只需要在调用的地方或者调用地方的上层进行捕获就行了。

例如：

```java
try {
    String s = processFile(“C:\\test.txt”);
    // ok:
} catch (FileNotFoundException e) {
    // file not found:
} catch (SecurityException e) {
    // no read permission:
} catch (IOException e) {
    // io error:
} catch (Exception e) {
    // other error:
}
```

因为java的异常是class，它的继承关系

```java
                     ┌───────────┐
                     │  Object   │
                     └───────────┘
                           ▲
                           │
                     ┌───────────┐
                     │ Throwable │
                     └───────────┘
                           ▲
                 ┌─────────┴─────────┐
                 │                   │
           ┌───────────┐       ┌───────────┐
           │   Error   │       │ Exception │
           └───────────┘       └───────────┘
                 ▲                   ▲
         ┌───────┘              ┌────┴──────────┐
         │                      │               │
┌─────────────────┐    ┌─────────────────┐┌───────────┐
│OutOfMemoryError │... │RuntimeException ││IOException│...
└─────────────────┘    └─────────────────┘└───────────┘
                                ▲
                    ┌───────────┴─────────────┐
                    │                         │
         ┌─────────────────────┐ ┌─────────────────────────┐
         │NullPointerException │ │IllegalArgumentException │...
         └─────────────────────┘ └─────────────────────────┘
```

从上面的继承关系可知。所有exception和error的父类是Throwable。error一般表示程序对此无能为力，比如：内存耗尽（OutOfMemoryError）,NoClassDefFounderError, StackOverflow.通常是由JVM抛出

而exception是运行时的异常，可以被程序捕获并且处理。而且某些异常是应用程序的一部分，应该捕获处理。例如：

- `NumberFormatException`：数值类型的格式错误
- `FileNotFoundException`：未找到文件
- `SocketException`：读取网络失败

还有一些异常是由于**程序编写的漏洞**造成的，所以应该根据这些异常修复程序本身。例如：

- `NullPointerException`：对某个`null`的对象调用方法或字段
- `IndexOutOfBoundsException`：数组索引越界

exception一般又分为两大类：

1. `RuntimeException`以及它的子类；
2. 非`RuntimeException`（包括`IOException`、`ReflectiveOperationException`等等）

Java规定：

​	必须捕获的异常，包括exception的子类，但是不包括runtimeException及其其子类。必须捕获的异常叫**checked exception**。**如果我们不捕获，会出现编译失败的问题**，**只要是方法声明可能会抛出Checked Exception，不在调用层捕获，也必须在更高的调用层捕获。所有未捕获的异常，最终也必须在`main()`方法中捕获，不会出现漏写`try`的情况。这是由编译器保证的。`main()`方法也是最后捕获`Exception`的机会。**

​	不需要捕获的异常，包括Error及其子类，runtimeException及其子类

**注意**：**编译器对RuntimeException及其子类不做强制捕获要求，不是指应用程序本身不应该捕获并处理RuntimeException。是否需要捕获，具体问题具体分析。**

**捕获后不处理的方式是非常不好的**，即使真的什么也做不了，也要先把异常记录下来：所有异常都可以调用`printStackTrace()`方法打印异常栈，这是一个简单有用的快速打印异常的方法。

#### 捕获方式

在Java中，凡是可能抛出异常的语句，都可以用`try ... catch`捕获。把可能发生异常的语句放在`try { ... }`中，然后使用`catch`捕获对应的`Exception`及其子类。

JVM在捕获到异常后，会从上到下匹配`catch`语句，匹配到某个`catch`后，执行`catch`代码块，然后**不再**继续匹配。简单地说就是：**多个`catch`语句只有一个能被执行**

存在多个`catch`的时候，`catch`的顺序非常重要：**子类必须写在前面**

```java
public static void main(String[] args) {
    try {
        process1();
        process2();
        process3();
    } catch (IOException e) {
        System.out.println("IO error");
    } catch (UnsupportedEncodingException e) { // 永远捕获不到
        System.out.println("Bad encoding");
    }
}
```

**无论是否有异常发生，如果我们都希望执行一些语句**，例如清理工作，怎么写？

可以把执行语句写若干遍：正常执行的放到`try`中，每个`catch`再写一遍.

Java的`try ... catch`机制还提供了`finally`语句，**`finally`语句块保证有无错误都会执行。**

某些情况下，可以没有`catch`，只使用`try ... finally`结构

```java
void process(String file) throws IOException {
    try {
        ...
    } finally {
        System.out.println("END");
    }
}
```

#### 捕获多种异常

如果某些异常的处理逻辑相同，但是异常本身不存在继承关系.

```java
public static void main(String[] args) {
    try {
        process1();
        process2();
        process3();
    } catch (IOException | NumberFormatException e) { // IOException或NumberFormatException
        System.out.println("Bad input");
    } catch (Exception e) {
        System.out.println("Unknown error");
    }
}
或者
public static void main(String[] args) {
    try {
        process1();
        process2();
        process3();
    } catch (IOException e) {
        System.out.println("Bad input");
    } catch (NumberFormatException e) {
        System.out.println("Bad input");
    } catch (Exception e) {
        System.out.println("Unknown error");
    }

```

#### 抛出异常

抛出异常分两步：

1. 创建某个`Exception`的实例；

2. 用`throw`语句抛出。

   ```java
   void process2(String s) {
       if (s==null) {
           throw new NullPointerException();
       }
   }
   ```

如果一个方法捕获了某个异常后，又在`catch`子句中抛出新的异常，就相当于把抛出的异常类型“转换”了。

```java
void process1(String s) {
    try {
        process2();
    } catch (NullPointerException e) {
        throw new IllegalArgumentException();
    }
}

void process2(String s) {
    if (s==null) {
        throw new NullPointerException();
    }
}
```

但是像上面这样写的话会**导致原始异常丢失**，所以为了能追踪到完整的异常栈信息，我们需要将原始的exception实例传进去，新的exception持有原始exception信息。

```java
public class Main {
    public static void main(String[] args) {
        try {
            process1();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    static void process1() {
        try {
            process2();
        } catch (NullPointerException e) {
            throw new IllegalArgumentException(e);
        }
    }

    static void process2() {
        throw new NullPointerException();
    }
}

```

```
java.lang.IllegalArgumentException: java.lang.NullPointerException
    at Main.process1(Main.java:15)
    at Main.main(Main.java:5)
Caused by: java.lang.NullPointerException
    at Main.process2(Main.java:20)
    at Main.process1(Main.java:13)
```

注意到`Caused by: Xxx`，说明捕获的`IllegalArgumentException`并不是造成问题的根源，根源在于`NullPointerException`，是在`Main.process2()`方法抛出的。

**在代码中获取原始异常可以使用`Throwable.getCause()`方法。如果返回`null`，说明已经是“根异常”了。**

有了完整的异常栈的信息，我们才能快速定位并修复代码的问题。

**捕获到异常并再次抛出时，一定要留住原始异常，否则很难定位第一案发现场**

**在`catch`中抛出异常，不会影响`finally`的执行。JVM会先执行`finally`，然后抛出异常。**

#### 异常屏蔽

如果在执行`finally`语句时抛出异常，那么，`catch`语句的异常还能否继续抛出？

不会，此时catch语句的异常会被屏蔽，因为只能抛出一个异常。但是可以通过调用`Throwable.addSuppressed()`，把原始异常添加进来，最后在`finally`抛出

```java
public class Main {
    public static void main(String[] args) throws Exception {
        Exception origin = null;
        try {
            System.out.println(Integer.parseInt("abc"));
        } catch (Exception e) {
            origin = e;
            throw e;
        } finally {
            Exception e = new IllegalArgumentException();
            if (origin != null) {
                e.addSuppressed(origin);
            }
            throw e;
        }
    }
}
```

#### 自定义异常：

**抛出异常时，尽量复用JDK已定义的异常类型；**

Java标准库定义的常用异常包括：

```java
Exception
│
├─ RuntimeException
│  │
│  ├─ NullPointerException
│  │
│  ├─ IndexOutOfBoundsException
│  │
│  ├─ SecurityException
│  │
│  └─ IllegalArgumentException
│     │
│     └─ NumberFormatException
│
├─ IOException
│  │
│  ├─ UnsupportedCharsetException
│  │
│  ├─ FileNotFoundException
│  │
│  └─ SocketException
│
├─ ParseException
│
├─ GeneralSecurityException
│
├─ SQLException
│
└─ TimeoutException
```

一个常见的做法是**自定义一个`BaseException`作为“根异常”**，然后，派生出各种业务类型的异常。**`BaseException`需要从一个适合的`Exception`派生，通常建议从`RuntimeException`派生**，自定义的`BaseException`应该**提供多个构造方法**

```
public class BaseException extends RuntimeException {
    public BaseException() {
        super();
    }

    public BaseException(String message, Throwable cause) {
        super(message, cause);
    }

    public BaseException(String message) {
        super(message);
    }

    public BaseException(Throwable cause) {
        super(cause);
    }
}
```

好的编码习惯可以极大地降低`NullPointerException`的产生，成员变量在定义时初始化,使用空字符串`""`而不是默认的`null`可避免很多`NullPointerException`，编写业务逻辑时，用空字符串`""`表示未填写比`null`安全得多。**避免空指针,最好的方式还是做好非空校验**

#### 断言Assertion

断言失败时会抛出`AssertionError`，导致程序结束退出。因此，断言不能用于可恢复的程序错误，只应该用于开发和测试阶段。**对于可恢复的程序错误，不应该使用断言。应该抛出异常并在上层捕获**：

JVM默认关闭断言指令，即遇到`assert`语句就自动忽略了，不执行。

要执行`assert`语句，必须给Java虚拟机传递`-enableassertions`（可简写为`-ea`）参数启用断言。

#### 日志

##### JDK Logging

日志就是**Logging，它的目的是为了取代`System.out.println()`**。

输出日志，而不是用`System.out.println()`，有以下几个好处：

1. 可以设置输出样式，避免自己每次都写`"ERROR: " + var`；
2. 可以设置输出级别，禁止某些级别输出。例如，只输出错误日志；
3. 可以被重定向到文件，这样可以在程序运行结束后查看日志；
4. 可以按包名控制日志级别，只输出某些包打的日志；
5. 可以……

java标准库提供JDK Logging，一共分6个级别

- SEVERE
- WARNING
- INFO
- CONFIG
- FINE
- FINER
- FINEST

默认只输出info及以上级别的。但是缺点是**Logging系统在JVM启动时读取配置文件并完成初始化，一旦开始运行`main()`方法，就无法修改配置；**

配置不太方便，需要在JVM启动时传递参数`-Djava.util.logging.config.file=<config-file-name>`。

因此，**Java标准库内置的Logging使用并不是非常广泛**。

```java
// logging
import java.util.logging.Level;
import java.util.logging.Logger;

public class Hello {
    public static void main(String[] args) {
        Logger logger = Logger.getGlobal();
        logger.info("start process...");
        logger.warning("memory is running out...");
        logger.fine("ignored.");
        logger.severe("process will be terminated...");
    }
}
```

##### Commons Logging

和Java标准库提供的日志不同，**Commons Logging是一个第三方日志库**，它是由Apache创建的日志模块。Commons Logging的特色是，它可以**挂接不同的日志系统**，并通过配置文件指定挂接的日志系统。默认情况下，Commons Logging**自动搜索并使用Log4j**（Log4j是另一个流行的日志系统），如果没有找到Log4j，再使用JDK Logging。

```java
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

public class Main {
    public static void main(String[] args) {
        Log log = LogFactory.getLog(Main.class);
        log.info("start...");
        log.warn("end.");
    }
}

```

但是注意因为Commons Logging是一个第三方提供的库，所以，必须先把它[下载](https://commons.apache.org/proper/commons-logging/download_logging.cgi)下来。然后配置它才行

Commons Logging定义了6个日志级别：

- **FATAL**
- **ERROR**
- **WARNING**
- **INFO**
- **DEBUG**
- **TRACE**

默认级别是`INFO`

使用Commons Logging时，如果在静态方法中引用`Log`，通常直接定义一个静态类型变量：

```
// 在静态方法中引用Log:
public class Main {
    static final Log log = LogFactory.getLog(Main.class);

    static void foo() {
        log.info("foo");
    }
}
```

在实例方法中引用`Log`，通常定义一个实例变量：

```
// 在实例方法中引用Log:
public class Person {
    protected final Log log = LogFactory.getLog(getClass());

    void foo() {
        log.info("foo");
    }
}
```

注意到实例变量log的获取方式是`LogFactory.getLog(getClass())`，虽然也可以用`LogFactory.getLog(Person.class)`，**但是前一种方式有个非常大的好处，就是子类可以直接使用该`log`实例**。例如：

```
// 在子类中使用父类实例化的log:
public class Student extends Person {
    void bar() {
        log.info("bar");
    }
}
```

由于Java类的动态特性，**子类获取的`log`字段实际上相当于`LogFactory.getLog(Student.class)`，但却是从父类继承而来，并且无需改动代码。**

此外，Commons Logging的日志方法，例如`info()`，除了标准的`info(String)`外，还提供了一个非常有用的重载方法：`info(String, Throwable)`，这使得记录异常更加简单：

```java
try {
    ...
} catch (Exception e) {
    log.error("got exception!", e);
}
```

##### 使用Log4j

Commons Logging，可以作为“日志接口”来使用。而真正的“日志实现”可以使用Log4j,他们两个是一对好基友。Log4j的架构如下：

```ascii
log.info("User signed in.");
 │
 │   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
 ├──>│ Appender │───>│  Filter  │───>│  Layout  │───>│ Console  │
 │   └──────────┘    └──────────┘    └──────────┘    └──────────┘
 │
 │   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
 ├──>│ Appender │───>│  Filter  │───>│  Layout  │───>│   File   │
 │   └──────────┘    └──────────┘    └──────────┘    └──────────┘
 │
 │   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
 └──>│ Appender │───>│  Filter  │───>│  Layout  │───>│  Socket  │
     └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

使用Log4j输出一条日志时，Log4j自动**通过不同的Appender把同一条日志输出到不同的目的地**。例如：

- console：输出到屏幕；
- file：输出到文件；
- socket：通过网络输出到远程计算机；
- jdbc：输出到数据库

在输出日志的过程中，**通过Filter来过滤哪些log需要被输出**，哪些log不需要被输出。例如，仅输出`ERROR`级别的日志。

最后，**通过Layout来格式化日志信息**，例如，自动添加日期、时间、方法名称等信息。

上述结构虽然复杂，但我们在实际使用的时候，并不需要关心Log4j的API，而是通过配置文件来配置它。

以XML配置为例，使用Log4j的时候，我们把一个`log4j2.xml`的文件放到`classpath`下就可以让Log4j读取配置文件并按照我们的配置来输出日志。下面是一个配置文件的例子：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
	<Properties>
        <!-- 定义日志格式 -->
		<Property name="log.pattern">%d{MM-dd HH:mm:ss.SSS} [%t] %-5level %logger{36}%n%msg%n%n</Property>
        <!-- 定义文件名变量 -->
		<Property name="file.err.filename">log/err.log</Property>
		<Property name="file.err.pattern">log/err.%i.log.gz</Property>
	</Properties>
    <!-- 定义Appender，即目的地 -->
	<Appenders>
        <!-- 定义输出到屏幕 -->
		<Console name="console" target="SYSTEM_OUT">
            <!-- 日志格式引用上面定义的log.pattern -->
			<PatternLayout pattern="${log.pattern}" />
		</Console>
        <!-- 定义输出到文件,文件名引用上面定义的file.err.filename -->
		<RollingFile name="err" bufferedIO="true" fileName="${file.err.filename}" filePattern="${file.err.pattern}">
			<PatternLayout pattern="${log.pattern}" />
			<Policies>
                <!-- 根据文件大小自动切割日志 -->
				<SizeBasedTriggeringPolicy size="1 MB" />
			</Policies>
            <!-- 保留最近10份 -->
			<DefaultRolloverStrategy max="10" />
		</RollingFile>
	</Appenders>
	<Loggers>
		<Root level="info">
            <!-- 对info级别的日志，输出到console -->
			<AppenderRef ref="console" level="info" />
            <!-- 对error级别的日志，输出到err，即上面定义的RollingFile -->
			<AppenderRef ref="err" level="error" />
		</Root>
	</Loggers>
</Configuration>
```

有了配置文件还不够，因为Log4j也是一个第三方库，我们需要从[这里](https://logging.apache.org/log4j/2.x/download.html)下载Log4j，解压后，把以下3个jar包放到`classpath`中：

- log4j-api-2.x.jar
- log4j-core-2.x.jar
- log4j-jcl-2.x.jar

因为Commons Logging会自动发现并使用Log4j，所以，把上一节下载的`commons-logging-1.2.jar`也放到`classpath`中。

要打印日志，只需要按Commons Logging的写法写，不需要改动任何代码，就可以得到Log4j的日志输出，类似：

在开发阶段，**始终使用Commons Logging接口来写入日志，并且开发阶段无需引入Log4j**。**如果需要把日志写入文件， 只需要把正确的配置文件和Log4j相关的jar包放入`classpath`**，就可以自动把日志切换成使用Log4j写入，无需修改任何代码。

