# MIT6.S081网课学习笔记

> [MIT6.S081 操作系统工程中文翻译 - 知乎 (zhihu.com)](https://www.zhihu.com/column/c_1294282919087964160) => 知乎大佬视频中文笔记，很全
>
> [6.S081 / Fall 2020 (mit.edu)](https://pdos.csail.mit.edu/6.S081/2020/schedule.html) => 课表+实验超链接等信息

# Lecture1 概述和举例

Introdution and Example

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

> [pipe和fork浅析_qq_43812167的博客-CSDN博客 ](https://blog.csdn.net/qq_43812167/article/details/113483030)<= 建议阅读
>
> [Linux rm删除文件后磁盘空间不释放 - 简书 (jianshu.com)](https://www.jianshu.com/p/ded2ada16aad) => 文件描述符占用，引用计数 > 0
>
> [文件描述符表、文件表、索引结点表 - PKICA - 博客园 (cnblogs.com)](https://www.cnblogs.com/guxuanqing/p/13377422.html) <= 建议阅读
>
> 每个进程都有一个属于自己的文件描述符表。
>
> 文件表存放在内核空间，由系统里的所有进程共享。
>
> 索引结点表也存放在内核空间，由所有进程所共享。
>
> 文件描述符表项有一个指针指向文件表表项，文件表表项有一个指针指向索引结点表表项。

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

**exec系统调用，保留了当前文件描述符表（table of file descriptors）**，不管当前程序引用过0、1、2或其他文件描述符，在新程序中也引用相同的东西（即我们已加载过的指令）。

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

# Lecture3 OS和系统调用

OS Organization and System Calls

## 3.1 本节重点概述

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

# Lecture4 页表Page Table(虚拟内存VM)

Page Table (VM, Virtual Memory)

## 4.1 虚拟内存概述

​	隔离性是我们讨论虚拟内存的主要原因。这节课主要关注虚拟内存的工作机制，以及如何运用这些机制完成一些技巧操作。

+ 地址空间（Address Spaces）
+ 支持虚拟内存的硬件
+ XV6中的虚拟内存代码，kernel address space and user address spaces

## 4.2 地址空间 (Address Spaces) 概述

> [DRAM动态随机存取存储器_百度百科 (baidu.com)](https://baike.baidu.com/item/动态随机存取存储器/12717044?fromtitle=DRAM&fromid=149572&fr=aladdin)
>
> [SRAM（静态随机存取存储器）_百度百科 (baidu.com)](https://baike.baidu.com/item/SRAM/7705927?fromModule=lemma_inlink)
>
> [地址空间_百度百科 (baidu.com)](https://baike.baidu.com/item/地址空间/1423980?fr=aladdin)
>
> 就像进程的概念创造了一类抽象的CPU以运行程序一样，地址空间为程序创造了一种抽象的内存。地址空间是一个进程可用于寻址内存的一套地址集合。每个进程都有一个自己的地址空间，并且这个地址空间独立于其他进程的地址空间（除了在一些特殊情况下进程需要共享它们的地址空间外）
>
> + 物理地址 (physical address): 放在寻址总线上的地址。放在寻址总线上，如果是读，电路根据这个地址每位的值就将相应地址的物理内存中的数据放到数据总线中传输。如果是写，电路根据这个地址每位的值就将相应地址的物理内存中放入数据总线上的内容。**物理内存是以字节(8位)为单位编址的**。
>
> + 虚拟地址 (virtual address): CPU启动保护模式后，程序运行在虚拟地址空间中。注意，并不是所有的“程序”都是运行在虚拟地址中。**CPU在启动的时候是运行在实模式的，内核在初始化页表之前并不使用虚拟地址，而是直接使用物理地址的**。

​	*回顾之前的知识点，通过虚拟内存和正确的管理page table，能实现强隔离性。这节课，我们着重关注内存的隔离性*。

​	所有程序在运行时，最终都需要处于物理内存中的某一位置。假设不存在内存隔离，那么进程A可能由于错误操作导致进程B的地址所在内存被写覆盖。**地址空间（Address Spaces）是一种能够将不同程序之间的内存隔离开的机制**。

## 4.3 页表(Page Table)

> [页表_百度百科 (baidu.com)](https://baike.baidu.com/item/页表/679625?fr=aladdin)
>
> 页表是一种特殊的[数据结构](https://baike.baidu.com/item/数据结构/1450?fromModule=lemma_inlink)，放在系统空间的页表区，存放逻辑页与物理页帧的对应关系。 每一个[进程](https://baike.baidu.com/item/进程/382503?fromModule=lemma_inlink)都拥有一个自己的页表，[PCB](https://baike.baidu.com/item/PCB/16067368?fromModule=lemma_inlink)表中有指针指向页表。
>
> [MMU_百度百科 (baidu.com)](https://baike.baidu.com/item/MMU/4542218?fr=aladdin)
>
> MMU是Memory Management Unit的缩写，中文名是[内存管理](https://baike.baidu.com/item/内存管理?fromModule=lemma_inlink)单元，有时称作**分页内存管理单元**（英语：**paged memory management unit**，缩写为**PMMU**）。它是一种负责处理[中央处理器](https://baike.baidu.com/item/中央处理器?fromModule=lemma_inlink)（CPU）的[内存](https://baike.baidu.com/item/内存?fromModule=lemma_inlink)访问请求的[计算机硬件](https://baike.baidu.com/item/计算机硬件?fromModule=lemma_inlink)。它的功能包括[虚拟地址](https://baike.baidu.com/item/虚拟地址?fromModule=lemma_inlink)到[物理地址](https://baike.baidu.com/item/物理地址?fromModule=lemma_inlink)的转换（即[虚拟内存](https://baike.baidu.com/item/虚拟内存?fromModule=lemma_inlink)管理）、内存保护、中央处理器[高速缓存](https://baike.baidu.com/item/高速缓存?fromModule=lemma_inlink)的控制，在较为简单的计算机体系结构中，负责[总线](https://baike.baidu.com/item/总线?fromModule=lemma_inlink)的[仲裁](https://baike.baidu.com/item/仲裁?fromModule=lemma_inlink)以及存储体切换（bank switching，尤其是在8位的系统上）。
>
> [操作系统中的多级页表到底是为了解决什么问题？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/63375062) （知乎用户6AIBRo，该用户的回答图文并茂，更易理解）
>
> [4.3 页表（Page Table） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/270577411) => 图片出处

​	页表(Page Table)是常见且灵活的实现地址空间(Address Spaces)隔离的一种方式。

​	页表(Page Table)的实现，需要硬件支持，所以实际上是处理器在硬件上实现的，或者具体说是通过内存管理单元（MMU，Memory Management Unit）实现的。

​	实际上CPU进行数据读写时，并不是直接操作物理地址，而是操作虚拟地址。虚拟地址传递到MMU后，MMU再将虚拟地址翻译成物理地址。后续再从具体的物理地址加载或写入数据。

​	从CPU的角度来看，一旦MMU已启动（正常运作中），那么CPU操作的每条指令对应的地址都是虚拟内存地址。**为了将虚拟地址转换为物理地址，MMU读取虚拟地址到物理地址的映射表。该映射表实际也存储在内存中，CPU用寄存器来存储该映射表在物理内存的实际地址（MMU启动时，CPU的寄存器告知MMU其映射表在物理内存中具体地址是什么）**。

​	**实际上，MMU本身并没有存储(虚拟地址到物理地址)映射表，MMU只是读取了CPU寄存器中指定物理地址的内存中存储的映射表，然后处理虚拟地址到物理地址映射的工作**。

​	**页表(Page Table)的基本思想即让每个进程都有属于自己的虚拟映射表。当操作系统将CPU从进程A切换到晋城B时，同时也需要切换CPU用于存放映射表物理地址信息的寄存器存储的内容**。正因如此，切换进程后，即使访问的虚拟地址看似一致，但实际映射的物理地址并不相同（因为每个进程都有各自的虚拟映射表）。

​	*假设存储映射表物理地址的CPU寄存器能存储64bit，那么理论上能映射2^64个物理地址。如果我们直接用2^64个地址管理物理内存，那么表单变得十分大，所有内存都会被耗尽，这不合理*。

（1）第一步：**不要为每个地址创建一个表单条目，而是以表单page为单位创建一个表单条目**。每一次MMU地址翻译针对一整个page。在RISC-V中，一个Page大小是4KB(连续的4096Bytes)，**几乎所有的处理器都使用或支持4KB大小的page**。此时，虚拟内存地址被分为两部分，index和offset，index用于查找page，offset表示偏移量(page内具体某一字节)。MMU先用index通过映射表翻译成物理内存中某一页Page(4KB，4096Bytes)的物理地址，而offset则表示该page中偏移量(某一具体字节)。

​	在RISC-V中，存储器的64bit并非都用做表示虚拟地址(仅使用39bit)，实际高25位没有使用，只有2^39个虚拟地址(512GB)，39bit中，高27bit作index，而低12bit作offset（2^12正好对应4KB）。

​	在RISC-V中，物理内存地址共56bit，大于单个虚拟内存地址空间。大多数主板还不支持2^56规模的物理内存。该方案中，<u>高44bit是物理page号（PPN，Physical Page Number），低12bit继承自虚拟地址，表示offset（连续的4KB，页内偏移量，定位到某一Byte）</u>。

​	想象一下，由于每个进程都有一张page table，那么如果每个进程都有个2^27个条目映射表单，那存储映射表就需要消耗大量的物理内存。

​	**实际上page table是一个多级结构**。前面提到RISC-V虚拟地址27bit作index，该27bit又拆成3部分，高9bit索引top level page table directory（L2），一个directory大小为4KB（2^12，和page保持一致），一个条目被称为PTE（Page Table Entry），大小8B(2^3，和寄存器大小64bit一致)，这意味着一个directory page有512个(2^9)条目。存储映射表地址的寄存器，实际存储的地址是top level page table directory条目的地址。这时我们读取index的高9bit得到一个PPN（物理page号），该PPN指向中层级的page directory（L1，index中间9bit），再通过L1完成索引，到L0（index低9bit），此时查询L0才真正将虚拟地址映射到物理地址。

​	<u>多级页表的优势在于，如果地址空间中大部分地址都没有用，那不需要为它们在page table directory中记录page table条目</u>。假设当前进程A需要一个4KB空间的内存，刚好是1page大小，那么在多级页表中，只需要L2、L1、L0各一条记录，共3个page directory，共3*512个PTE。而在单一page table的方案中，即使只需要1page大小的内存地址，仍然需要2^27个PTE。

​	**显然，通过多级页表比起单级页表，更节省内存地址空间资源**。

---

问题：如果通过3级page table directory的PPN翻译到最后的物理地址？

回答：最高级的L2 page table directory中的PPN包含了下一级L1的物理地址，以此类推，在最低级L0 page table directory，最终得到44bit PPN，这里包含实际想要翻译的内存物理地址。

**问题：为什么存在page table directory中的PTE记录的是PPN（物理地址），而不是虚拟地址？**

**回答：因为我们需要在物理内存中查找下一个page table directory的物理地址，毕竟我们不能让地址翻译依赖另一个地址翻译，否则就无限递归循环了。所以page table directory本身必须存储在物理地址中，且PTE记录PPN（下一个Level page table directory的物理地址）。**

**问题：既然page table directory必须存储在物理地址，那么记录页表地址的寄存器（教材是SATP寄存器），其存储的64bit内容，是物理地址信息，还是虚拟地址信息？**

**回答：这里SATP寄存器（记录页表地址的寄存器）存储的信息必须是page table directory的物理地址，毕竟最高级L2 page table directory存储在物理地址中。**根据前面提到的不能依赖地址翻译来进行另一项地址翻译，这里SATP必须知道L2 page table directory的物理地址。

问题：前面提到虚拟地址index由27bit组成，在多级页表的视线中，27bit拆成3个9bit对应3个page table directory的索引，怎么理解这3部分9bit的关系？

回答：虚拟地址27bit的index中，最高9bit索引L2 page table directory，中间9bit索引L1 page table directory，后续的9bit索引L0 page table directory。

问题：当一个进程请求一个虚拟内存地址时，CPU会查看SATP寄存器，（从物理地址加载寄存器存储的物理地址信息，）得到最高层级L2 page table directory，然后接下去该做什么呢？

回答：之后会使用虚拟内存地址的top 27bit完成对page table directory内PTE的索引。

问题：如果虚拟地址对page table directory索引后，发现结果为空，MMU会新建一个page table吗？

回答：不会，MMU会告知操作系统or处理器，当前虚拟地址无法翻译，这种情况会转换成一个page fault，后续会讲这个话题。 如果一个虚拟地址无法被翻译成物理地址，那么MMU就不会翻译这个地址，就像CPU在计算时会拒绝除0操作一样。

**问题：读取L2 page table directory后，通过虚拟地址index中top 9bit索引到L2的page table directory中PTE44bit的PPN+虚拟地址的12bit的offset获取完整的56bit page table 物理地址？**

**回答：这里L2 page table directory索引到L1 page table directory的过程，不会加上虚拟地址的offset，而是使用12bit的0。这里用44bit的PPN+12bit的0，得到56bit的物理地址，即下一级L1 page table directory的物理地址。因为这里要求每级page table directory都和物理的page对齐。**

问题：3级page table directory是操作系统实现的，还是由硬件自己实现的？

回答：由硬件实现，所以3级page table directory的查找都发生在硬件中。MMU是硬件的一部分，而不是操作系统的一部分。在XV6教学操作系统中，有一个叫`walk`函数实现了page table的查找，因为XV6有时候也需要完成硬件的工作。`walk`函数在软件中实现了等同于MMU硬件page table查找的功能。

![img](https://pic1.zhimg.com/80/v2-0534c528dc7123ea3f448a050a3fceb0_1440w.webp)

​	**在RISC-V中，PTE(54bit)由44bit（PPN，物理页编号）+10bit（Flags，标志位，用于控制地址权限）组成，还有10bit用于未来扩展**。

​	下面介绍PTE的Flags标记(用于控制地址权限)，这里只介绍前5位重要标识，后3位未介绍：

+ Valid（V）：1表示当前是一条valid PTE，可以直接用于地址翻译（虚拟地址=>物理地址）。为0时，MMU不能直接使用当前PTE。

+ Readable（R）：为1表示可以读取对应的page
+ Writable（W）：为1表示可以写对应的page
+ Executable（X）：为1表示可以从对应page执行指令
+ User（U）：为1表示对应page可以被运行在用户空间的进程访问
+ Global（G）：
+ Accessed（A）：
+ Dirty（D）：0 in page directory
+ Reserved for supervisor software（8-9两位）

## 4.4 转译后备缓冲区(TLB, Translation Lookaside Buffer)

> [(TLB, Translation Lookaside Buffer)转译后备缓冲区_百度百科 (baidu.com)](https://baike.baidu.com/item/转译后备缓冲区/22685572?fromtitle=TLB&fromid=2339981&fr=aladdin)
>
> **转译后备缓冲器**，也被翻译为**页表缓存**、**转址旁路缓存**，为[CPU](https://baike.baidu.com/item/CPU?fromModule=lemma_inlink)的一种缓存，由存储器管理单元用于改进[虚拟地址](https://baike.baidu.com/item/虚拟地址?fromModule=lemma_inlink)到物理地址的转译速度。当前所有的桌面型及服务器型处理器（如 [x86](https://baike.baidu.com/item/x86?fromModule=lemma_inlink)）皆使用TLB。
>
> TLB 用于缓存一部分标签页表条目。TLB可介于 CPU 和[CPU缓存](https://baike.baidu.com/item/CPU缓存?fromModule=lemma_inlink)之间，或在 CPU 缓存和[主存](https://baike.baidu.com/item/主存?fromModule=lemma_inlink)之间，这取决于缓存使用的是物理寻址或是虚拟寻址。<u>如果缓存是虚拟定址，定址请求将会直接从 CPU 发送给缓存，然后从缓存访问所需的 TLB 条目。如果缓存使用物理定址，CPU 会先对每一个存储器操作进行 TLB 查寻，并且将获取的物理地址发送给缓存</u>。两种方法各有优缺点。
>
> **指令与数据可以分别使用不同的TLB ，即Instruction TLB (ITLB)与 Data TLB (DTLB)，或者指令与数据使用统一的TLB，即Unified TLB (UTLB)，再或者使用分块的TLB (BTLB)**
>
> **在任务（task）切换时，部分 TLB 条目可能会失效，例如先前运行的进程已访问过一个页面，但是将要执行的进程尚未访问此页面。最简单的策略是清出整个 TLB。较新的 CPU 已有更多有效的策略；例如在Alpha EV6中，每一个 TLB 条目会有一个“地址空间号码”（address space number，ASN）的标记，而且只有匹配当前工作的 ASN 的 TLB 条目才会被视为有效**。
>
> 两种在现代体系结构中常用的解决 TLB 不命中的方案：
>
> - **硬件式 TLB 管理**，CPU 自行遍历标签页表，查看是否存在包含指定的虚拟地址的有效标签页表条目。如果存在这样的分页表条目，就把此分页表条目存入 TLB ，并重新执行 TLB 访问，而此次访问肯定会寻中，程序可正常运行。如果 CPU 在标签页表中不能找到包含指定的虚拟地址有效条目，就会发生标签页错误[异常](https://baike.baidu.com/item/异常?fromModule=lemma_inlink)，[操作系统](https://baike.baidu.com/item/操作系统?fromModule=lemma_inlink)必须处理这个异常。处理标签页错误通常是把被请求的数据载入物理存储器中，并在标签页表中创建将出错的虚拟地址映射到正确的物理地址的相应条目，并重新启动程序（详见标签页错误）。
> - **软件管理式 TLB**，TLB 不命中时会产生“TLB 失误”异常，且操作系统遍历标签页表，以软件方式进行虚实地址转译。然后操作系统将分页表中响应的条目加载 TLB 中，然后从引起 TLB 失误的指令处重新启动程序。如同硬件式 TLB 管理，如果 操作系统 在标签页表中不能找到有效的虚实地址转译条目，就会发生标签页错误， 操作系统 必须进行相应的处理
>
> [mfence, lfence, sfence什么作用?_清海风缘的博客-CSDN博客_sfence](https://blog.csdn.net/liuhhaiffeng/article/details/106493224)

​	**根据前文，这里RISC-V使用三级页表，每次处理器从内存加载数据时，虚拟地址到物理地址的翻译，就需要读取3次内存**。

（1）第一次根据SATP寄存器存储的信息，读取对应物理地址的L2 page table directory
（2）第二次根据虚拟地址的top 9bit 索引 L2 page table direcotry的某条PPN（44bit），然后补充12bit 0，读取对应物理地址（56bit）的L1 page table directory

（3）第三次根据虚拟地址 middle 9bit 索引 L1 page table directory的某条PPN（44bit），然后补充12bit 0，读取对应物理地址（56bit）的L0 page table directory，

​	后续通过最低9bit索引L0 page table directory的某条PPN（44bit），加上虚拟地址12offet，即将虚拟地址翻译成具体的物理地址。

​	**实际操作中，为了减少虚拟地址翻译带来的性能损耗，几乎所有处理器都会对最近使用的虚拟地址的翻译结果进行缓存。这种缓存通常称为Translation look-aside buffer（TLB），也就是Page Table Entry（PTE）的缓存。**

​	**当处理器第一次查询一个虚拟地址（对应的物理地址）时，硬件通过3级page table directory最后得到PPN，TLB会保存本次虚拟地址到物理地址的映射关系（VA，PA，PN，PA）。下次如果访问同一个虚拟地址，处理器先查看TLB，发现有映射记录则直接返回对应的物理地址**。

​	**page table提供了一层抽象（a level of indirection），这层抽象指的是虚拟地址到物理地址的映射，且这层抽象完全由操作系统控制。正因这层抽象由操作系统控制，操作系统可以实现很多功能。比如，当一个PTE（在MMU中）是无效的，硬件会raise page fault，操作系统发现page fault后，可以尝试更新page table，然后重新执行（原本在执行的）指令**。显然，通过操作page table，可以在运行完成很多复杂操作，后续会针对page table和page fault讲解操作系统能做哪些有趣的事情。page table为操作系统带来更多灵活性，这也是为什么page table得以流行。

---

问题：TLB map VA（virtual address）along the with offset to PA（physical address）of the page，TLB缓存虚拟地址（带有offset）到物理地址的具体映射，在page级别做cache缓存是不是更高效？

回答：实现TLB有很多方式，但这不是我们讨论的重点，因为**TLB的实现是CPU的一些固有逻辑，即使操作系统也不可见**，操作系统也不需要知道TLB如何运作。**我们需要关注的关于TLB的知识是，当切换了page table，操作系统OS需要告诉处理器CPU当前正在切换page table，TLB needs to be flushed（TLB需要被CPU即使擦除/清空）。本质上，如果你切换了一个新的page table，TLB中的缓存条目将不再有效，它们需要被移除removed，否则虚拟地址的翻译会不准确**。所以，操作系统OS只知道TLB的存在，并且时不时通知硬件当前TLB缓存无效，因为需要切换page table了。在RISC-V中，flush the TLB（擦除/清空 TLB）的指令是`sfence_vma`。

问题：TLB缓存虚拟地址到物理地址映射到机制，发生在什么地方？

回答：**TLB、MMU都在CPU核中（每个CPU核都有自己的TLB、MMU）**。

**问题：除了TLB这种维护虚拟地址到物理地址的映射的缓存以外，有时候CPU并没有直接访问内存，这时CPU是从什么缓存中获取数据的？**

**回答：RISC-V处理器有L1、L2缓存。有些缓存是根据物理地址索引的，有些缓存是根据虚拟地址索引的。由虚拟地址索引的cache位于MMU之前（即直接访问cache获取数据，而不需要再通过MMU转译物理地址查找物理内存）；而由物理地址索引的cache在MMU之后（即需要MMU将虚拟地址翻译成物理地址后再索引该cache中的数据）。**

**问题：既然硬件TLB能够自己缓存虚拟地址到物理地址的映射，并且MMU的page table directory索引也是发生在硬件中，那么为什么还需要软件实现`walk`函数，来达到page table查找的效果？**

**回答：这有许多原因。首先，XV6的`walk`函数设置了initial page table，它需要对3级page table进行编程，所以它首先需要能模拟3级page table。另外一点，你们在systcall实验中遇到了，XV6中，内核有它自己的page table，用户进程也有它自己的page table。例如，用户进程指针指向sys_info结构体，该指针处于用户空间的page table，但是内核需要将这个指针翻译成自己能够读写的物理地址。如果你看copy_in或copy_out，会发现内核使用用户进程的page table翻译user virtual address到物理地址，这样内核后续就能（根据物理地址）读写对应的物理内存。这就是XV6需要实现`walk`函数的一些原因。**

问题：为什么硬件不开发类似`walk`的函数，这样我们（软件）就不需要另外开发，减少不必要的bug。为什么没有一个特殊权限指令，接收虚拟地址，然后返回物理地址？

回答：这就像，你往一个虚拟地址写数据，硬件会自动帮你完成工作一样（即硬件帮你自动将虚拟地址翻译成物理地址，然后后续操作物理地址的内存进行写数据操作）。你们在page table实验中，会完成相同的工作，你可以将page table设置得稍微不一样，这样就可以避免copy_in和copy_instr中的walk函数。

## 4.5 内核页表(Kernel Page layout)

> [4.5 Kernel Page Table - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/270578959) <= 图片出处
>
> [中断控制器_百度百科 (baidu.com)](https://baike.baidu.com/item/中断控制器/15732992?fr=aladdin)
>
> 多个外部中断源共享中断资源，必须解决相应的一些问题，例如CPU芯片上只有一个INTR输入端，多个中断源如何与INTR连接、中断矢量如何区别、各中断源的优先级如何判定等。
>
> 当CPU响应中断并进入[中断服务程序](https://baike.baidu.com/item/中断服务程序/10510195?fromModule=lemma_inlink)的处理过程后，中断控制器仍负责对外部中断请求的管理。

​	下图中，左边是内核的虚拟地址，右边上半部分是内核的物理地址(或者说对应DRAM)，右边下半部分是IO设备。**左边部分（虚拟地址），完全由硬件决定，在硬件设计时就确定了结构**。

![img](https://pic2.zhimg.com/80/v2-a38d82284c87bb98970e671ff9dd53c5_1440w.webp)

​	**在教学操作系统中，操作系统启动时，从虚拟地址`0x80000000`开始运行，该地址也由硬件设计者决定。基本上，主板的设计人员决定了，在虚拟地址翻译成物理地址后，如果物理地址高于`0x80000000`，就会走DRAM芯片（下图CPU下面的DRAM）；如果物理地址低于`0x80000000`，就会走不同的IO设备。这种物理结构的布局，完全由主板设计人员决定。**如果想了解物理结构布局的细节，可以阅读主板对应的手册。

​	上图中，右侧物理地址最下方0地址是未使用的地址，而`0x1000`是boot ROM的物理地址。**当主板通电时，主板第一件做的事情就是运行存储在boot ROM中的代码，当boot完成后，会跳转后物理地址`0x80000000`，操作系统设计者需要保证在该物理地址有一些数据/指令能够启动操作系统**。

​	在boot ROM到操作系统之间的物理地址范围内，还有一些其他IO设备，比如上图中：

+ `0x0C000000`物理地址指向中断控制器（PLIC，Platform-Level Interrupt Controller），`0x02000000`物理地址指向的CLINT（Core Local Interruptor）也是中断控制器的一部分。**基本上，多个设备都可以产生中断interrupt，需要由中断控制器路由将这些中断信号路由到合适的中断处理函数（这些功能都由中断控制器实现）**。
+ `0x10000000`物理地址指向的UART（Universal Asynchronus Receiver/Transmitter），负责与console、显示器交互。 
+ `0x10001000`物理地址指向VIRTIO_disk，该设备负责与磁盘进行交互。

​	<u>当你向上图中`0x02000000`虚拟地址（也映射到相同的物理地址）进行读写指令，实际你是向实现了CLINT的芯片进行读写操作。后续会介绍细节，这里可以先认为是直接和硬件设备进行了交互，而不是读写物理内存</u>。

​	上图的左侧，即XV6的虚拟内存地址空间，当机器当boot启动，还没有可用的page，XV6操作系统会先设置好内核使用的虚拟地址空间page table。为了让教学XV6操作系统简单易懂，大部分虚拟地址到物理地址的映射直接是等同的关系（即虚拟地址值和物理地址一致）。在上图中，低于PHYSTOP所在的虚拟地址`0x86400000`的虚拟地址，与右侧的物理地址值是一样的。

​	<u>上图中，有一些page在虚拟地址的高段位，比如kernel stack，因为在它之前（较低的虚拟地址）有一个未被映射的Guard page，这个Guard page对应PTE条目的Valid标记位bit没有设置（即等于0值）。因此，如果Kernel stack耗尽了，会导致触发page falut（当然这样好过内存越界），这让我们能及时发现Kernel stack出错了。同时我们也不想浪费物理内存给Guard page，所以Guard page只是在高段的虚拟地址中，并且它的PTE没有消耗任何物理内存（毕竟Valid = 0）</u>。在上面的例子中，Kernel stack（Kstack）对应的物理地址被映射了两次，第二次在PHYSTOP下的Kernel data。**通过page table机制，你可以实现虚拟地址到物理地址一对一、一对多、多对一的映射**。XV6有至少1-2个例子用上这个技巧，上图stack和Guard page就是XV6的page table使用技巧的一个例子，主要是为了追踪漏洞。

​	另外一个，关于权限，例如上图左侧Kernel text page被标记为R-X，意味着能在这块虚拟地址内进行读、执行操作，但是不能进行写操作。通过权限设置，能尽早发现bug和避免bug。而上图左侧Kernel data标记RW-，能够对这个虚拟地址进行读写操作，但不能进行执行操作，所以没有设置X标识位。

![img](https://pic4.zhimg.com/80/v2-84540695386e8abb924ffb4d7518a5ff_1440w.webp)

![img](https://pic3.zhimg.com/80/v2-3b17dd0b606c3d8f15c5ebe3442af1ea_1440w.webp)

![img](https://pic1.zhimg.com/80/v2-428206cddd1491b332599cf2f2cf53e8_1440w.webp)

​	由上图手册可知道，物理地址0保留未使用，物理地址`0x10090000`对应以太网(Ethernet MAC)，物理地址`0x80000000`对应DDR Memory。

---

问题：这里说的内存物理结构布局由硬件决定，硬件指的是CPU还是CPU所在的主板？

回答：CPU所在的主板。毕竟CPU只是主板上的一部分，包括DRAM出现在物理内存布局中也只是主板的一部分，且DRAM并不在CPU内。主板设计者将处理器CPU、DRAM，其他许多IO设备汇总在一起。同理，对于操作系统OS而言，CPU只是一部分，设计操作系统时，需要考虑CPU和其他IO设备等硬件。比如当你需要向互联网发送一个报文时，操作系统需要调度网卡驱动（network driver）和网卡（NIC）来完成这工作。

问题：低于`0x80000000`的物理地址，它们并不存在于DRAM中，当我们使用这些地址时，指令会直接走其他硬件对吗？

回答：是的，高于`0x80000000`的物理地址对应DRAM芯片。比如以太网Ethernet对应低于`0x80000000`的一个物理地址`0x10090000`，我们可以对这个叫做内存映射I/O（memory mapped IO）的地址进行读写操作，我们通过load加载和store存储指令，可以为以太网控制器Ethernet Controller编程。

问题：为什么上图中物理地址最上面一块的物理地址被标记为未使用Unused？

回答：物理地址总共有2^56这么多，但是你不需要真的在主板上接入那么多内存设备。正因为一般不会真的在主板上插入那么大且多的内存条，所以物理内存地址最上方总会有用不到Unused的部分。且实际上，在XV6教学操作系统中，我们限制了内存的上限为128MB。

问题：当读指令从CPU发出后，它是怎么路由到正确的I/O设备的？比如说，当CPU要发出指令时，发现地址低于`0x80000000`，但是它怎么将指令送到正确的IO设备？且确保指令不会被发送到其他地方，比如DRAM芯片中？

回答：你可以认为在RSIC-V有一个多路输出选择器（demultiplexer），是由内存控制器路由的。

问题：对于不同的进程会有不同的Kernel stack吗？

回答：是的，每个用户进程都有一个对应的Kernel stack。之后会讲相关知识。

问题：用户进程的虚拟地址会映射到未使用的物理地址空间吗？

回答：如上面左侧虚拟地址有Free Memory，右侧物理地址也有Free Memory，XV6使用Free Memory存放用户进程的page table。如果某个时间点运行特别多用户进程，导致Free Memory耗尽，这时候再执行fork或者exec会返回error。

问题：接上一个问题的回答，这意味着用户进程的虚拟地址空间比内核的虚拟地址空间小得多，对吗？

回答：本质上用户进程和内核的虚拟地址空间大小一样，只是用户进程通常虚拟地址空间使用率更低（即虚拟地址中，实际使用且分配对应物理内存的，占整个可分配的虚拟地址空间比例小）。

问题：如果每个进程都将大块的虚拟地址映射到了同一个物理地址，这里会优化合并这一个mapping映射吗？

回答：XV6不会做这种事情，但是有一个page table实验就是做这个事情。真正的操作系统能做到这点。

## 4.6 页表初始化代码讲解

> [4.6 kvminit 函数 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/270579809) <= 图片出处，这部分主要边调试边讲。
>
> [关于用户进程页表和内核页表_ONIM的博客-CSDN博客_内核页表和用户页表的区别](https://blog.csdn.net/DKH63671763/article/details/104996395/)
>
> [页表到底是保存在内核空间中还是用户空间中？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/493153133) => 有不少优质回答，建议看看
>
> **第一个问题“那如果页表在[内核空间](https://www.zhihu.com/search?q=内核空间&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})中，访问页表不是会陷入内核？”**
>
> 创建和删除[页表](https://www.zhihu.com/search?q=页表&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})的确是在内核空间操作的。页表不能在用户空间进行操作一点都不奇怪，你要知道**页表的作用不仅仅是[虚拟地址](https://www.zhihu.com/search?q=虚拟地址&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})到物理地址的映射，还有关键的权限[访问控制](https://www.zhihu.com/search?q=访问控制&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})和页面属性的记录**。下图是[armv8](https://www.zhihu.com/search?q=armv8&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})中level 1的页表格式，类似于x86中的PUD的结构：
>
> ![img](https://pica.zhimg.com/80/v2-2c8453aa638c3eae1d007e5d4d46bf66_1440w.webp?source=1940ef5c)
>
> 其中权限控制是[内核代码](https://www.zhihu.com/search?q=内核代码&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})区分内核态与[用户态](https://www.zhihu.com/search?q=用户态&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})的基础，如果页表数据让用户进程控制，那区分内核空间和用户空间也没有什么意思了。这样的话的用户进程就能看见很多关键数据，安全性怎么来保证呢？你还要知道用户进程的页表是覆盖整个[地址空间](https://www.zhihu.com/search?q=地址空间&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})的，也就是说，用户进程页表中的内核部分对所有进程来说都是一样的，这怎么能让用户进程去控制呢？
>
> **回答你的第二个问题“这样不会影响效率吗？”**
>
> 第一，就算再多的进程去分配内存，就目前的页表操作数量级来说，相对于硬件中断和[进程调度](https://www.zhihu.com/search?q=进程调度&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})，操作页表是小概率事件。
>
> 第二，你以为在用户进程中分配内存的时候，就马上通过[系统调用](https://www.zhihu.com/search?q=系统调用&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})陷入内核，然后进行页表操作吗？内核如今已经发展的很成熟了，当然不会这么傻。在你兴高采烈的分配好一块内存后，内核只是给你找了一块独一无二的[虚拟内存](https://www.zhihu.com/search?q=虚拟内存&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})空间，并没有映射到物理内存，所以根本没有页表的操作。只有你真正用到你的内存时，MMU发现无法进行虚拟内存到物理内存的转换，只好抛出page fault异常，然后进入内核进行物理内存的分配过程，接着就给你把页表创建好了，这个整个过程叫做**惰性分配**。更重要的是，其实[libc库](https://www.zhihu.com/search?q=libc库&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})在进程创建的时候，就已经把堆空间用内存池的方式管理起来，在进程分配小于128kb的内存时，根本不需要内核进行任何操作，因为堆这个段的虚拟内存早就映射好了物理内存，何谈效率的影响？
>
> **第三个问题“[内核页表](https://www.zhihu.com/search?q=内核页表&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})和普通的页表到底有什么区别？”**
>
> 对于所有进程来说它们页表中的内核空间页表部分都是一模一样的，它们都是从1号进程的init_mm结构中copy的，只有[用户空间](https://www.zhihu.com/search?q=用户空间&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2185333892})的页表不尽相同。用户空间的页表是用来进行不同进程地址空间隔离的，所以相同的虚拟地址可以映射到不同的物理地址，当然一般情况下这也是必须的，而内核只有一个。

​	首先启动XV6操作系统，通过QEMU模拟主板，用gdb调试boot流程。上次调试到main函数，main函数中调用过kvminit，kvminit会设置好kernel的地址空间。

```c
#include "param.h"
#include "types.h"
#include "memlayout.h"
#include "elf.h"
#include "riscv.h"
#include "defs.h"
#include "fs.h"

void vmprint(pagetable_t);

/*
 * the kernel's page table
 */
pagetable_t kernel_pagetable;

extern char etext[]; // kernel.ld sets this to end of kernel code.

extern char trampoline[]; // trampoline.S

/*
 * create a direct-map page table for the kernel.
 */
void kvminit()
{
  // 函数第一步为最高一级的page table directory分配physical page
  kernel_pagetable = (page_table_t) kalloc();
  // 然后将将这段directory的PTE都初始化为0
  memset(kernel_pagetable, 0, PGSIZE);
  // 通过kvmmap函数，将IO设备映射到内，例如这里将UART0映射到内核地址空间。在memlayout.h文件中，可以看到UART0对应值为0x10000000。通过kvmmap，可以将虚拟地址映射到等值的物理地址上。
  // uart registers
  kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W);
  
  // 打印最高级内核page table directory信息。后面内核也是持续用kvmmap设置地址空间
  vmprint(kernel_pagetable);
  
  // 同理，这里对VIRTIO0进行地址映射，下面的kvmmap也是同理。
  // virtio mmio disk interface
  kvmmap(VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
  
  // CLINT
  kvmmap(CLINT, CLINT, 0x10000, PTE_R | PTE_W);
  
  // PLIC
  kvmmap(PLIC, PLIC, 0x400000, PTE_R | PTE_W);
  
  // map kernel text executable abd read-only.
  kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_W);
  
  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel
  kvmmap(TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
  // 打印完整的page table directory信息
  vmprint(kernel_pagetable);
}

// 这个函数首先设置了SATP寄存器，内核告知MMU使用刚刚设置好的page table。
// 这里需要思考的是，在这个函数执行之前，都是使用物理地址，而MMU开始使用该page table后，后续指令中使用的地址都是虚拟地址。
// Switch h/w page table register to the kernel's page table.
// and enable paging.
void kvminithart()
{
  w_satp(MAKE_SATP(kernel_pagetable));
  sfence_vma();
}
```

​	memlayout.h文件部分内容如下：

![img](https://pic4.zhimg.com/80/v2-bcf567bdc67647ca8c7494bdde0ee63f_1440w.webp)

​	管理虚拟内存的一个难点是，一旦执行了类似SATP这样的指令，你相当于把一个page table加载到了SATP寄存器，后续每一个地址都会被设置到的page table所翻译。如果假设这里page table设置错误了，会产生page fault，因为错误的page table会导致虚拟地址可能根本就没法翻译，进而导致内核kernel停止工作并panic。如果page table中有bug，你将会看到奇怪的错误和崩溃（errors and crashes）。

---

问题：`walk`函数，从代码看它返回了最高级L2 page table directory的PTE，但是它是如何工作的？就像其他函数一样，它们期望返回的是PTE而不是物理地址。

回答：从代码中可以看出，`walk`函数返回了page table的PTE，而内核可以读写PTE。比如现在可以把值插入PTE中。

问题：接上一个问题，疑惑为什么需要到第三级page table，然后才返回第一个PTE？

回答：不，实际上返回的是第三级page table directory的PTE。

问题：每个进程都有自己的三级page table directory，用于将虚拟地址映射到物理地址。但当我们将内核虚拟地址映射到物理地址时，我想我们没有考虑内核虚拟地址的页表或者其他进程虚拟地址在哪里？例如页表树和页表树实际指向的物理内存。

回答：当kernel为进程分配一个proc和page table时，它们将被分配到内核中未被使用的虚拟地址（Free Memory），Kernel会编程，对用户进程分配几个page table并且填充PTEs。当Kernel运行这个进程时，它将加载分配给用户进程的页表对应的root physical address或者建立在SATP寄存器中的页表。此时，处理器将在kernel为该进程构造的虚拟地址空间运行。

问题：接上一问回答，所以Kernel会为该进程放弃一些自己的内存。但是理论上，用户进程和Kernel的虚拟空间一样大，实际上根本不是，对吧？

回答：是的，下面是用户级进程的虚拟地址空间布局图（the layout of virtual address space of a user level process）。同样，它从0到MAXVA，和内核地址空间一样。它也同样有自己的页表用于映射kernel为它设置的虚拟地址到物理地址的翻译。

问题：但是用户进程并不能使用所有MAXVA的虚拟地址，对吧？

回答：是的，我们不能。因为内存不足以支持。所以实际上许多用户进程占用的物理内存大小比总的可分配的虚拟地址空间小得多。

![img](https://pic2.zhimg.com/v2-1e94512320719c3f602d7b145be0f1c5_r.jpg)

问题：`walk`函数，在写完SATP寄存器后，内核还能直接访问物理地址吗？在代码中，看起来像是后来都通过page table将虚拟地址翻译成了物理地址。由于SATP寄存器已经被设置了，这时如果再拿到一个本该是物理地址的值，会不会被错认为是虚拟地址？

回答：我们回顾`kvminithart`函数，`kvminit`函数构造了内核地址空间，内核页表最初是一个物理地址，（然后又）被翻译成物理地址，这些实际也是写入到SATP寄存器中的。从那之后，我们的代码都运行在我们构造出来的地址空间中，就像之前的`kvminit`函数一样。`kvmmap`会对每个地址或者每个page调用`walk`函数。

问题：在SATP寄存器设置完之后，`walk`函数是不是还是按照原本的方式工作？

回答：是的。原因是，内核设置了虚拟地址等于物理地址的映射关系。（换言之就是前面"4.5 内核页表(Kernel Page layout)"提到过，内核虚拟地址空间中，比PHYSTOP`0x86400000`低的虚拟地址，都是直接和物理地址进行了等值映射）有很多函数能正常工作，正是因为内核设置了（一部分）等值的虚拟地址到物理地址的映射。

问题：每一个进程的SATP寄存器在哪里？

回答：每个CPU核只有一个SATP寄存器。但是阅读`proc.h`能知道，在每个proc结构体中，有一个指向page table的指针。

问题：3级页表为什么就比单级的大page table好？

回答：在3级page table directory中，你可以保持大量PTE为空不填充值（leave a lot of entries empty）。如果最高级的page table directory中一个PTE为空，那么你完全不需要创建它对应的中间级L1或者最底层L0的page table。这就像是，在虚拟地址空间中，有一大部分地址都不需要有映射一样（前面看到的内核虚拟地址分布图，就有大块的Free Memory）。

问题：前面讲解的`kvminit`函数中，`kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_W);`会不会分配到它不应该分配的内存？

回答：不会。这里`KERNABSE`是内核在内存中的起始位置，而`etext`是内核最后一条指令的位置，所以相减就能算出内核所需要占用的内存大小，而DRAM完全足以支撑这些内存。

## 4.7 实验描述中的一些知识点

> [边信道攻击_百度百科 (baidu.com)](https://baike.baidu.com/item/边信道攻击/7342042?fr=aladdin)
>
> 边信道攻击(side channel attack 简称SCA)，又称侧信道攻击，核心思想是通过加密软件或硬件运行时产生的各种泄漏信息获取密文信息。

+ 直到几年前，不少内核在用户空间、内核空间下使用相同的per-process页表，该页表同时维护用户态、内核态的地址映射，避免用户态和内核态之间切换时还需要切换页表。但这种实现方式，也导致一些边侧信道攻击能够得逞。Until a few years ago many kernels used the same per-process page table in both user and kernel space, with mappings for both user and kernel addresses, to avoid having to switch page tables when switching between user and kernel space. However, that setup allowed side-channel attacks such as Meltdown and Spectre.

# Lecture5 RISC-V的调用协议和栈帧(Calling Convention and Stack Frames)

## 5.1 汇编、指令集概述

​	**处理器并不能直接处理C语言代码，处理器只能处理汇编对应的二进制编码**。

​	当我们提到处理器是RISC-V时，意味着处理器能理解的是RISC-V指令集（instruction set）。**每个处理器都有一个关联的ISA或指令集。每条指令都有一个关联的二进制编码（binary encoding）或者操作码（op code）。当一个处理器运行时，它看到一个特定的编码，然后知道该做什么。**

​	让C程序在处理器上运行的一般流程是，编写C代码，然后编译成汇编语言（assembly，`.S`文件），之后ASM被翻译成二进制文件（object或者说`.o`文件）。

​	这里需要强调的是，课程使用的是RISC-V指令集。如果你用RISC-V指令集，那么你的程序很可能不能在Linux上运行。因为大多数现代计算机运行在x86或x86-64处理器上，它们使用的是另一套ISA。x86拥有一套不同的指令集，尽管看上去和RISC-V指令集相似。Intl和AMD的CPU都实现了x86。

​	<u>RISC-V的"RISC"是精简指令集（reduced instruction set）的意思，而x86通常被称为CISC，复杂指令集（complex instruction set）</u>。

​	RISC和CISC之间有一些关键性的区别：

|            | CISC                                 | RISC                                       |
| ---------- | ------------------------------------ | ------------------------------------------ |
| 指令数量   | 特别多                               | 相对少                                     |
| 指令复杂度 | 大多数指令内部执行多项操作，复杂度高 | 单条指令一般内部操作少，运行的时间周期更短 |
| 是否开源   | 否                                   | 是，市面上唯一的开源指令集                 |

​	当然，两者各有着重的应用场景，没有绝对的优劣之分。

​	在日常中，ARM也是使用RISC指令集。如果你使用安卓手机，那么大概率运行在RISC上。

## 5.2 C语言到汇编代码讲解

​	教程中的C语言代码，经过每个人电脑的编译后得到的汇编代码可能不同。原因有很多，比如编译器优化。

```assembly
.section .text
.global sum_to

/*
	int sum_to(int n) {
		int acc = 0;
		for(int i = 0; i <= n; ++i) {
			acc += i;
		}
		return acc;
	}
*/

sum_to:
	mv t0, a0				# to <- a0
	li a0, 0				# a0 <- 0
 loop:
 	add a0, a0, t0	# a0 <- a0 + t0
 	addi t0, t0, -1	# t0 <- t0 - 1
 	bnez t0, loop		# if t0 != 0: pc <- loop
 	ret
```

​	上面的汇编代码中，首先`mv t0, a0`将a0寄存器的值在t0保存一份，然后`li a0, 0`将a0寄存器的值置0。`loop`进入循环，循环`add a0, a0, t0`将a0寄存器和t0寄存器内的值相加后存入a0寄存器，t0寄存器的值每次减1，直到t0寄存器内的值为0时退出循环。

​	需要注意的是，这里调试的代码都在内核中，不在用户空间中，所以不会在设置断点时遇到麻烦问题。

---

问题：这些汇编代码中，`.global`、`.section`、`.text`是什么意思？

回答：`.global`代表你可以在其他文件中调用这个函数。`.text`代表当前文件内写的是代码code。拓展，如果你对内核感兴趣，编译后，可以查看`kernel.asm`文件，可以看到XV6完整内核的汇编代码。文本中每一行左边的数字代表当前这条指令在内存中哪个位置。

问题：`.asm`和`.S`文件有什么区别？

回答：两者都是汇编文件，但`.asm`文件包含大量的额外注释，而`.S`文件没有。而一般C语言编译后得到的是不包含行号的汇编代码，即`.S`文件。如果想知道怎么获取`.asm`文件，makefile里包含了具体的步骤。

## 5.3 寄存器和汇编

> [5.4 RISC-V寄存器 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/295439950) <= 图片出处

​	**每个CPU核都有一套寄存器，它们被预先定义好用来存储数据。实际上，汇编代码经常操作寄存器的数据，而非只是直接操作内存数据**。因为寄存器比内存快，所以我们更倾向于使用寄存器。

​	所以我们经常看到，汇编代码的模式是，先用`load`或类似的指令将数据从寄存器or内存加载到另一个寄存器，之后在对寄存器中的数据进行运算，如果关注结果值，那还会用`store`或类似的指令将数据存到寄存器或者内存某个地址。

​	通常讨论寄存器时，我们使用它们的ABI name，因为写汇编代码时需要使用它们的ABI name。

![img](https://pic1.zhimg.com/80/v2-b6b20bee8d933246a9692a706f0deab8_1440w.webp)

​	<u>a0～a7寄存器用来存储函数的参数值，如果一个函数有超过8个参数，我们就需要用内存存储参数</u>。（隐含含义，尽量不要定义参数超过8个的函数，否则不得不用内存存储参数）

​	上图中，第四列Saver，Caller表示not preserved across fn call，Callee表示preserved across fn call。比如ra寄存器是Caller类型的，表示函数A中调用函数B得到函数B的返回值时，函数B可以覆盖ra寄存器的值。如果一个寄存器是Callee saved类型的，则被调用的函数需要考虑如何存储这些寄存器的值。

​	由于RISC-V用的寄存器是64bit的，各类数据会被以64bit为单位进行处理，保证能存到寄存器中。比如一个32bit的整数，根据其是否负数，通常会在前面补全32个0或者1，使得这个整数变成64bit存放到寄存器中。

​	提醒一下，上图的f寄存器，即浮点数寄存器，在这节课不会提到，可以先不管。

----

问题：返回值可以放到a1寄存器中吗？

回答：理论上是可以的。如果一个函数的返回值是long long，即128bit类型，我们可以考虑把返回值存到a0~a1寄存器中。

问题：为什么寄存器不是连续的？为什么上图中s1寄存器后续紧接着a0寄存器？

回答：我猜测是因为有一个压缩版的RISC-V指令集，它的大小是16bit，而不是64bit，你可以用压缩指令集使得占用更少的内存。而当你使用16bit指令时，你只能访问寄存器x8～15。我认为s1与s2-11分开，是因为他们想要明确在压缩指令模式下，s1可用，但是s2-11不行。我不知道为什么他们选择x8-15，但是如果看一些代码，会发现这些是最常用的寄存器。

问题：除了帧指针frame pointer，还有栈指针stack pointer，我不认为我们需要更多的Callee Saved寄存器，但是实际上却有很多，这是为啥？

回答：s0-s11都是Callee寄存器，供编译器或程序员使用。在一些特定场景下，你会想要确保一些数据在函数调用之后仍然能保存，这时候编译器可以选择使用s1～s11寄存器。很遗憾我手头没有具体的例子可以进行说明，但我确定会有这种场景以至于需要这么多的Callee寄存器。但基本上都是编译器or程序员会选择使用s1~s11寄存器。

## 5.4 函数调用栈Stack

> [5.5 Stack - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/295440833) <= 图片来源

![img](https://pic3.zhimg.com/v2-1bb012d535c8fbb60510a401c534759e_r.jpg)

​	上图中，每一个区域就是一个stack frame栈帧。每调用一个函数，函数本身会为自己创建一个Stack Frame。函数通过移动Stack Pointer来完成另一次函数调用。上图中sp表示stack pointer的当前位置。**Stack从高地址往低地址方向使用，所以stack总是向低地址延伸**。想创建一个新stack frame时，通过地址做减法而非加法。

​	<u>stack frame for function函数栈帧，包括寄存器register和局部变量local variables。如果argument register用完了，额外的参数就会出现在stack中，所以Stack Frame大小总是不尽相同，尽管上图看上去一致。不同的函数有不同数量的本地变量、寄存器的诉求，所以Stack Frame总是不一样</u>。

​	**能确定的是，Return address总是在Stack Frame第一位（栈顶），而指向前一个Stack Frame的指针，总是在它前一个位置（栈顶往下数第二个位置）**。

​	Stack Frame有两个重要的寄存器：

+ SP（Stack Pointer），指向Stack的底部，表示当前Stack Frame的位置。
+ FP（Frame Pointer），指向当前Stack Frame的顶部。当我们想快速找到Return address或指向前一个Stack Frame的指针，就可以通过FP寄存器快速定位。

​	**我们存储指向前一个Stack Frame的指针，是为了可以回溯跳转。所以当函数返回时，我们可以将它存储到FP寄存器中，以次将FP指向上一个Stack Frame的顶部位置，确保我们总是指向正确的函数**。

​	Stack Frame需要汇编代码创建，你读到的任何约定文档convertions document，都会指出这由编译器完成。即编译器遵循convertion生成了汇编代码，进而创建正确的stack frame。

​	**通常在ASM function汇编函数的开头，你会看到funciton prologue，之后是the body of function，最后才是epilogue**。

​	前面"5.2 C语言到汇编代码讲解"提到的`sum_to`汇编函数，只有函数体，没有其他部分，因为它足够简单，只是个leaf函数，**leaf函数指不调用其他函数的函数**。leaf函数不需要考虑保存自己的Return address或者其他Caller Saved寄存器的值，因为它不调用其他函数。

​	而下面一个例子，`sum_then_double`就不是一个leaf函数，因为它调用了`sum_to`函数。

```assembly
.global sum_then_double
sum_then_double:
	addi sp, sp, -16
	sd ra, 0(sp)
	call sum_to
	li t0, 2
	mul a0, a0, t0
	ld ra, 0(sp)
	addi sp, sp, 16
	ret
```

​		上面可以看到`call sum_to`调用了`sum_to`函数，所以`sum_then_double`函数需要包含prologue。这里我们通过`addi sp, sp, -16`对Stack Pointer减16，这样我们为新的Stack Frame创建了16字节的空间，之后我们将Return address保存在stack中，然后`call sum_to`调用`sum_to`，后续两行将`sum_to`结果值乘以2。最后是epilogue，先将Return address值加载回ra寄存器，再对sp加16以删除刚创建的Stack Frame。最后通过ret返回/退出函数。

​	这里如果删除call之前的prologue，以及删除mul之后的epilogue，会发生什么？如果删掉这两个操作，由于调用了`sum_to`会覆盖ra寄存器的值，进而导致`sum_then_double`最后ret跳转时，按照`sum_to`覆盖的ra寄存器值跳转，陷入无限循环一直重复执行`sum_then_double`中call往后的指令（li、mul、ret）。

---

问题：为什么`sum_then_double`最开始要对sp-16？

回答：这是为了给Stack Frame创建空间，-16相当于地址向前16字节，这样我们有足够空间存Stack Frame，可以在那存储数据。我们并不想直接覆盖原本stack pointer位置的数据。

问题：上一问接着，为什么-16，而不是-4？

回答：是的，我们确实不需要-16那么多，但-4也太少了，至少需要-8，因为接下去要存ra寄存器的是64bit（8字节）。这里习惯用16字节，是因为我们要存Return address和指向上一个Stack Frame的pointer。

## 5.5 C语言Struct的内存结构

​	基本上，C语言的Struct在内存中是一段连续的地址。通常，我们可以将Struct作为参数传递给函数。

---

问题：谁创建了编译器来将C代码转换成各种各样的汇编代码，是不同的指令集创建者，还是第三方？

回答：我认为不是指令集的创建者，通常是第三方创建的。你们常见的两大编译器，一个是gcc，这是由GNU基金会维护的；一个是Clang llvm，这个是开源的，你可以查到相应的代码。当一个新的指令集，例如RISC-V，发布之后，调用约定文档和指令文档一起发布，我认为会指令集的创建者和编译器的设计者之间会有一些高度合作。简言之，我认为是第三方与指令集作者一起合作创建了编译器。RISC-V或许是个例外，因为它是来自于一个研究项目，他们的团队或许自己写了编译器，但是我不认为Intel对于gcc或者llvm有任何输入。

# Lecture6 隔离性和系统调用mode切换(Isolation & System Call Entry_Exit)

## 6.1 user/kernel space切换的Traps机制

​	本节主要讲用户进程在运行时**在用户态、内核态之间切换的Traps机制**。

​	当程序发起系统调用请求(make a system call)或者遇到诸如缺页(page fault)、除0(divide by zero)等fault，或者一个设备触发了中断(decides to interrupt)需要得到内核设备驱动(kernel device driver)的响应，这里就会发生<u>用户态和内核态之间的切换，对应的机制即Traps</u>。

​	traps的实现细节加强了操作系统的**隔离安全性(isolation security)**、**性能(performance)**。因为系统调用、缺页等高频事件经常需要从用户态切换到内核态，所以traps实现需要尽量简单且高效。

​	在用户态到内核态的切换时，我们需要关注硬件的状态，因为很多工作都是硬件从执行用户代码(user code)切换到执行内核代码(kernel code)。其中比较需要注意的是CPU的32个用户寄存器(user register)，<u>其中程序计数器寄存器（program counter register）表明当前是内核态还是用户态</u>；前面虚拟内存章节的SATP寄存器包含指向page table directory物理内存地址的指针；**STVEC寄存器，指向了内核处理traps的指令的起始地址；SEPC寄存器在trap过程中保存PC程序计数器的值**。

​	显然，我们需要对这些表示状态的寄存器进行一些操作，才能切换到执行内核中的C程序（比如系统调用）。

​	在trap的起初阶段，CPU的所有状态都被设置成在执行user code，而非kernel code。**在切换到内核态前，我们需要先保存32个用户寄存器，以便后续恢复用户进程状态**，尤其是用户程序时不时会被其他设备硬件中断而暂时暂停执行。<u>我们希望内核在用户进程无感知的情况下完成响应中断，然后再恢复用户程序的执行</u>。

+ 由于寄存器数量就那么多，我们在内核中执行代码时还是需要占用32个用户寄存器，所以我们需要在别处存储原本用户进程的32个用户寄存器的值。
+ 并且在内核态执行时，需要将mode切换成supervisor mode，因为我们需要在内核中执行各种特权指令。
+ SATP寄存器原本存的user table page directory，并不包含整个内核数据的内存映射。运行kernel code之前也需要切换页表。
+ 另外，我们需要SP堆栈寄存器指向内核某个地址，毕竟后续要用一个栈帧(stack frame)执行内核的C代码函数。
+ ....

​	一旦这些前置工作都准备好，就可以切换到内核中执行C代码，后续整体流程就和在执行用户代码无差别。

​	**操作系统出于安全、隔离性考虑，帮助我们抽象了这些前置工作，避免用户程序直接参与user/kernel的切换，破坏系统安全性。这意味着traps机制涉及到内核、硬件操作，并且不能依赖用户空间(user space)的任何事物**。比如不能依赖32个用户寄存器，它们可能存储恶意数据，XV6的trap机制不会查看这些寄存器而仅仅是将它们的值保存起来。与隔离性同样重要的是，**需要对用户代码透明**，在trap机制运行时，用户代码应该无感知。

## 6.2 supervisor mode简述

​	当mode从user mode切换到supervisor mode时，实际上享有的特权也没有很夸张。我们来看看supervisor mode能做什么？

+ 能够读写寄存器（SATP寄存器、STVEC寄存器、SEPC寄存器、SSCRATCH寄存器等）
+ **仅**能够使用`pte_u`标记位=0的PTE。（`pte_u` flag=1表明user code可以使用该PTE）

​	**上面两点就是supervisor mode能做的事情了，其他额外的事情做不了**。比如supervisor mode code不能读写arbitrary physical address，它同样需要通过page table访问内存。如果一个虚拟地址不不在当前SATP寄存器指向的page table中，或者对应PTE的`pte_u`标识=0，那么supervisor mode不能访问这个虚拟地址。

## 6.3 Trap代码执行流程简述

> [6.2 Trap代码执行流程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/318442190) <= 图片来源
>
> [内存映射文件_百度百科 (baidu.com)](https://baike.baidu.com/item/内存映射文件/7167095?fr=aladdin)
>
> 内存映射文件，是由一个文件到一块内存的映射。Win32提供了允许应用程序把文件映射到一个进程的函数 (CreateFileMapping)。内存映射文件与[虚拟内存](https://baike.baidu.com/item/虚拟内存/101812?fromModule=lemma_inlink)有些类似，通过内存映射文件可以保留一个[地址空间](https://baike.baidu.com/item/地址空间/1423980?fromModule=lemma_inlink)的区域，同时将[物理存储器](https://baike.baidu.com/item/物理存储器/7414328?fromModule=lemma_inlink)提交给此区域，内存文件映射的物理存储器来自一个已经存在于磁盘上的文件，而且在对该文件进行操作之前必须首先对文件进行映射。**使用内存映射文件处理存储于磁盘上的文件时，将不必再对文件执行[I/O操作](https://baike.baidu.com/item/I%2FO操作/469761?fromModule=lemma_inlink)，使得内存映射文件在处理大数据量的文件时能起到相当重要的作用**。
>
> 内存映射文件是由一个文件到进程地址空间的映射。Win32中，每个进程有自己的地址空间，一个进程不能轻易地访问另一个进程地址空间中的数据，所以不能像16位Windows那样做。Win32系统允许多个进程（运行在同一计算机上）使用内存映射文件来共享数据。**实际上，其他共享和传送数据的技术，诸如使用SendMessage或者PostMessage，都在内部使用了内存映射文件**。
>
> 这种函数最适用于需要读取文件并且对文件内包含的信息做[语法分析](https://baike.baidu.com/item/语法分析?fromModule=lemma_inlink)的应用程序，如：对输入文件进行语法分析的彩色语法编辑器，编译器等。
>
> 把文件映射后进行读和分析，能让应用程序使用内存操作来操纵文件，而不必在文件里来回地读、写、移动[文件指针](https://baike.baidu.com/item/文件指针?fromModule=lemma_inlink)。
>
> [内存映射文件(memory-mapped file)能让你创建和修改那些大到无法读入内存的文件。_UCJJff的博客-CSDN博客](https://blog.csdn.net/UCJJff/article/details/2404555)
>
> 下面这些材料很详细，感兴趣的可以看看
>
> [Memory-Mapped Files | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/io/memory-mapped-files?redirectedfrom=MSDN)
>
> [Managing Memory-Mapped Files | Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/ms810613(v=msdn.10))
>
> [Memory-Mapped File Information - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/psapi/memory-mapped-file-information)
>
> [What is Memory-Mapped File in Java? - GeeksforGeeks](https://www.geeksforgeeks.org/what-is-memory-mapped-file-in-java/) <= 包括java实际操作代码
>
> Memory-mapped files are casual special files in Java that help to access content directly from memory. Java Programming supports memory-mapped files with java.nio package.  Memory-mapped I/O uses the filesystem to establish a virtual memory mapping from the user directly to the filesystem pages. It can be simply treated as a large array. Memory used to load Memory-mapped files is outside of Java Heap Space.
>
> **Merits of the memory-mapped file are as follows:**
>
> - **Performance**: Memory-mapped Files are way faster than the standard ones.
> - **Sharing latency:** File can be shared, giving you shared memory between processes and can be more 10x lower latency than using a Socket over loopback.
> - **Big Files:** It allows us to showcase the larger files which are not accessible otherwise. They are much faster and cleaner.
>
> **Demerits of the memory-mapped file are as follows:**
>
> - **Faults**: One of the demerits of the Memory Mapped File is its increasing number of page faults as the memory keeps increasing. Since only a little of it gets into memory, the page you might request if isn’t available into the memory may result in a page fault. But thankfully, most operating systems can generally map all the memory and access it directly using Java Programming Language.

​	后续会通过gdb跟踪代码，看trap如何进入内核空间，这里先绘图讲解trap机制的代码执行大致流程。

![img](https://pic1.zhimg.com/80/v2-26e8cea84bf7a60d10670768383aa48c_1440w.webp)

​	在Shell中请求`write`系统调用，对于Shell来说实际是执行`write()`这个C语言函数，而该**函数内部通过ECALL来执行系统调用**。ECALL指令会切换到supervisor mode的内核中。

（1）ECALL切换到supervisor mode的kernel后，第一条指令的指令是汇编语言的函数`uservec`。这个函数是内核代码`trampoline.S`文件的一部分。

（2）在`uservec`汇编函数中，jump跳转到C语言函数`usertrap`中。这个函数在`trap.c`文件中。

（3）`usertrap`函数中执行了`syscall`函数，该函数根据系统调用编号索引到具体的内核系统调用函数(which looks at the system call number in a table and calls the particular function inside the kernel)

（4）根据系统调用编号找到的`sys_write`被执行，将需要输出的内容输出到console。

（5）`sys_write`执行完毕后，返回到`syscall`函数。为了返回到用户空间(user space)，还需执行`usertrapret`函数，该C函数完成了一部分从内核返回到用户空间的工作，`usertrapret`函数也在`trap.c`文件中。

（6）剩下一部分从内核到用户空间的工作，只能由`trampoline.S`文件中的汇编语言`userret`函数完成。

（7）在`userret`汇编函数中，执行机器指令(machine instruction)返回到用户空间，并恢复ECALL之后的用户程序执行。

---

问题：`vm.c`运行在什么mode？

回答：`vm.c`中的所有函数都是kernel的一部分，所以都运行在supervisor mode。

问题：这些函数如此命名是有什么规则or含义吗？

回答：supervisor mode下执行的，都是以`s`开头。

问题：`vm.c`中的函数不是直接访问access物理内存physical memory吗？

回答：是的，这些函数能这么做是因为内核在page table已经小心地设置好了PTEs。每当内核试图读写一个虚拟地址时，会通过kernel page table将其翻译成等值的物理地址（也就达到了直接访问物理地址的效果），然后再进行读写。kernel page table方便在kernel中使用这些(虚拟地址到物理地址的)映射关系，但直到trap机制真正切换到内核之前，kernel page table显然是不可用的（不可访问）。在trap完成切换之前，我们使用的仍然是user page table。

**问题：`read`和`write`系统调用，相比直接的内存读写，还需要模式切换，性能代价更高。有没有可能在打开一个文件时直接获取page table mapping而不是返回文件描述符？这样就能往映射到设备的地址上读写，并且为了隔离性可以考虑类似文件描述符能设置只读只写一类的，在PTE上增加类似的标志位。这样就不需要用户态/内核态之间切换来操作文件了。**

**回答：实际上，很多操作系统都提供了内存映射文件(Memory-mapped file access)机制。这种机制能够将用户虚拟地址和文件内容通过page table完成映射，这样就能直接通过内存读写文件。**在后续的mmap实验你们就需要实现这个机制。对于大多数程序而言，这样确实比`read/write`系统调用效率高。

## 6.4 跟踪XV6系统调用代码执行流程

> [6.3 ECALL指令之前的状态 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/318448376) <= 这节主要都是调试流程，详细可以看该博主文章。下面只写一些我自己觉得相对重要的知识点。
>
> [6.5 uservec函数 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/318458722)
>
> [6.6 usertrap函数 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/318464618)

​	本节主要通过跟踪XV6中Shell请求write系统调用到后来操作系统输出的流程。

​	在Shell中调用的write实际只是关联到Shell的一个库函数，可以在`usys.S`查看该库函数的源代码。

```assembly
# usys.S 文件内代码片段
Write:
	# 将 数字SYS_write(也就是16) 加载到 a7寄存器。告诉 kernel要执行 编号16的系统调用。16对应的就是write系统调用
	li a7, SYS_write
	# ecall后，即跳转到 kernel
	ecall
	# kernel完成工作后，返回用户空间，往下执行ecall往下的指令，这里直接就是ret。后续就返回到Shell中了，因为ret从write库函数返回到了Shell中。
	ret
```

​	<u>在调试过程中，在ecall指令处打断点（当前状态即正要执行ecall指令之前），可以注意sp和pc指向的位置，都是接近0的位置，用户空间的所有地址通常都在低地址，验证了这时仍在用户空间下执行代码。后续如果跳转到内核空间，由于内核都是用高地址，就能观察到sp和pc指向的值明显变化</u>。

​	**在系统调用的时间点，会有大量状态（寄存器值）发生变化，在切换内核之前最重要的且仍需依赖的状态即当前的page table**。

​	这里通过QEMU打印当前用户的page table，看到6条PTE（与shell的指令、数据有关），然后**还有一个invalid page用做stack guard page，以防止Shell尝试使用过多的stack page**。

![img](https://pic2.zhimg.com/80/v2-08e159f806646ae682ebf3246a6edf15_1440w.webp)

​	上图第三行PTE的attr标记位`rwx`表示可读、可写、可执行，但是u标识位0，而**用户空间只能使用u标识位1的PTE**。<u>其他标记位比如a（Access）表示当前PTE是否被使用过，d（Dirty）表示当前PTE是否被写过</u>。

​	上图中，最后两条PTE有很高的虚拟地址，非常接近虚拟地址的顶部。如果看过XV6的书，就会知道这两个PTE分别对应trapframe page和trampoline page，它们都没有u标识，用户不能访问这两个PTE，但是切换到supervisor mode后就能访问了。

​	<u>对于这里的page table，需要注意的是，它没有包含任何kernel的映射关系，没有到kernel data、kernel insructions的任何物理地址映射。而最后两条PTE（虽然没有u标识），基本上是为了用户代码执行而创建的，对在kernel中执行代码没有什么直接的特殊作用</u>。

​	**这里接着断点位置执行ecall指令，执行前我们还在用户空间，执行后我们就在内核空间了**。ecall执行完后，查看pc程序计数器，之前指向低虚拟地址（0xde6），而现在指向一个高虚拟地址（`0x3ffffff000`）。

​	此时通过QEMU查看page table，发现page table和之前一致，且发现我们pc指向的即trampoline page，这些指令是内核在supervisor mode中将要执行的最开始几条指令，即前面"6.3 Trap代码指令流程简述"中提到的trap机制最开始要执行的几条指令。

​	此时观察寄存器的值，发现里面都还是之前的用户程序的数据。现在初步切换到内核中，我们需要将这些寄存器的值保存到某处，以免后续内核执行完毕返回用户空间时无法恢复用户程序当时的状态值。

​	我们此时在`0x3ffffff000`，处于trampoline page，这个page内包含了内核的trap处理代码。

​	**正如我们所见，ecall并不会切换page table**。**所以这也意味着trap处理代码必须存在于每一个用户页表中，因为ecall不切换page table，我们需要在user page table中某个地方执行kernel最初的代码。而trampoline page，是由内核小心地映射到每个user page table中的，使我们仍在user page table中时，内核也能在此执行trap机制的最开始一些指令。**

​	**（pc程序计数器跳转到user page table中的`0x3ffffff000`这一虚拟地址）这由STVEC寄存器完成控制，这是一个只能在supervisor mode下读写的特权寄存器。在从内核空间进入到用户空间之前，内核会设置好STVEC寄存器指向内核希望traps代码运行的位置**。查看寄存器当前情况，确实会发现STVEC寄存器当前值为`0x3ffffff000`，也就是trampoline page的起始位置。<u>正因为STVEC寄存器的内容是`0x3ffffff000`，在ecall指令执行之后，pc程序计数器就跳转到了`0x3ffffff000`（trampoline page）</u>。

​	需要注意的是，即使trampoline page是映射到user address space中的user page table，user code也不能修改这条PTE，因为PTE的u标识位是0（用户代码只能访问u标识位1的PTE，而前面"6.2 supervisor mode简述"提过supervisor则只能访问u标识位0的PTE）。正因trampoline page的PTE无法被用户代码影响，所以trap机制是安全的。

​	**虽然强调了现在是supervisor mode了，但是我们没有手段直接确认我们处于何种mode。不过我们通过观察，可以发现pc程序计数器在执行trampoline page内的代码，而该PTE没有设置`pte_u` flag。只有supervisor mode执行代码，才能保证访问`pte_u`为0的PTE不崩溃或报错，所以推导出我们当前必然在supervisor mode**。

​	实际上，ecall只会改变3个事情：

（1）ecall改变mode从user mode 到 supervisor mode。

（2）ecall将pc程序计数器的值保存到SEPC寄存器中。

（3）ecall会跳转到STVEC寄存器指向的指令（即将PC寄存器设置成控制寄存器STVEC的内容）。

​	虽然ecall帮我们做了一部分工作，但我们到执行内核C代码之前还有其他要做的工作，比如：

+ 需要保存32个用户寄存器的内容，后续如果从内核切换回用户空间，才能保证用户代码正常执行。
+ 需要切换到 kernel page table（目前仍在user page table中）。
+ 需要创建或找到一个stack，将SP寄存器指向kernel stack，后续给C代码提供stack。
+ 需要跳转到内核中C代码的某处合理位置

​	<u>ecall不会做上述任何一件事，当然我们也可以编程硬件使ecall完成上述工作，而不是让软件完成。实际上，确实有一些机器在系统调用时，会在硬件层面完整上诉这几点工作</u>。但是教学的RISC-V并不会，ecall在RISC-V中只完成尽量少且必须完成的工作，其他的需要软件实现。RISC-V如此设计的初衷是为软件开发者提供最大的灵活性，操作系统开发者可以根据喜好自行设计操作系统程序。

+ 为何RISC-V的ecall不直接做保存用户寄存器的工作

  或许在一些系统调用过程中，有些寄存器不需要保存。哪些需要/不需要保存，取决于软件、编程语言和编译器。通常不保存所有32个用户寄存器或许可以节省大量时间。所以你不希望ecall强迫保存所有用户寄存器值。

+ 为何RISC-V的ecall不直接做page table切换的工作

  或许某些操作系统可以在不切换页表的情况下执行部分系统调用（因为切换page table成本高），如果ecall强制完成了切换页表的操作，那就不能实现这类不需要切换page table的系统调用了。还有些操作系统同时将user和kernel的虚拟地址映射到同一个page table，这样user和kernel之间切换时根本就不需要切换page table。

+ 为何RISC-V的ecall不将SP寄存器指向kernel stack

  对于某些简单的系统调用或许根本不需要任何stack。对于某些关注性能的操作系统，肯定希望ecall不自动做stack切换。

​	出于性能考虑，ecall做的事情越简单越好（越少越好）。

​	回到我们讨论的点，ecall执行完后我们进入kernel了。<u>此时需要做的第一件事就是保存寄存器内容</u>，在RISC-V上，大多数操作离不开寄存器。其他机器可能直接将32个用户寄存器写到物理内存某处，而RISC-V不能这么做，因为在RISC-V中，supervisor mode的代码不能直接访问物理内存。虽然在XV6没有这么做，但是有一种做法是直接将SATP寄存器指向kernel page table，之后我们就可以直接使用所有kernel mapping来辅助存储用户寄存器。这时合法操作，因为supervisor mode可以修改SATP寄存器。然而在trap代码的当前位置（目前就是trap的起始位置），我们不知道kernel page table的地址，并且更改SATP寄存器的指令要求写入SATP寄存器的内容来自另一个寄存器。

​	ecall后第一步——保存寄存器内容，XV6在RISC-V上的实现包括两个部分：

（1）XV6在每个user page table映射了trapframe page，这样每个进程有自己的trapframe page。这个page存了各种数据，其中最为重要的即用来保存32个用户寄存器的预留内存空槽。trap执行到这里，kernel事先已经为每个user page table设置了trapframe page PTE（其指向的内存可以用来存储当前进程的用户寄存器内容），该虚拟地址固定为`0x3ffffffe000`。

（2）在进入user space之前，kernel会将trapframe page的地址写入SSCRATCH寄存器（`0x3fffffe000`）。SSCRATCH寄存器的作用就是保存另一个寄存器的值，并将自己的值加载到另一个寄存器（RISC-V的csrrw指令运行交换任意两个寄存器的值）。实际上`trampoline.S`第一行之行的指令`csrrw a0,sscratch,a0`就将a0寄存器和sscratch寄存器的值进行了交换。

---

​	下面是`proc.h`的trapframe结构体，有很多unit64字段用于存储对应的用户寄存器值。前5个是kernel事先存放在trapframe中的数据。比如第一个`kernel_satp`存储kernel page table 指针值（地址值）。

![img](https://pic3.zhimg.com/80/v2-6f4f910813bb76ad01f471221f89e6ba_1440w.webp)

​	在trampoline code的开头（即`uservec`函数），我们可以看到它做的第一件事情就是执行csrrw指令（这里交换了a0和sccratch寄存器的值）。通过交换寄存器值，我们将a0寄存器的值保存起来了，并且拿到sccratch寄存器存的指向trapframe page的指针，下面30多行代码通过sd指令将用户寄存器的值写入trapframe page指向的内存的不同偏移量位置。

![img](https://pic2.zhimg.com/80/v2-b789d1f4c42b4dd6153c38ee48653ed5_1440w.webp)

​	`ld sp, 8(a0)`将a0寄存器值往后偏移8字节的数据加载到SP寄存器，查`proc.h`的trapframe结构体，可以得知这里就是将a0寄存器指向的trapframe page的kernel_sp的值存到SP寄存器。`kernel_sp`的值在内核进入用户空间之前就被设置成kernel stack。所以这条指令作用就是初始化SP寄存器，使其指向当前进程的kernel stack的顶部(point to the top of this process's kernel stack)。打印SP寄存器值，会发现是一个高地址（当前进程的kernel stack），因为XV6在每个kernel stack下面设置了一个guard page。

​	下一条指令`ld t0,32(a0)`同理查`proc.h`的trapframe结构体，得知将`kernel_hartid`的值写入到了tp寄存器。在RISC-V中，没有一个直接的方法用于确认当前运行在多核处理器的哪个核上，XV6会将CPU核的编号(hardid)保存到tp寄存器中。内核通过这个指确定某个CPU核上运行了哪些进程。这里打印TP寄存器的值会发现是0，因为我们之前配置QEMU模拟器只给XV6分配和一个核。

​	下一条指令`ld t0,16(a0)`，同理查`proc.h`的trapframe结构体，得知将`kernel_trap`的值写入t0寄存器，这个值是我们将要执行的第一个C函数`usertrap()`的指针。

​	再往下`ld t1,0(a0)`，同理查`proc.h`的trapframe结构体，得知将`kernel_satp`的值写入t0寄存器，即写入kernel page table的地址。

​	再往下`csrw satp,t1`，将t1的值写入satp，即切换到kernel page table。

​	到这里为止，我们的SP已经指向了kernel stack，并且SATP指向了kernel page table。我们已经可以准备执行内核中的C代码了。

​	**这里有个问题，为什么切换到kernel page table后，程序还能正常执行(没有崩溃)，PC当前指向的虚拟地址之后不会通过kernel page table寻址走到一些无关的page中？因为在kernel page table和之前在user page tabe中，当前正在执行的代码所在的trampoline page对应的mapping是一致的。这两个page table其他的所有mappings都是不同的，只有trampoline page映射是一样的。**trampoline page之所以这么命名，是因为它某种程度进行了"弹跳"，从用户空间到了内核空间。

![img](https://pic1.zhimg.com/80/v2-8340790fb87af0d2e282037831d36368_1440w.webp)

​	最后一条指令`jr t0`执行后，从trampoline跳转到t0指向的内核C代码`usertrap()`函数中。`usertrap`函数是`trap.c`文件中的一个函数。**`usertrap`就像一个trampoline page似的，有很多情况会进到`usertrap`函数中，比如系统调用、除0、使用未被映射的虚拟地址（page fault）、device interrupts等**。

​	usertrap某种程度上存储并恢复硬件状态，但它也需要检查触发trap的原因，以确定相应的处理方式。后续sertrap执行过程中会看到这两个行为。

​	usertrap做的第一件事情就是更改STVEC寄存器的值。**XV6处理trap的方式不同，取决于trap来自user space还是kernel space。我们这里只讨论trap是由user space发起时发生的事情**。*如果trap在kernel space中运行，由于本身仍处于kernel table page，所以很多user space需要做的事情都不需要了*。这里代码`w_stvec((uint64)kernelvex);`先将STVEC指向了kernelvec变量，这是在内核空间中，而不是用户空间中trap处理代码的位置(which is the kernel trap handler，而不是user trap handler)。

​	接下来我们要保存user PC，它仍然保存在SEPC寄存器中。如果程序在内核执行中时，我们切换到另一个进程并进入另一个进程的用户空间，这时如果该进程又请求系统调用，就可能导致SEPC寄存器的内容被覆盖。所以，我们需要保存当前进程的SEPC寄存器到一个与该进程关联的内存中（这里用trapframe保存user PC），保证数据不会被覆盖。

​	接下来我们要找出我们出现在usertrap函数的原因（前面提到多种情况会进入usertrap），**根据触发trap的原因，RISC-V的SCAUSE寄存器会有不同的数字，数字8则表示我们在trap代码中是因为系统调用**。这里可以看到SCAUSE寄存器的值就是8。

​	下面的if中第一件事确认是不是有其他进程杀kill了当前用户进程（如果有，则异常退出），这里我们Shell没有被kill，继续往下执行。**在RISC-V中，存储在SEPC寄存器中的PC程序计数器，是用户程序触发trap的指令的地址。但是当我们后续恢复用户程序时，我们希望在下一条(用户)指令恢复执行，所以对于系统调用，我们对保存的用户程序计数器+4，这样我们会在ecall的下一条指令恢复执行，而不是重复执行ecall指令**。

​	再往下`intr_on()`，**XV6会在处理系统调用的时候设置允许被中断，这之后如果有其他地方发起中断，XV6也可以考虑先处理其他中断，因为有些系统调用需要占用大量时间。中断总是会被RISC-V的trap hardware关闭，所以这里我们需要显式地打开中断**。

​	往下执行`syscall()`，该函数定义在`syscall.c`文件中，它的作用是查找syscall的表单，根据系统调用编号查找对应的系统调用函数。如果你记得之前的内容，Shell调用的write函数将a7设置成了16（对应系统调用sys_write，无参的C函数，在`sysfile.c`中）。**需要注意的是，系统调用需要找它们的参数，前面write函数的参数分别是文件描述符2，写入缓存的buffer pointer，写入长度2，而syscall函数直接通过trapframe来获取这些参数**。

​	现在syscall中系统调用真正执行了，之后`sys_write`返回，在`syscall.c`中的`p->trapframe->a0 = syscalls[num]();`，这里用trapframe中的a0寄存器接收返回值。**所有的系统调用都有一个返回值，比如`sys_write`返回写入的字节数，而RISC-V上的C代码习惯/约定(conversion)是不管调用什么函数，都将函数的返回值存储在a0寄存器中**。当我们后面返回到用户空间时，会发现trapframe中a0槽位的字段值被写回到实际的a0寄存器中。Shell会认为a0寄存器中的数值是write系统调用的返回值。

​	从syscall函数返回之后，我们回到`trap.c`中的`usertrap`函数，再次检查当前用户进程是否被kill，因为我们不想恢复一个被杀掉的进程。当然，Shell在目前没有被kill。

​	**最后usertrap调用了usertrapret函数，来处理返回到用户空间之前，内核需要做的工作**。在`trap.c`中可以查看usertrapret具体执行的逻辑。

​	<u>usertrapret中，它首先执行`intr_off`停止响应其他地方发起的中断请求，确保在更新STVEC指向到用户空间的trap handler时，仍在内核中执行代码。因为如果在此时处理其他地方的中断，程序会走向用户空间的trap handler代码，即使这时我们仍在内核中，出于其他细节原因，会导致内核错误</u>。

​	往下，`w_stvec(TRAMPOLINE + (uservec - trampoline))`设置STVEC寄存器指向trampoline，在那里执行的代码最后最执行sret指令返回到用户空间。**位于trampoline代码最后的sret指令会重新打开中断`intr_on`，这样后续执行用户代码时就能继续正常响应中断请求了**。

​	再往下几行向trapframe的slots填充数据，对后面执行trampoline代码很有帮助。

```c
// set up trapframe values that uservec will need when
// the process next re-enter the kernel.
p->trapframe->kernel_satp = r_satp(); // kernel page table
p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
p->trapframe->kernel_trap = (uint64)usertrap;
p->trapframe->kernel_hartid = r_tp(); // hartid for cpuid()
```

​	当usertrapret准备trap时，我们在trapframe准备好了下一次需要的值。下一次trap时从用户空间到内核的过度。我们在sstatus控制寄存器中设置一些东西。sstatus寄存器的SSP bit位控制sret指令的行为，这里设置0表示下次执行sret需要切换到user mode了（而不是切换到supervisor mode）。sstaus寄存器的SPIE bit位控制我们后续进入用户空间后，希望打开中断（即允许响应中断请求），这里设置1。

​	我们在trampoline代码最后执行了sret指令，sret会将PC寄存器设置成SEPC寄存器中的值，所以现在我们将SEPC的值设置成用户程序计数器的值（保证后面PC指向用户程序）。

​	后续，我们需要**切换kernel page table到user page table，这也需要在trampoline汇编代码中执行，因为只有trampoline的PTE映射在user space和kernel space都是一样的**。

​	由于我们还在C代码usertrapret函数中，所以需要将page table指针提前准备好（后续才能跳转到trampoline汇编代码完成kernel page table切换到user page table）。

​	usertrapret函数倒数第二行`uint64 fn = TRAMPOLINE + (userret - trampoline)`，算出要跳转到的trampoline汇编代码函数`userret`函数的位置，这个函数包含所有能将我们带回到用户空间的指令。

​	usertrapret最后一行，将fn作为函数指针执行对应的函数，传入TRAPFRAME，stap作为参数（分别存到了a0和a1寄存器）。

​	程序跳转到trampoline汇编代码的userret函数。userret会将user page table存储到SATP寄存器中。正如前面多次强调，因为trampoline在user space、kernel space的PTC mapping一样，所以程序还能正常执行。

​	....

​	后续将之前保存的用户寄存器的值重新载入用户寄存器。

​	在trampoline倒数第二行，返回用户空间之前，执行`csrrw a0,sscratch,a0`，交换a0和sscratch寄存器的值。原本a0寄存器内是trapframe的地址，执行完这个指令后，a0寄存器变成持有系统调用的返回值，而SSCRATCH持有trapframe的地址，也就是内存的最后一页。之后SSCRATCH会一直存储着trapframe的地址，直到某个程序执行了trap。

​	最后执行sret，切换回用户模式。sret会将sepc的值复制到pc。此时从write系统调用返回到Shell中了。

![img](https://pic3.zhimg.com/80/v2-8d88bf3f994f32df450d5ca79cabaf0e_1440w.webp)

![img](https://pic1.zhimg.com/80/v2-3a09709d9f658c2255fe6a9ab1341224_1440w.webp)

---

问题：PTE中的a标识位是什么意思？

回答：a表示当前这条PTE管辖的地址范围是否曾被访问过（Access）。d（Dirty）表示PTE是否被写指令访问过。**这些标识位由硬件维护以方便操作系统使用**。对于比XV6更复杂的操作系统，当剩余物理内存吃紧时，我们需要释放这些页表，方式可能是将一些页表对应的内存数据写入到磁盘，同时将PTE设置为无效，以释放物理内存。这里有很策略让操作系统抉择哪些页表能够被释放，比如我们通过看a标识位判断这个PTE最近是否被使用过，如果它没有被使用或最近一段时间没被使用，那么可以将PTE对应的内存数据保存到磁盘，释放该page对应物理内存。

问题：csrrw指令干啥用的？`csrrw a0,sscratch,a0`

回答：后续会介绍，这条指令交换了寄存器a0和sscratch的内容（将a0寄存器的值保存到sscratch，同时又将sscratch内的数据保存到a0寄存器中）。这之后，内核就可以任意的使用a0寄存器了。这指令很重要，使得内核中的trap代码能够在不使用任何寄存器的前提下做任何操作（因为要避免修改用户寄存器导致后续切换回用户空间时，用户进程由于部分寄存器值被修改导致执行错误）。

问题：当sscratch寄存器和a0寄存器进行交换时，trapframe的地址是怎么出现在sscratch寄存器中的？

回答：在内核前一次切换回user space时，kernel会设置sscratch的内容为`0x3fffffe000`，也就是trapframe page的虚拟地址。所以当我们运行用户程序Shell时，sscratch寄存器保存的就已经是指向trapframe的地址。而之后Shell执行了ecall指令，跳转到了STVEC寄存器存储的地址，即trampoline page。在trampoline page中第一条指令就是交换a0和sscratch寄存器的内容，后续a0寄存器也就存储了指向trapframe的指针。

问题：接上一个问题回答，这流程发生在进程创建的过程中吗？这个sscratch寄存器在哪里？

回答：它是CPU上一个特殊的寄存器。而内核什么时候设置sscratch寄存器的值，这比较复杂。在`trampoline.S`中最后执行的代码，是内核返回用户空间之前执行的最后两条指令——`csrrw a0,sscratch,a0` ，以及下一行`sret`。在内核返回用户空间前，会恢复所有之前保存的用户寄存器，然后再执行`csrrw`恢复a0寄存器的值（sscratch寄存器则又变回存储指向trapframe page地址的指针）。而最后的`sret`返回到user spage。你或许会好奇a0如何有trapframe page的地址，查看`trap.c`代码，可以看到最后执行的C代码`((void (*)(uint64,uint64))fn)(TRAPFRAME, satp)`，而C代码中函数的第一个参数会被存到寄存器a0中，所以a0里的数值是指向trapframe的指针。这里`fn`函数就是之前位于`trampoline.S`的代码。

问题：当你启动一个进程，之后进程在某个时间点执行了ecall指令，那么在什么时候会执行上一个问题中提到的fn函数呢？因为这里是进程第一次调用ecall指令，这进程之前应该没有调用过fn函数吧。我不明白你说的usertrapret是什么。

回答：一台机器总是从boots up in the kernel。**任何时候，从kernel到user space的唯一方法就是执行sret指令**。`sret`指令是`RISC-V`定义用来从supervisor mode切换到user mode的指令。所以，在任何用户代码执行之前，kernel会执行fn函数，并设置好所有东西，比如SSCRATCH、STVEC寄存器。

问题：当我们在汇编代码中执行ecall指令，是什么触发了trampoline代码的执行。是CPU的从user到supervisor的标志位切换吗？

回答：在我们的例子中，Shell在用户空间执行了ecall指令。**ecall设置当前mode为supervisor mode，保存PC程序计数器到SEPC寄存器，将PC寄存器设置成控制寄存器STVEC的内容**。STVEC是内核在进入用户空间之前设置好的众多数据之一。内核会事先就将STVEC设置成trampoline page的起始位置，所以当用户程序执行ecall指令时，ecall拷贝STVEC的值到PC寄存器，之后程序执行会执行程序计数器指向的地址，即trampoline page对应内存的代码。

问题：ecall之后，trampoline page指向的代码将32个用户寄存器保存到了trapframe page。**为什么这里要用另一块内存（trapframe page）来存储用户寄存器，而不是直接使用程序堆栈(program stack)？**

回答：两个问题，一，为什么保存寄存器，内核保存寄存器是因为之后运行的C代码会覆盖这些寄存器，确保之后恢复用户程序正常，就需要能将寄存器恢复成ecall调用之前的数值，所以我们将所有寄存器的值保存到trapframe page中。二，**为什么保存到trapframe page，而不是直接存到user stack中，因为我们不确定用户程序是否有stack，必然有一些编程语言没有stack，对于这些编程语言的程序，SP的值就是0；或者有些语言虽然有stack，但是格式奇怪以至于kernel不能理解，比如有些编程语言从heap中取small blocks分配给stack，编程语言运行时知道如何使用这些small blocks of memory作为stack，但是kernel不知道。所以，如果我们想运行人意编程语言实现的用户程序，内核就不能假设用户内存的哪一部分可以访问，内核需要自己管理这些寄存器的保存。这就是为啥kernel将寄存器保存到trapframe而不是user memory。**

问题：为什么在gdb中看不到ecall的具体内容？好像我们是直接跳到trampoline代码的。

回答：我们确实跳过了ecall。ecall会更新CPU中mode标志位为supervisor，设置PC寄存器为STVEC寄存器的值，而内核事先将trampoline page的地址存到了STVEC寄存器。所以ecall下一条指令的位置是STVEC指向的地址，也就是trampoline page的起始地址。

问题：为什么trampoline代码中不保存SEPC寄存器？

回答：可以存储，trampoline代码没有像其他寄存器一样保存SEPC寄存器。但是我们欢迎大家修改XV6来保存它。这个SEPC寄存器实际上在C代码中usertrap中保存，而不是在汇编代码trampoline中保存。我想不出这里哪种方式更好。**用户寄存器必须在汇编代码中保存，因为任何需要经过编译的语言（比如C语言），都不能修改renew用户寄存器。所以对于用户寄存器，必须在进入C代码之前在汇编代码中保存好**。但是对于SEPC寄存器，我们早点或晚点保存都行。

问题：在内核要切换回用户空间时，执行到trampoline汇编代码函数userret中时，这里trapframe中的a0寄存器存储的是当前系统调用的返回值吗？（前面说过所有函数返回值会放到a0寄存器）

回答：是的，系统调用的返回值覆盖了a0寄存器，我们希望切换回用户进程Shell时，看到a0寄存器的值是系统调用的返回值。

问题：sret执行时，如果其他程序触发中断，会怎么样？

回答：sret是在supervisor mode中的最后一条指令，其会重新打开中断（允许响应其他中断请求）。同时也会设置PC为SEPC的值，且切换到user mode。

问题：我看到有一个uie位，我想它在sstaus寄存器中，但是我们不使用它，我们只用spie，并且在user space将它设置为false。我想知道我们为什么不用uie。

回答：我对uie不太清楚，我猜测sret最后会把spie复制到任何控制中断和用户模式的地方，可能是sstatus中的bit位uie。这仅仅是我的猜想。

## 6.5 系统调用小结

​	系统调用被刻意设计得像是普通的函数调用，虽然系统调用更加复杂，因为其需要保持user/kernel之间的隔离性，内核不能信任任何来自user space的东西。同时，系统调用希望有更简单快速的硬件机制(hardware mechanisms)。实际上XV6设计上不太注重性能。但现实中的操作系统设计者、CPU设计者会非常关心如何提高trap的效率。显然其他操作系统还有很多有别于XV6的trap实现方式。

# Lecture7 Q&A for Labs

> [Lecture 7 - Q&A for Labs 中文版Beta_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1rS4y1n7y1/?p=6&spm_id_from=pageDriver&vd_source=ba4d176271299cb334816d3c4cbc885f)

+ 每个进程都有自己的Kernel页表

  + 方法一，copy：每次都复制一个新的kernel page
  + 方法二，share：共享kernel page

  两种方式各有利弊，该课讲师更偏向方法二，但没有说方法二就一定更好。

+ 我们实际在Kernel页表的PLIC条目下的地址存储user page table。用户程序实际在Kernel页表的底部。

+ exec基本上做的就是，构造一个新的用户地址空间，然后将新用户空间复制到kernel页表的一行记录中。

+ 一个stack只占用一页（4096byte），如果超过了会触及guard page（无映射到物理地址），会触发page fault。

---

**提问：有个page的flag标志位是f，即V、R、W、X都是1，而U正好是0，有人知道这个page是什么吗？**

**回答：guard page，守护页。图中stack从高往低地址扩展，stack和data之间夹杂guard page，如果栈中条目超过4096条，触及guard page时，由于guard page没有设置U（User）位标识，此时会触发page fault或trap into the kernel。因为MMU不能转换guard page上的地址到物理地址（因为没有设置user位，所以被禁止转换）。**

提问：上图中最后一条标志位是b的page是什么？（b=11=8+2+1，X、R、V）

回答：trampoline页，它设置了X位，可执行，R可读，这就是我们用来恢复和保存寄存器的那个页。这里需要注意的是没有设置U，意味着用户程序不能执行指令。往前一页标识位WRV，同样没有设置U，只有内核能够读写。

问题：我记得书上说trampoline和trapframe位于地址空间的顶部，但上图中，却处于root page table的255而不是511条目，这是为啥？

回答：我们总说trampoline位于地址空间的顶部，即指向top level directory的第511条目，而现在却是255，有人知道为什么吗？

再回答：我记得说过有一个bit我们要用但实际没有用，因为符号扩展问题，且为了更简单，我们不需要那么多内存。

回答：这是一个技术细节，虚拟地址原则上是39bit，但是实际上在XV6中，只有38bit。因此顶端的MAXVA对于我们来说只有255条目。我们不使用39bit的原因是没有合适的理由需要这么做。如果设置了第39bit，则64bit的其他剩余位都必须是1，我们不想处理这个问题。如果我们设置了第39位，那么就还需要设置40、41...一直到64位。

问题：为什么文本text和数据data在同一页上？

回答：为什么我们不把它们放到单独page上呢？这样不是能更仔细地设置权限了。我们这样做的主要原因是为了简单。通常`exec`会很复杂，但是我们想让`exec`尽量简单点。**在一个真正的操作系统上，是不会同一页中同时包含text和data的**。实际上，你看makefile中的loader flag，会看到一个`-N`选项，我们强制数据data和文本text在一个连续的page，而不是在各自的pages中。

问题：你说我们只用了38bit，是硬件提供给我们39bit，但我们自己设计操作系统时只用38bit吗？

回答：是的。所以如果机器有超过2^38 bytes的内存，那么我们就用不着那些内存了。所以我们现在假设我们运行的内存远小于2^38bytes。但如果是真正的操作系统，我们会做得更好，我们现在这么做只是为了更简单。

问题：你是不是把CLINT映射到新的进程Kernel page table了？

回答：是的。因为用户进程不会比CLINT更大，当我映射PLIC和CLINT时，我认为assignment告诉我们，PLIC是最低的地址，而用户进程不会大于PLIC地址。这么做仅仅是为了更简单。

问题：能不能复制0～512，然后每次进程切换时，仍然用全局的root page table，你只是复制了first root page table。那么每次切换进程，你都把用户地址进行复制。能不能这么做？

回答：理论上可以，而不是分配一个进程再释放它。你可以在调度程序切换期间动态执行，这看着会更复杂且花费更多的时间。这意味着每次切换进程，都必须去复制Kernel页表的一部分，性能上可能有影响。你这么做的话，可能在usertests（作业测试程序）中超时。（后面同学补充说他试过觉得这很糟糕，但只是确认下这种思路是否可行。老师认为理论上可行，可以每次分配新页表并切换，并在切换出来的时候再释放它。我不认为这简单，但理论上确实可行。）

提问：我们目前的设计中，用户程序的页表，最多增长到CLIENT地址。如果后续我们想一直增长到更高的地址，怎么做？

回答：我们可以将CLIENT等重新映射到更高的地址空间，腾出空间用于用户空间。真正的操作系统就是这么做的。因为内核本身用不着太多的地址空间，很多都是空闲的。

**问题：一旦我们有了进程的Kernel页表，这是否意味着在trap代码中，我们就不需要切换页表了？**

**回答：是的，这是个很好的设计问题。主要原因在于，Kernel或trampoline代码，麻烦就麻烦在我们需要复制用户页表，我们必须从Kernel页表切换到用户页表或其他没有kernel映射的页表。换种说法，你可以简化entry和exit，如果你在Kernel中有一个单独的page table映射user，这样你就不需要切换了。你需要对XV6进行更多的修改，才能做到这点。但原则上可以做到。<u>实际上，直到最近LInux还在使用这种策略，即Kernel and user code位于同一个single page table，依赖u flag标识位来确保用户程序无法修改任何Kernel pages。在这种情况下，entry、exit code会更加简单一些，因为在进入/离开Kernel时不必切换页表。但这也可能导致一些问题，比如meltdown attack熔断攻击，side channel attack侧信道攻击</u>。针对侧信道攻击，Linux会切换到另一种模式运行，它有两种模式，一种叫kpti mode，该模式下和XV6所做的事类似，有一个单独的Kernel page table和单独的user page table**。

**追问：假设每个用户进程和Kernel使用相同的page table，如果用户内存必须设置u bit，kernel将无法访问该用户内存，对吗？**

**回答：实际上，在Intel处理器上就没有这条rule。在Intel处理器上，如果设置了u flag，Kernel仍然可以写入和读取那个页**。

**追问：那就是说设置u flag后，kernel无法访问，这个rule只限于RISC-V上吗？**

**回答：实际即使在RISC-V上，你可以修改它，在sstatus寄存器有一个bit，你可以修改它。如果你修改了，那么在Kernel mode中，u flag位就会被忽略**。

# Lecture8 Page Faults

