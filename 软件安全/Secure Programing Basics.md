# CA647 Secure Programing Basics

## Process Address Space (32 bit PAS)

![img](https://i.stack.imgur.com/p803C.png)

|                Name                | started address |
| :--------------------------------: | :-------------: |
|               Kernel               |   0xFFFFFFFF    |
|            Environment             |   0xBFFFFFFF    |
|      Command line parameters       |                 |
|               Stack                |                 |
|                Heap                |                 |
| DLLs(Dynamically Linked Libraries) |   0x40000000    |
|                Data                |                 |
|                Text                |   0x00000000    |

## Multiple Threads Process

| Kernel  | Shared                          |
| ------- | ------------------------------- |
| stack 1 | Private (Thread 1)              |
| stack 2 | Private (Thread 2)              |
| Heap    | Shared                          |
| Data    | Shared (Race condition happens) |
| Text    | Shared                          |

## Process Life Cycle

![image-20211203150352861](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203150352861.png)

CPU ->WAITING: wait some low speed resource

WAITING -> RUNNING: Some event signals 

CPU -> RUNNING: time slice goes off or high priority process jumps in.

CPU -> STOPPED: debug

CPU -> ZOMBIE: program returned or exit and enters into ZOMBIE state wait to be cleaned by parent process.

## Virtual Memory

![image-20211203163230224](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203163230224.png)

In 32 bits machine. Pages: 4 kb, page table:1kb, page directory 1kb. (1024 * 1024 * 4k =  4GB)

## Address Translation

In RAM, translate logical address into physical address

![image-20211203164006869](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203164006869.png)d: 10 bits, p: 10 bits, o: 12 bits = 4 kb(size of the page frame)

Cache: the translation process of looking up these tables takes time, so to speed up, the VM instead of doing the process every time, the cache will store recent translation entry's mapping of (d, p, f). Cache is much more faster than RAM.

## System Call

![image-20211203164955281](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203164955281.png)

0x80: int(software interrupt)

eax:  responsible for storing system call number(cause when CPU is in kernel mode, it can't get access to the user's process address space, so we have to store arugs and system call number in the registers).

After invoke software interrupt(0x80), cpu will switch to kernel mode, and the system call handler will look up the system call number table by the value in eax, and then invoke system call.

The system call number is the index in the table where stores the address of system call code implementation

## Stack

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203172511145.png" alt="image-20211203172511145" style="zoom:50%;" />

esp: always point to the currently active stack frame(stack's top frame)

push dest: push the dest onto the stack top, and then esp - 4

pop dest: pop the top of  stack into dest, and then esp + 4

### epilog and prolog

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203172946578.png" alt="image-20211203172946578" style="zoom:50%;" />

Inside every function includes:

prolog: push %ebp and mov %esb, %ebp, these two assemble instructions call epilog. They mean first push the current ebp onto the stack(save the caller's frame base pointer), and then second move current esp to the ebp register(means init the current callee's frame base pointer).

epilog: leave(mov %ebp, %esp; pop %ebp）, ret, These two steps mean restore caller's ebp and then return to the place where saved eip points to.

the side effect of call instruction: push the address of following instruction onto stack(save return address: saved eip)

###### call procedure

![image-20211203175759845](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203175759845.png)

##### registers table

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203174359170.png" alt="image-20211203174359170" style="zoom:50%;" />

- <img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211203175415050.png" alt="image-20211203175415050" style="zoom: 33%;" />