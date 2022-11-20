# MIT6.S081网课学习笔记

# Lecture1 概述和举例

## 1.1 一些OS的基本概念

+ 硬件抽象 Abstract

  抽象硬件的使用，比如进程、文件系统等。高层级开发只需学会调用底层接口即可，无需关注硬件层级的指令等细节。

+ 硬件复用 Multiplex

  即多个进程底层通过操作系统，能够利用相同的硬件资源（CPU、内存等）

+ 隔离性 Isolation

  不同进程之间互不影响，一个进程bug一般只作用于当前进程。

+ 共享 Sharing

  不同进程可能需要操作相同文件/内存等资源，达到数据共享/传输的目的

+ 权限/安全 Security

  有访问控制机制

+ 良好的性能 Performance

  操作系统不应该成为开发的瓶颈，甚至应该能促进开发，提高应用程序性能

+ 泛用性强 Range of uses

  一个泛用的操作系统，需要能支持多种类型的应用运行

## 1.2 用户空间和内核空间 User/Kernel

​	用户在**用户空间(User)**内运行用户进程（Shell、docker等），而**内核空间(Kernel)**的进程辅助管理用户空间的进程，提供一系列数据结构/接口，以支持硬件资源的抽象访问(文件系统、进程管理、内核复用、内存分配等)。

内核Kernel指责包括且不限于：

+ Processes 进程管理
+ Mem Alloc 内存分配/管理
+ Access CTL 访问控制(用户进程请求访问硬件资源:磁盘、网卡等)

---

​	应用程序通过系统调用(system call)访问内核，看似函数调用，实际上跳转到内核kernel，在内核中完成系统调用。

举例：

+ 通过open系统调用，获取指定的文件描述符`fd = open("out", 1)`
+ 通过write系统调用，向指定的文件描述符写入数据`write(fd, "hello\n", 6)`
+ 通过fork系统调用，创建新进程，返回进程标识符pid，`pid=fork()`

---

系统调用调入内核，和标准的函数调用跳到另一个函数相比，有何区别？

**内核(Kernel)是一段始终驻留的代码，其在机器运行时加载，拥有特殊权限，可以直接访问各种硬件。而普通的程序连磁盘都不能直接访问**。普通的函数调用一般没有任何与硬件相关的特权，而系统调用跳转到内核后，内核中的系统调用实现能够访问各种受保护的、敏感的硬件资源（比如直接访问硬盘）。

## 1.3 系统调用-举例

系统调用是操作系统用于提供服务的接口。

### 1.3.1 copy

下面举例一个简单的系统调用代码`copy.c`

```c
// copy.c: copy input to output

#include "kernel/types.h"
#include "user/user.h"

int main()
{
  char buf[64];
  while(1) {
    // 课程用的操作系统，遵循Unix惯例，0作为 console的输入文件描述符; buf 是指向某个内存的指针，程序可以通过指针读取对应地址的内存中的数据; 第三个参数是代码想要读取的最大长度,这里表示最多读取64字节的数据.
    // 这里如果第三个参数写65，虽然能通过，但是会导致内存中buf预期的64字节后的1个字节被写入脏数据，可能导致其他进程出现异常
    int n = read(0, buf, sizeof(buf));
    // 读到文件末尾返回0; 出现其他错误如文件描述符不存在等返回-1
    if(n <= 0)
      break;
    // 遵循Unix惯例，1作为console的输出文件描述符
    write(1, buf, n);
  }
  exit(0);
}
```

### 1.3.2 open

`open.c`

```c
// open.c: create a file, write to it.

#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

int main()
{
  // 系统调用，以"output.txt"作为文件名创建文件，第二个参数 O_ 开头参数告知内核实现的系统调用，我们需要用该文件名创建文件并执行写入操作。open返回一个新分配的文件描述符。
  int fd = open("output.txt", O_WRONLY | O_CREATE);
  // 传递文件描述符，需要写入的数据对应的指针，写入的字写数
  write(fd, "ooo\n", 4);
  exit(0);
}
```

实际上**，操作系统为每个进程独立维护一套索引到内核中的文件描述符表**，该表维护每个进程的状态。

不同的进程运行时，即使打开不同的文件也可能得到相同的文件描述符编号。**相同的文件描述符，在不同进程内可能对应不同的实际文件**。

---

提问：编译器编译代码后，是编译后的代码段中包含了操作系统的系统调用指令吗？

上述的举例代码经过编译后，实际对应的汇编代码中，包含对操作系统的系统调用指令的调用，其会跳转到内核在执行，内核通过检查当前进程的内存和寄存器等获取系统调用的参数，从而完成系统调用。

### 1.3.3 fork

fork创建一个新的进程，如下`fork.c`

fork拷贝当前进程的内存，并创建一个新的进程。fork系统调用在两个进程中都返回，

+ **原始进程中fork系统调用返回新进程id（> 0）**
+ **新创建的进程中fork系统调用返回0**

```c
// fork.c: create a new process

#include "kernel/types.h"
#include "user/user.h"

int main()
{
  int pid;
  
  pid = fork();
  
  printf("fork() returned %d\n", pid);
  
  if(pid == 0) {
    printf("child\n");
  } else {
    printf("parent\n");
  }
  
  exit(0);
}
```

### 1.3.4 exec

执行exec系统调用的代码举例，`exec.c`

exec从指定的特定文件中读取指令，并替换调用进程。即从文件中加载指令，覆盖当前进程，丢弃当前内存，然后开始执行这些指令。

```c
// exec.c: replace a process with an executable file

#include "kernel/types.h"
#include "user/user.h"
 
int main() 
{
  // 0 即空指针，因为C语言没有高级的数组语法，这告知内核，这个数组到0(空指针)即结束
  char *argv[] = {"echo", "this", "is", "echo", 0};
  // 操作系统从echo文件中加载指令，在当前进程中，替换当前进程的内存，然后执行这些指令。这里argv是传入的参数，相当于调用echo指令，并传递 argv对应的参数列表。
  // 这里实际上该程序通过调用exec系统调用，将自身替换成了echo程序，并产生对应的输出
  exec("echo", argv);
  
  printf("exec failed!\n");
  
  exit(0);
}
```

exec系统调用，保留了当前文件描述符表（table of file descriptors），不管当前程序引用过0、1、2或其他文件描述符，在新程序中也引用相同的东西（即我们已加载过的指令）。

原始进程中，exec不会有返回值，因为exec已经完全替换了当前进程的内存。

exec仅当出现错误时，才会在原始进程中有返回值（返回-1表示出现一些错误，比如找不到指定的文件）。

### 1.3.5 forkexec

**Unix中惯用程序的套路：如果想运行一个程序，然后重新获取其控制权，那就先fork，然后让子进程调用exec**。

如下示例`forkexec.c`

```c
#include "user/user.h"

// forkexec.c: fork then exec

int main()
{
  int pid, status;
  // 创建子进程
  pid = fork();
  if(pid == 0) {
    char *argv[] = {"echo", "THIS", "IS", "ECHO", 0};
    // 子进程通过调用系统调用，将当前进程替换成执行"echo"程序
    exec("echo", argv);
    // 正常不会执行到下面，因为exec如果正常运作则不会有返回值，因为子进程已经被完全替换。
    // 只有exec出现错误，才会返回-1，然后才会继续执行如下代码
    printf("exec failed!\n");
    // 退出参数1，如果执行到这，则将1传递给父进程wait函数
    exit(1);
  } else {
    printf("parent waiting\n");
    // wait系统调用，允许(当前，即父)进程等待它的任何子进程返回。
    // 这个状态参数是退出子进程的一种方式，将一个32位的整数值从当前的子进程传递给等待的父进程
    // 这里将status的地址传递给内核，内核向该地址写入子进程向exit传入的参数
    wait(&status);
    printf("the child exited with status %d\n", status);
  }
  
  exit(0);
}
```

**Unit惯例，一个程序成功完成并退出，则退出状态码（exist with status 0）为0，出现异常时退出则约定退出码为1。**

**如果当前程序关心子进程的运行状态是否成功，就可在父进程中调用wait系统调用，捕获子进程的退出状态码**。

wait系统调用：

+ 如果当前进程有子进程，只要其中一个退出，该wait系统调用就会返回
+ 如果当前进程没有子进程，那wait会立即返回-1错误，表示当前进程没有子进程
+ 如果fork了N次创建N个子进程，则需要调用至少N次wait保证至少获取N次子进程退出的状态码。
+ wait方法返回值即子进程进程号pid，借此可以区分是哪个子进程退出

---

<u>这里可以看出来，fork后紧接着exec是一个很常见的做法，但是fork复制整个父进程显得有些资源浪费，因为exec执行后会丢弃原本复制的所有内存，将其替换成代码中指定运行的文件的内容。这就引入了后面的概念，**(copy on write) 写时拷贝的 fork**</u>。

> Unix没有方法能够让子进程等待父进程退出，只有wait方法支持让父进程等待子进程退出。

### 1.3.6 redirect

IO重定向，举例`redirect.c`：

运行下述程序时，子程序被"echo"指令替换，并且输出被重定向到文件"output.txt"

```c
// redirect.c: run a command with output redirected

int main()
{
  int pid;

  pid = fork();
  if(pid == 0) {
    // 这里关闭1，是因为想让文件描述符1指向其他地方。我们不想用父进程文件描述符1（也就是shell连接到控制台的描述符）。前面提到过子进程会复制父进程的文件描述符表，所以这里关闭的是复制过来的文件描述符1
    close(1);
    // 这里open一定返回1，因为open的语义保证返回未被使用的最低的文件描述符号。
    // 文件描述符0仍然连接到控制台console，而文件描述符1刚被关闭
    // 这里文件描述符1，链接到文件"output.txt"
    open("output.txt", O_WRONLY | O_CREATE);

    char *argv[] = {"echo", "this", "is", "redirected", "echo", 0};
    // 这里 由于本身 "echo" 会从 文件描述符0获取输入，输出到文件描述符1，而这里1被指向了文件"output.txt"，所以"echo"指令在不知情的情况下，将数据输出到了"output.txt"中。
    exec("echo", argv);
    printf("exec failed!\n");
    exit(1);
  } else {
    wait((int *) 0);
  }
  exit(0);
}
```





