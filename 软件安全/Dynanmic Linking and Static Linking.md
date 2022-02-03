# Dynanmic Linking and Static Linking

因为在程序开发中，因为在开发中我们不去手动的实现所有的系统调用，我们要用到其他的的函数库。于是在把源程序编译链接成可执行程序时就涉及到了两种链接方式。这里的链接指的是：将源文件所需要的库函数映射到可执行程序中。传统的链接方式有静态链接和动态链接(SL,DL)

## Static Linking

静态链接指的是在链接这个过程发生在程序运行前。例如下图（引自：https://blog.csdn.net/kang___xi/article/details/80210717），比如在hello.c中我们引用了printf()函数，printf()其实是通过头文件<stdlib.h>引入的，而printf()相对应的printf.o其实是在目标程序libc.a中的（多个.o文件打包成了.a文件,详见：https://stackoverflow.com/questions/654713/o-files-vs-a-files/654757）。而在我们的hello.c编译成hello.o后，此时就需要进入链接阶段。linker此时会将hello.o和libc.a以及其他所需要的.a作为输入，然后linker的作用就是将这些输入链接然后输出可执行文件，此时的可执行文件hello中就可以直接执行了。

以上就是静态链接的过程。设想下我们基本大多数程序都会引入stdlib.h或者类似的标准库，那么如果在内存中运行多个程序，内存中就会有多个stdlib.h的copy,这样会造成内存的浪费；其次是我们如果想更新一个大范围被引用的库时，由于采用的是静态链接，所以毫无疑问我们需要对所有引用了这个更新库的程序进行重新编译、链接会造成更新的不方便。

<img src="https://img-blog.csdn.net/20180505235327609" alt="img" style="zoom: 67%;" />

![Static vs Dynamic Linking - Stack Overflow](https://i.stack.imgur.com/85V5p.png)

## Dynamic Linking

动态链接指的是链接这个过程发生在程序运行时（其实这个运行时指的是程序在加载到内存的时候），还是以hello.o为例子，当我们采用动态链接时，在生成.o文件后，hello.o就可以被加载到内存中，但是此时只是可执行文件，这时linker会动态的将hello.o所需要的库(linux中.so，windows中是.dll)加载到内存中，然后linker将这些库在内存的地址映射到hello.o的进程虚拟地址空间中，随之生成执行程序,然后就可以执行了。如果此时有一个foo.o也采用了动态链接并且加载到内存中准备执行，那么foo.o所需要c标准库之类的已经被加载到内存的就不会被重复加载了，而是通过linker直接将这些库的地址映射到foo.o的进程虚拟地址空间中，然后就生成执行程序开始执行。但是并不是说在采用动态链接的时候linker在程序运行前就不工作，只是此时linker：我象征性的记一下笔记哈，这个.o文件需要的其他relocatable object是哪些。

<img src="https://i.stack.imgur.com/XVVKu.jpg" alt="img" style="zoom:150%;" />





