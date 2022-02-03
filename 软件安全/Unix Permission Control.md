#  Unix/Linux Permission Control

## • File permissions

在Linux中，一切皆是文件。所以理解文件的重要性不言而喻。我们先从最普通的shell command: ls -l说起

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211205172838880.png" alt="image-20211205172838880" style="zoom: 33%;" />

上图中-rwxrwxr-x 1 enjiang enjiang 92 oct 14 00:02 repeat.sh中，头部-代表是一个普通文件，-d代表directory,之后分别是owner, group user, other对此文件的操作权限。（著名的421原则）之后数字，如果是普通文件，代表Inode的链接数，如果是directory表示一级目录数。第一个enjiang是owner name, 第二个是group name，随后是last modified time。最后是文件名。下图可以更明了（from http://www.51gjie.com/linux/1015.html）

<img src="http://www.51gjie.com/images/2019/qtj4a1n2.i4m.jpg" alt="img" style="zoom:50%;" />

首先来看文件类型和权限值（记住directory在Linux里也是文件的一种）。对于文件各种用户的权限值的不同其实我们可以很轻松的理解（read, write, execute）。这里讲一下，权限值的左右就是：当这个文件被某一个process执行r,w,x操作时，系统会先去检查该process的effective user id(euid)【注：每个process都有6个id，是从该进程的调用者继承的】，然后根据euid去确定该进程是否有权限去操作文件，如果有就将这个文件作为system call的一个参数, invoke system call（因为所有的操作归根到底都是system call）， 没有就会false。

这里要需要注意的是，从上面的文件信息中看到文件有rwx操作，但是没有delete操作？肯定是有的，只不过不在这里。在该文件所属的directory里。directory的permission所代表的略有不同，因为directory也是一种文件，这个文件里包含的是directory entries， 每一个directory  entry is a **filename and corresponding inode**(where the real information about the file stored on disk. 我们可以用stat filename去查看文件的inode信息.

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211205175915913.png" alt="image-20211205175915913" style="zoom:50%;" />

对于directory， r grants permission to list directory contents, w grants permission to create and delete files, x grants permission to traverse the directory boundary（这个跟r权限不一样，意味着可以遍历也就是进入到这个目录（cd））,例如Apache上运行着一个server，然后用户访问这个server的前提是用户process对这个server 目录有x的权限（可以不要r的权限）

**以上说明的是**：

1. 如果一个用户对这个directory有w的权限，该用户就可以删除、创建该目录下的文件。而与文件的owner，group or permission settings 是无关的
2. 如果一个用户对这个directory有x的权限(无r)，该用户如果知道directory下文件的名字，是可以read或execute该文件的
3. 一般对home directory 的权限设置是rwx-----x,对owner full permission,对group没有任何权限，对other只有x权限。这样设置是为了：web server (e.g. Apache) can see inside your directory and serve files from there to the outside world
4. 如果我们想要open 任何files（no matter open for reading, writing or executing），我们必须有这个文件路径中每个directory的x权限，并且拥有这个文件的正确权限 

**另外一点需要说明的是，Unix/Linux 中除开一切皆是file之外，还有user id model也是个很重要的概念。**

Unix/Linux系统中，其实跟id相关的有process, user, file。其中user 有一个unique 的UID，一个 main GID和多个 supplementary GID；process 有3个UID(EUID, RUID, SUID), 3 GID(EGID, RGID, SGID); file 有一个UID和GID

其中关于process的6个id其实是各有作用的，比如对于3个UID来讲。

1. EUID：effective user id定义了access rights(当process去访问资源时，kernel进行权限检查时就是匹配的EUID)
2. RUID: real user id定义了这个process的owner，也就是这个进程调用者的UID
3. SUID: saved user id是保存RUID，目的是当发生privilege 改变for later restoration。

而3个GUID的用法跟3个UID是类似的。

#### process 的3个UID是如何设置的？

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211206111935660.png" alt="image-20211206111935660" style="zoom:50%;" />

#### File的UID和GID是如何设置的？

当一个file被创建时，如果3 additional bits are disabled。UID 是直接inherit parent process's EUID, GID是直接inherit from parent process's EGID. 如果 3 additional bits中有enabled的, 看下一小节就知道了。

#### File被创建时，file的default permission bits是如何设置的呢？

notice:The system default permission values are 777 (`rwxrwxrwx`) for folders and 666 (`rw-rw-rw-`) for files.

Linux中有一个每个进程都有一个相关的file creation mask, 简称umask。该umask继承自parent process或者create user。一般为0022，0027. 当一个file被创建时，permission bits = default permission - umask。所以从这里可以看出umask的作用是specifies the permissions that should be disabled on any new files. 

在umask中有4bits，那多出来的1bit是什么呢？这其实是3 additional bits，分别是the sticky bit, the set user ID bit 和the set group ID bit. 他们的作用分别是：

1. The sticky bit: 对于executable meant a copy of its text section was kept in swap after the program terminated so it would start faster next time（但是已经弃用）；对于directory means that in order to delete files from it we need to both have write permission on the directory and the the file's owner(比如**对于/tmp directory**，任何人都可以在这个directory下create, write, read任何files, 但是我们不希望任何人都可以删除里面的file,我们希望只有文件的owner才能delete它。但是传统directory permission bits又不能到达这个效果，所以我们可以set /tmp 的sticky bit). 

   **The sticky bit's octal value is 1**, 我们也可以通过chmod +/-t dirname来设置sticky bit的值. 对文件的话一般没什么用。

   ![image-20211209154505367](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211209154505367.png)

2. The set group ID bit: 对于directory，如果set the set group ID bit on, 那么新的directory在创建时将不会inherit group ID from parent process but from the parent directory（**特别是在设置group shared directory***); 对于non-executable, it enables mandatory record locking; 对于executable而言跟the set user ID bit效果类似。

   **The set group ID's octal value is 2.** 也可以通过chmod g+/-s dirname/filename

   ![image-20211209154617571](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211209154617571.png)

3. The set user ID bit: 对于executable而言，如果the set user ID bit is enabled，这个executable program将不会inherit the EUID from the parent process而是直接复制这个executable's owner的UID来作为自己的UID。（Unix通过the UID-setting system calls, which allow a process to raise and drop privileges）。**最典型的应用就是*passwd 程序.***

   **The setuid's octal value is 4.** 也可一用chmod u+s  filename来修改

   ![image-20211209154655109](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211209154655109.png)

   我们希望除开root没有任何人能够看到/etc/shadow，但是我们又希望每个user能够修改自己在/etc/shadow中存储的密码。根据现有的设置/etc directory 的permission bit并不能达到这种相互矛盾的privilege 要求，那 passwd executable是如何实现的呢？就是通过The set user ID, 当passwd被调用时，它不会像一般没有设置the set user ID bit那样去inherit EUID from parent process，而是直接继承自passwd的owner的UID，所以当用户运行passwd时，kernel检测到passwd这个process的UID是0，所以自然就可以修改了。所以passwd 这一类executable file也叫set user ID program.

### • How to set 3 additional bits in Linux?

https://www.liquidweb.com/kb/how-do-i-set-up-setuid-setgid-and-sticky-bits-on-linux/

## • Access Process

在Unix系统中当一个process要去access any resource 时，kernel会通过一些规则检查是否grant这个access

1. 如果process的EUID是0，grant access
2. 如果process的EUID跟这个resource的Owner的uid一致，而且相关操作的permission bit对owner是set on的，grant access, 否则deny access
3. 如果process的EGID或者supplementary GroupID跟这个resource的Group id一致，而且相关操作的permission bit对group user是set on的, grant access, 否则deny access（以为一个进程可以继承user的group id,而user的GID一个main group和多个supplementary group）
4. 如果相关操作的permission bit对other user是set on的, grant access, 否则deny access

## *•* Set User ID programming and privilege levels

![](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211206112508571.png)

![image-20211206113421988](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211206113421988.png)

![image-20211206113539078](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211206113539078.png)

## *•* Time-of-check, time-of-use (TOCTOU) issues

In [software development](https://en.wikipedia.org/wiki/Software_development), **time-of-check to time-of-use** (**TOCTOU**, **TOCTTOU** or **TOC/TOU**) is a class of [software bugs](https://en.wikipedia.org/wiki/Software_bug) caused by a [race condition](https://en.wikipedia.org/wiki/Race_condition) involving the *checking* of the state of a part of a system (such as a security credential) and the *use* of the results of that check.

example:

```c
if (access("file", W_OK) != 0) {
    exit(1);
}

fd = open("file", O_WRONLY);
write(fd, buffer, sizeof(buffer));
```

Here, *access* is intended to check whether the real user who executed the `setuid` program would normally be allowed to write the file (i.e., `*access*` checks the [real userid](https://en.wikipedia.org/wiki/Real_userid) rather than [effective userid](https://en.wikipedia.org/wiki/Effective_userid)).

This race condition is vulnerable to an attack:

```c
// 
//
// After the access check
symlink("/etc/passwd", "file");
// Before the open, "file" points to the password database
//
//
```

Fix solution:

![image-20211206130348732](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20211206130348732.png)

## *•* Hard links, soft links, inodes, directories, directory entries