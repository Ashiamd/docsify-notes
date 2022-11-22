# MIT6.S081网课学习笔记

> [MIT6.S081 操作系统工程中文翻译 - 知乎 (zhihu.com)](https://www.zhihu.com/column/c_1294282919087964160) => 知乎大佬视频中文笔记，很全

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

> [pipe和fork浅析_qq_43812167的博客-CSDN博客](https://blog.csdn.net/qq_43812167/article/details/113483030)
>
> [Linux rm删除文件后磁盘空间不释放 - 简书 (jianshu.com)](https://www.jianshu.com/p/ded2ada16aad) => 文件描述符占用，引用计数 > 0

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

# Lecture3 OS组成和系统调用

## 3.1 本节重点

+ Isolatoin
+ Kernel / User Mode，隔离操作系统内核和用户应用程序
+ System Calls，应用程序通过调用系统调用跳转到内核执行一层方法
+ XV6：本课程用到的小型OS

## 3.2 假设没有OS

​	设想一个场景，假设当前不存在操作系统，应用程序能够直接导入一个OS代码库，直接操作底层硬件资源。

​	那么在单核机器上，应用程序A和应用程序B同时运行时，假设A占用CPU，为了避免A永久占用CPU，其会主动适时让出CPU资源，这种机制也叫做**协同调度（Cooperative Scheduling）**。

​	但是这种场景下，如果程序A有bug导致进程A无限占用CPU资源，由于没有操作系统干预，我们甚至不能用第三方程序来kill进程A，这种场景就达不到真正的**多路复用multiplexing**。

​	**multiplexing**不管占用CPU资源的进程在做什么，都该能迫使进程时不时的释放CPU，使其他程序有运行的机会。

​	另外，如果没有OS，不同应用之间的内存写入操作可能互相覆盖，而我们希望内存资源的使用具有**隔离性Isolation**。

## 3.3 操作系统抽象硬件层使用

> [Cache and memory affinity optimizations - IBM Documentation](https://www.ibm.com/docs/en/aix/7.2?topic=optimizations-cache-memory-affinity)

​	CPU、内存等硬件，本身以某些形式处理二进制0/1数据，本身没有什么其他特殊能力。而操作系统抽象出进程实体，创造出分时复用CPU的概念，使得应用程序无需关注CPU如何运作，转而关注进程本身。

​	某种意义上，可以有如下类比：

+ 进程processes 抽象了 CPU资源

  应用程序以进程的形式运行，分时复用CPU资源，但无感知CPU如何运作。

+ exec指令 抽象了 内存资源

  exec加载制定文件对应的程序，替换当前进程分配的内存，但也并非直接访问物理内存1000-2000这段地址。操作系统提供了**内存隔离**并控制内存。

+ files 抽象了 磁盘

  应用程序不直接读写磁盘扇区上的区块(the block of disk)，而是读写抽象的文件实体。实际上Unix也不允许应用程序直接操作磁盘，应用程序只能读写文件，而随后操作系统再决定文件如何与磁盘的块映射。通过files抽象能做到不同用户之间、同用户但不同进程之间的文件操作隔离。

---

问题：复杂的内核会不会尝试把同一个进程调度到同一个CPU上，以减少缓存丢失（cache miss）？

答：有一种机制，叫做缓存关联（cache affinity），其主要目的即尽量避免缓存丢失和类似问题，以提高性能。

## 3.4 OS应该有自我防护机制(Defensive)

​	操作系统需要确认所有进程在无异常情况下都能正常运作。如果应用程序无意/故意向系统调用传入错误的参数，OS也不应该崩溃。

​	为了实现OS能defensive，我们需要OS和应用程序之间strong isolation，通常可以借助硬件的隔离性达到该目的。

常见的**硬件隔离性**体现有：

+ user/kernel mode
+ page table（虚拟内存 Virtual Memory）

​	**一般来说，如果一个CPU处理器需要能运行一个支持多应用的操作系统，需要同时支持user/kernel mode 和 page table（虚拟内存 Virtual Memory）**。

## 3.5 CPU User/Kernel Mode

> [bios_百度百科 (baidu.com)](https://baike.baidu.com/item/bios/91424?fr=aladdin)
>
> 其实，它是一组固化到[计算机](https://baike.baidu.com/item/计算机?fromModule=lemma_inlink)内[主板](https://baike.baidu.com/item/主板?fromModule=lemma_inlink)上一个[ROM](https://baike.baidu.com/item/ROM?fromModule=lemma_inlink)[芯片](https://baike.baidu.com/item/芯片?fromModule=lemma_inlink)上的[程序](https://baike.baidu.com/item/程序?fromModule=lemma_inlink)，它保存着计算机最重要的基本输入输出的程序、开机后自检程序和系统自启动程序，它可从[CMOS](https://baike.baidu.com/item/CMOS/428167?fromModule=lemma_inlink)中读写系统设置的具体信息。 其主要功能是为计算机提供最底层的、最直接的硬件设置和控制。此外，BIOS还向作业系统提供一些系统参数。系统硬件的变化是由BIOS隐藏，程序使用BIOS功能而不是直接控制硬件。现代作业系统会忽略BIOS提供的抽象层并直接控制硬件组件。
> [用户与内核的交互-用户程序向内核发起交互的方式_anniesqq的博客-CSDN博客](https://blog.csdn.net/weixin_42534168/article/details/115711991)
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210414233525391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjUzNDE2OA==,size_16,color_FFFFFF,t_70)
>
> [从零开始学内核:用户进程和系统调用 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/524880521)
>
> [80x86中断 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/450293200)

​	为了支持User/Kernel Mode，处理器会有两种操作(工作)模式，一种即User Mode，另一种则是Kernel Mode。

+ Kernel Mode

  在该模式下运行时，CPU可以执行特定权限的指令（privileged instructions）。

  特殊权限指令都是一些直接操作硬件和设置保护的指令，例如设置page table寄存器，关闭时钟中断。处理器有多种状态，只有特权指令能够操作/变更这些状态。当一个普通应用要求执行一条特殊权限指令，处理器会拒绝执行该指令，因为不允许在User Mode执行特殊权限指令。通常来说，这种场景下，CPU控制权会从User Mode切换到Kernel Mode，而操作系统拿到控制权后，或许会kill该进程，因为其misbehaving。

+ User Mode

  在该模式下运行时，CPU只能执行普通权限的指令（unprivileged instructions）。

  比如两个寄存器相加ADD，相减SUB，跳转JRC、BRANCH指令等。所有应用程序都允许执行这些普通权限指令。

---

问题：如果kernel mode允许一些指令的执行，user mode不允许一些指令的执行，那么是谁在检查当前的mode并实际运行这些指令，并且怎么知道当前是不是kernel mode？是有什么标志位吗？

回答：**在处理器中，有1bit作为flag用于标识当前运行的Mode（1 User Mode，0 Kernel Mode）。如果CPU将执行的指令是特殊权限指令，并且该flag当前值为1，那么CPU就会拒绝执行该指令**（就像运算不能除0一样）。

问题：CPU只提供这一方式控制User/Kernel Mode之间的切换？

回答：是的，并且**只有特殊权限指令能够修改表示CPU运行Mode的1bit位flag**，否则其他应用程序随意切换CPU的Mode，就能运行各种特殊权限指令了。

问题：考虑到安全性，所有用户代码通过内核才能访问硬件，是否存在一种方法使用户能随意访问内核？

回答：如果设计足够严谨，用户就不能随意访问内核。**或许有些程序在操作系统认可下，能拥有额外的特权，但并不是所有用户都能拥有这些特权，只有root用户有特权执行安全相关的操作**。

问题：BIOS是在操作系统之前还是之后运行？

回答：BIOS是计算机自带的代码，BIOS启动后，由BIOS来启动操作系统。所以BIOS必须是可信的代码，最好是正确且无恶意的。

问题：既然修改处理器中用于标记Mode的1bit位flag的指令是特权指令，用户程序无法修改CPU的当前Mode为Kernel Mode，那么用户程序怎么才能让内核执行指定的特权指令？

回答：（视频中断，这里我网上查了一下可能正确的回答）用户进程调用系统调用函数，触发中断(INT 0x80)，操作系统将当前用户进程挂起，进入中断处理函数，切换到内核态，执行与系统调用函数内系统调用号一致的系统调用，执行系统调用相关逻辑，将返回值写入用户态也能读取的寄存器中（比如EAX），切换回用户态，恢复用户进程，用户进程从寄存器中取得系统调用返回值。

## 3.6 CPU 虚拟内存(Virtual Memory)

> [CPU架构：缓存（1）—— 虚拟地址 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/455279314)

​	基本所有CPU都支持虚拟内存(Virtual Memory)，CPU包含的页表即是具体的实现，通过页表，将虚拟地址映射到实际的物理地址。

​	**每个进程都有独立的页表(page table)，且只能访问自己page table中对应的物理内存。操作系统维护页表，使得每个进程能够拥有不重合的物理内存映射（尽管每个进程页表都是从0开始，到2^n结束，但是实际映射的物理地址并不重叠）。这实现了强内存隔离性（strong memory isolation）。**

## 3.7 控制权转交内核(Entering Kernel)

​	用户进程为了能够执行特权指令，需要能够切换到内核态，由于用户进程本身没有权限修改CPU的Mode，所以需要有一种机制允许用户进程**将控制权转交给内核(Entering Kernel)**。

​	在课程教学用的XV6系统中，ECALL指令接受一个数字参数。当一个用户程序想要将程序执行的控制权转移到内核，它只需要执行ECALL指令，并传入一个数字。这里的数字参数代表了应用程序想要调用的System Call。

​	XV6存在一个唯一的系统调入点，每一次应用程序调用ECALL指令，应用程序都会通过该系统调入点进入到内核。<u>比如用户程序中调用fork函数，并不是直接调用操作系统中对应的fork函数，而是内部调用ECALL指令，并将fork系统调用对应的数字作为参数传递给ECALL</u>。在内核侧有一个在`syscall.c`中的函数`syscall`，每一个程序通过ECALL（实际用户进程也只能调用ECALL，不能直接越权调用fork等系统调用函数）发起的系统调用都会调用到`syscall`函数，其会检查ECALL的参数，内核通过该参数得知用户进程需要调用的函数是fork。

----

问题：看似用户进程只需执行ECALL指令，之后就能完成系统调用（内核去执行），但是内核何时决定当前用户进程是否有权限能执行特定的系统调用（即内核何时相应用户的ECALL指令，去执行对应数字参数的系统调用）？

回答：原则上，内核中fork被执行时，它首先会检查入参的有效性，决定调用方是否被允许执行fork系统调用。在Unix中，任何应用程序都能调用fork。以write为例，write的实现中会检查传递给write的地址是否在用户进程的页表范围内。

问题：当应用程序无意或恶意死循环执行，那么内核如何夺回控制权限？

回答：具体后续章节介绍。内核会通过硬件设置一个定时器，定时器到期后会将控制权从用户空间转移到内核空间，之后内核有了控制权，可以重新调度CPU到另一个进程中。

问题：是什么驱动操作系统的设计人员使用C作为编程语言？

回答：C提供了很多对于硬件的控制能力，比如你要编程一个定时器芯片时，用C语言很容易实现，因为其提供的库函数能对任何硬件资源进行大量底层控制。

问题：C比C++在操作系统编程中更为流行，仅仅是因为历史原因（C语言库很早就具备底层硬件的操控能力）吗？

回答：大部分操作系统用C而不是C++，可能有一部分原因是Linus比起C++更喜欢C。

## 3.8 TCB (Trusted Computing Base)

​	由于内核在执行系统调用等方法时，有较完善的参数检测（包括参数具体的内存是否越界，用户进程A尝试访问用户进程B的内存等），以确保不会被错误的参数欺骗，所以内核有时也被称作可被信任的计算空间（安全术语TCB，Trusted Computing Base）。

+ Kernel must have no bug

  能被称为TCB，意味着内核首先要是正确且没有bug的。假设内核有bug，攻击者能通过该bug进行攻击，转而将其变成漏洞，使得攻击者能够打破操作系统的隔离性，随意操控内核。

+ Kernel must treat processes as malicious

  另一方面，内核必须把所有用户进程或进程当作是恶意的。内核设计者编程时需要有较强的安全意识。

## 3.9 宏内核(Monolithic Kernel Design)

​	**为了保证敏感操作都只能在Kernel Mode中执行，一种选择是将整个操作系统都运行在Kernel Mode中。大多数Unix操作系统实现都在Kernel Mode中运行。这种形式被称为宏内核(Monolithic Kernel Design)**。*本课程的XV6所有操作系统服务都在Kernel Mode中。*

宏内核的缺点：

+ 内核直接运行整个操作系统，任何一个bug都可能升级为漏洞，且bug出现概率更高。

宏内核的优点：

+ 文件系统、虚拟内存、进程管理等子模块都在同一个程序中，集成紧密度高，性能更高，比如Linux。

## 3.10 微内核(Micro Kernel Design)

​	**与宏内核相反，微内核关注点在于减少内核中的代码，其被称为微内核(Micro Kernel Design)。在这种模式下，也是有内核存在，但是内核只有非常少的子模块**。

​	微内核通常会有一些IPC实现(或者是其他消息传递形式)，且对虚拟内存的支持少，可能只支持page table，以及CPU分时复用。<u>微内核目的在于将大部分的操作系统运行在内核之外</u>。

​	比如，微内核，可能会将文件系统或者虚拟内存的一部分实现放在用户进程中（User Mode）。

微内核的优点：

+ 内核的代码更少，意味着更少的bug。

微内核的缺点：

+ 性能开销更高/性能更低。假设文件系统在用户进程中运行，那么某个用户进程需要调用文件系统的方法时，首先需要通过IPC和内核交互，内核再转发消息给用户进程中的文件系统，文件系统处理完后响应内核，内核再转发结果给用户进程，这是典型的通过消息实现的传统系统调用。（微内核下，任何用户进程访问文件系统，都需进行2次完整的User/Kernel Mode的跳转，而宏内核下，文件系统在内核中，用户进程访问文件系统只需1次完整的User/Kernel Mode的跳转）。
+ 微内核中，内核子模块之间隔离开了，内核子模块之间很难实现高性能的共享。

## 3.11 宏内核和微内核应用

​	前面对宏内核、微内核仅仅是做了笼统的概述，实际两种内核设计都会出现。

​	由于历史原因，大多数桌面操作系统都是典型的单一系统。许多O-S紧张的应用，例如数据中心服务器上的操作系统，通常是宏内核实现，因为Linux提供了良好的性能。

​	而很多嵌入式系统，如Minix、Cell，都是为内核设计。

​	两种内核实现都十分流行。而如果现在着手设计操作系统，你可能会优先考虑为内核，因为一旦以宏内核的方式实现的操作系统足够复杂，将其重写到微内核设计将十分困难。并且人们一般没有足够的动机改写/重构内核，而是重点关注实现新功能/服务。

## 3.12 编译XV6的Kernel代码

​	XV6作为宏内核实现的操作系统，在kernel目录下的所有文件最后被编译成名为"kernel"的二进制文件，其之后会被运行在Kernel Mode中。

​	另一部分在user目录下，基本都是运行在User Mode的程序。

​	还有一个目录mkfs，该目录下的"mkfs"二进制文件，它会创建一个空的文件镜像，我们将该镜像存在磁盘上，这样我们就能直接使用一个空的文件系统。

理解编译流程：

（1）makefile读取一个C文件，比如`proc.c`

（2）调用gcc编译器，生成`proc.s`（汇编语言文件）

（3）通过汇编解释器（assembler），生成`proc.o`（汇编语言的二进制格式）

（4）makefile为所有内核文件重复以上动作（1）～（3），之后系统加载器（Loader）收集所有`.o`文件，将它们链接到一起，生成内核文件

​	教程为了方便理解，makefile还生成了`kernel.asm`，包含了内核的完整汇编语言。（可以用于定位哪个指令导致bug）

## 3.13 QEMU

> [3.8 QEMU - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/266502180)

QEMU即模拟主板（有CPU、PCI-E等），个人觉得没什么好说的，建议看知乎网友的笔记。

## 3.14 XV6启动过程

> [3.9 XV6 启动过程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/266502391) <= 摘自该博文，这里大概了解一下，知道操作系统启动时的大致子模块初始化的步骤顺序就好了。

XV6启动时，大致步骤如下：

+ `consoleinit()`：初始化，启动console
+ `kinit()`：设置好页表分配器（page allocator），physical page allocator
+ `kvminit()`：设置好虚拟内存，这是下节课的内容，create kernel page table
+ `kvminithart()`：打开页表，也是下节课的内容，turn on paging
+ `processinit()`：设置好初始进程或者说设置好进程表单，process table
+ `trapinit()/trapinithart()`：设置好user/kernel mode转换代码，trap vectors；install kernel trap vector
+ `plicinit()/plicinithart()`：设置好中断控制器PLIC（Platform Level Interrupt Controller），后面在介绍中断的时候会详细的介绍这部分，用来与磁盘和console交互，set up interrupt controller；ask PLIC for device interrupts
+ `binit()`：分配buffer cache，buffer cache
+ `iinit()`：初始化inode缓存，inode cache
+ `fileinit()`：初始化文件系统，file table
+ `virtio_disk_init()`：初始化磁盘， emulated hard disk
+ `userinit()`：最后当所有的设置都完成了，操作系统也运行起来了，会通过userinit运行第一个进程（后续能看出来是通过ECALL请求系统调用EXEC调用init程序，而init程序配置console，调用fork创建子程序执行了shell），first user process

​	最后的`userinit()`创建初始进程，执行3条指令后跳转回内核空间。（首先将init中的地址加载到a0（la a0, init），argv中的地址加载到a1（la a1, argv），exec系统调用对应的数字加载到a7（li a7, SYS_exec），最后调用ECALL。所以这里执行了3条指令，之后在第4条指令将控制权交给了操作系统）

​	对照`syscall.h`，得知这里`userinit()`最后ECALL请求调用的是EXEC系统调用。sys_exec中的第一件事情是从用户空间读取参数，它会读取path，也就是要执行程序的文件名。这里首先会为参数分配空间，然后从用户空间将参数拷贝到内核空间。之后我们打印path，发现传入的是"/init"程序，即`userinit()`请求调用init程序。

​	init程序配置好console，调用fork，并在fork得到的子进程中执行shell。

​	这里简单的介绍了一下XV6是如何从0开始直到第一个Shell程序运行起来。并且我们也看了一下第一个系统调用是在什么时候发生的。我们并没有看系统调用背后的具体机制，这个在后面会介绍。
