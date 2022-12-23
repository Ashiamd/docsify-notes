# MIT6.S081网课学习笔记-01

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

## 8.1 page fault概述

> [谈谈缺页、swap、惰性分配和overcommit - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/471040485) <= 推荐阅读

​	围绕page fault的virtual memory functions

+ lazy allocation
+ copy-on-write fork
+ demand paging
+ memory mapped files（mmap）

​	几乎所有操作系统都实现了这些功能，比如Linux就实现了以上所有功能。但在XV6中，一个都没有实现。在XV6中，一旦用户空间进程触发了page fault，会导致用户进程被kill，这是非常保守的处理方式。

---

​	回顾一下虚拟内存的两个优点：

+ Isolation：使得操作系统能够为每个应用程序提供属于各自的地址空间（user-user之间，user-kernel之间，提供隔离性）。
+ level of indirection：处理器和所有指令都可以使用虚拟地址，而内核会定义从虚拟地址到物理地址的映射关系。

​	虽然XV6有很多虚拟地址是直接映射到物理地址，但也有一些有趣的地方，比如：

+ trampoline page：它使得内核可以将一个物理内存page映射到多个用户地址空间中。
+ guard page：它同时在内核空间和用户空间用来保护Stack。

---

​	之前我们提到的关于page table的场景，都比较静态，基本上最初设置好页表后，就不再有任何变动。而利用page fault机制，可以让内核动态更新page table。

## 8.2 触发page fault

​	当发生page fault时，内核需要什么信息才能响应page fault？

+ 触发page fault的虚拟地址

  当page fault出现时，XV6内核会打印具体的虚拟地址，而这个地址会被保存到**STVAL寄存器**中。当一个用户程序触发了page fault，trap机制会切换到内核模式，同时将虚拟地址存放到STVAL寄存器中。

+ 触发page fault的原因类型（触发原因）

  我们可能需要对不同的page fault作出不同处理，比如load指令触发的page fault，又或者是store、jump等其他指令触发的page fault。实际上如果看RISC-V的文档，有介绍引起page fault的源头信息会存放在**SCAUSE寄存器**中。有3类原因与page fault有关，分别是read、write、instruction。

+ 触发page fault的指令的虚拟地址

  在XV6中，作为trap代码的一部分，该值存放在**SEPC寄存器**（Supervisor Exception Program Counter）中。

## 8.3 lazy allocation理论

​	sbrk是XV6提供的系统调用，使应用程序能调整自己的heap大小。

​	**当一个应用程序启动时，sbrk指向heap的底部，stack的顶部**。它通过进程数据结构的sz字段表示，即`p->sz`。

​	当调用sbrk时，它的参数是整数，代表想要申请的page数量。

+ 传入正数，内核会分配一些物理内存，并将这些内存映射到用户应用程序的地址空间，然后将内存内容初始化为0，再返回sbrk系统调用。这样，应用程序可以通过多次sbrk系统调用来增加它所需要的内存。
+ 传入负数，减少或者压缩它的地址空间。

​	在XV6中，sbrk的默认实现是eager allocation。即一旦调用了sbrk，内核会立即分配应用程序所需要的物理内存。但是实际上，应用程序本身很难预测自己需要多少内存，所以通常应用进程倾向申请多于实际需求的内存，意味着进程会浪费更多内存（有些多申请的内存永远用不上）。

---

​	lazy allocation的核心思想非常简单。sbrk系统调用只改变地址空间的指向，即修改`p->sz + n`，n表示需要新分配或缩减的内存page的数量，<u>但此时内核并不真正调整物理内存的分配</u>。**当应用程序真正使用到这片内存时，会触发page fault，因为我们没有将新内存映射到page table**。此时触发page fault时，发现当前虚拟地址小于当前`p->sz`，并且大于stack地址，那么虚拟地址一定在stack之上，是来自heap的地址，但是内核还没有分配对应的物理内存。在page fault handler中，通过kalloc函数分配一个内存page，初始化page内容为0，并将该page映射到user page table，最后重新执行（原本在执行的）指令，由于此时分配了物理内存，重新执行指令一般就能正常运行了。

---

问题：在eager allocation场景下，应用程序可能耗尽物理内存。而在lazy allocation下，应用程序怎么知道当前已经没有物理内存可用了？

回答：从应用角度会有一个错觉——存在无限多可用的物理内存。在某个时间点，要是应用程序真的申请耗尽了物理内存，此时如果再访问一个未被分配的page，但又没有更多物理内存，此时内核有两个选择：返回错误并kill当前进程，因为Out of Memory了；另一个更聪明的方式后续介绍。

问题：为什么虚拟地址必须从0开始？

回答：在地址空间中，我们有stack、data、text。通常我们将`p->sz`设置成一个更大的值（越过stack很大一块地址）。此时如果使用的地址低于`p->sz`，认为这是一个用户空间中有效的地址；而使用的地址大于`p->sz`，则认为是个程序错误，程序尝试解析一个不属于自己的内存地址。

问题：为什么内存耗尽之后，要kill当前进程。就不能是返回个类似OOM的错误，尝试做一些别的操作么？

回答：在XV6的page fault中，默认操作就是kill当前进程。实际的操作系统会处理得更妥善，但如果最终还是找不到可用内存，还是可能会kill当前进程，基本没别的选择。

## 8.4 lazy allocation代码

> [8.2 Lazy page allocation - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/336624464) <= 图片出处
>
> [6.S081/Fall 2020 lab5 lazy allocation - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/372148138) <= 实验

​	我们首先要修改的是`sys_sbrk`函数，`sys_sbrk`会完成实际增加应用程序的地址空间，分配内存等等一系列相关的操作。

```c
// 1. 修改前
uint64
  sys_sbrk(void)
{
  int addr;
  int n;
  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  if(growproc(n)<0)
    return -1;
  return addr;
}
```

​	这里我们要修改这个函数，让它只对`p->sz`加n，并不执行增加内存的操作。

```c
// 2. 修改后
uint64
  sys_sbrk(void)
{
  int addr;
  int n;
  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz = myproc()->sz+n;
  //if(growproc(n)<0)
  //  return -1;
  return addr;
}
```

​	修改完后，执行`echo hi`得到一个page fault。

![img](https://pic1.zhimg.com/80/v2-7c01ee32d7d6333ebcaa9339d4f31e28_1440w.webp)

​	因为Shell中执行echo会先fork一个子进程，子进程中exec执行echo。这过程需要申请内存，于是Shell调用了`sys_sbrk`，然后就出现page fault。因为刚才修改了`sys_sbrk`代码，不会实际分配物理内存。

​	上图可以看出：

+ SCAUSE寄存器的值15，触发page fault的原因类型，15表示store page fault。

+ SEPC寄存器的值`0x12a4`，触发page fault的指令的虚拟地址，查看Shell汇编代码，得知`0x12a4`对应store指令。
+ STVAL寄存器的值`0x4008`，触发page fault的当前虚拟地址，查看Shell汇编代码，发现当前操作的地址为a0寄存器值偏移8个字节，即`0x4000`+8。

![img](https://pic4.zhimg.com/80/v2-2724ac4410f4532acffd0f8197950627_1440w.webp)

---

​	以上就是page fault的信息。我们接下来看看如何能够聪明的处理这里的page fault。

​	首先查看`trap.c`中的usertrap函数。在usertrap中根据不同的SCAUSE完成不同的操作。

![img](https://pic4.zhimg.com/80/v2-00d9f1e72d46e20b8d72f543a29af4cf_1440w.webp)

+ `SCAUSE==8`，处理普通系统调用，然后其中有一行检查是否有其他设备中断和处于设备中断内的进程，如果两个条件都不满足，则打印一些信息并kill进程。
+ 我们在这里加上`SCAUSE==15`的相关处理逻辑代码，在增加的代码中，首先打印一些调试信息，之后分配一个物理内存page。
  + 没有足够物理内存（OOM）时，我们直接kill进程。
  + 如果有物理内存，首先会将内存内容设置为0，之后将物理内存page指向用户地址空间中合适的虚拟内存地址。具体来说，我们首先**将虚拟地址向下取整**。比如这里引起page fault的虚拟地址是`0x4008`（进入第五页的第8字节），我们想将这个物理page映射到虚拟page底部，即`0x4000`，PTE需要设置常用的权限标志位，u，w，r bit位。

![img](https://pic2.zhimg.com/80/v2-2b8a27560554fc281bbf5d5115877849_1440w.webp)

​	此时修改完代码，重新运行`echo hi`，抛出两个page fault。第一个是前面见到的`0x4008`，第二个`0x13f48`，在`uvmunmap`报错，它尝试`unmap`的page并不存在。有人知道这里`unmap`的内存是什么吗？

![img](https://pic4.zhimg.com/80/v2-02542f1797f9cbb04cb65bfa9f7ccc7b_1440w.webp)

​	这里unmap的内存，即lazy allocated，但是还没实际使用的内存，这里还没有对应的物理内存。当PTE的v flag为0，并且没有对应的mapping，panic打印信息，符合预期。

![img](https://pic2.zhimg.com/80/v2-e0a27fa36739b8020b849ca8d2f52241_1440w.webp)

​	这里修改`uvmunmap`的代码，去掉panic，直接continue，因为前面修改`usertrap`代码，已经为当前情况分配物理内存了。

![img](https://pic2.zhimg.com/80/v2-bf899617f9017f0a07668720503cad25_1440w.webp)

​	重新执行`echo hi`，虽然看到page fault，但正常打印"hi"。

​	![img](https://pic3.zhimg.com/80/v2-abff7a44dad73a739a2ce9a40588e51a_1440w.webp)

​	这里没有严谨的检查触发page fault的虚拟地址是否小于`p->sz`，后续做实验需要自己注意边界问题。

----

问题：为什么这里`uvmunmap`可以直接删掉`panic`，改成`continue`？

回答：这个bug表明我们试图释放没有映射的page，发生这种情况的唯一原因是sbrk移动了`p->sz`，但是应用程序还没有使用那部分内存。这时由于lazy allocation，那对应的物理内存还没分配，新增的内存没有映射关系。因为没有分配内存，没有mapping，那么对应内存当然也不能释放。由于没有对应的虚拟地址，毕竟实际没有分配物理内存，这时我们对这个page不用做释放，直接跳过，处理下一个page即可。

追问：在`uvmunmap`中，我认为之前的panic存在是有原因的。我们是不是该判断下，对特定场景仍然panic？

回答：为什么之前这里有panic，过去未经修改的XV6，永远不会出现用户内存未map的情况，所以一旦出现这种意外情况就需要panic（前面说了XV6默认的sbrk实现，会直接分配物理内存和设置映射关系）。但这里我们修改了XV6，lazy allocation就会存在这种情况，所以去掉panic。

问题：前面说过sbrk的参数是整数，可以传负数事吗？

回答：是的，负数意味着缩小用户的内存。

## 8.5 基于page fault和page table的zero-fill-on-demand

> [8.3 Zero Fill On Demand - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/336625055) <= 图片出处

​	基于page fault和page table，一个简单但频繁使用的功能，zero-fill-on-demand。

​	在操作系统中，有很多空页。在一个正常的操作系统执行exec，会看到申请的**用户程序地址空间包含Text（用户程序指令）、Data（初始化后的全局变量）、BSS（未被初始化或初始化为0的全局变量）三部分**。

​	**BSS这里有很多内容全是0的page，对应的调优操作，物理内存只需要分配一个page，这个page内容全是0，然后将所有虚拟地址空间全0的page都map到这个物理page上**。

​	需要注意的是，这里mapping后，不能允许对这个page进行写操作，因为所有虚拟地址空间page都期望page的内容是全0。**这里PTE都得是read only只读的**。

​	**当用户进程需要修改BSS中某一个page内的一两个变量的值，会得到page fault。此时，如何处理该page fault？假设store指令发生在BSS顶端的page中，我们想做的是在物理内存中申请一个新的page，并将其内容设置为0，之后更新这个page 的mapping关系。首先将PTE设置成read write，然后将其指向新物理page。这里相当于copy and update PTE，然后继续执行指令**。

​	为什么要这么做？这里类似lazy allocation，假设程序申请了初始为0的全局变量大数组，但或许最后只使用其中一部分。第二个好处，在exec中需要做的工作更少了，程序可以启动更快，因为你不需要分配内存，不需要将内存设置值为0。你只需要分配一个内容全是0的物理page。你只需要写这个PTE。

![img](https://pic1.zhimg.com/80/v2-2e021fe6c4a82fc54c4a0f5580ba944c_1440w.webp)

---

问题：这里BSS类似lazy allocation的处理方式，是不是会导致update或write更慢？因为每次都会触发一个page fault。

回答：这是一个好观点。我们将一些操作推迟到了page fault再去执行，并且期望不是所有page都被使用。如果一个page是4096字节，我们只需要对每4096字节消耗一次page fault即可。我们的确增加了一些由page fault带来的代价。但是我们要想想如何衡量这个page fault的代码，比如会和store指令相当吗？或甚至更高？

追问：page fault代价是不是更高？我们store指令可能会消耗一些时间访问RAM，但是page fault需要进入kernel 。

回答：是的，仅在trap处理代码中，就有至少100个store指令用来存储当前的寄存器。除此之外，还有从用户空间到内核空间的额外开销，以及为保存和恢复状态而执行的所有指令的开销等等。所以page fault并不是没有代价的。

## 8.6 copy-on-write fork(COW fork)

> [8.4 Copy On Write Fork - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/336625238) <= 图片出处
>
> [再谈 copy-on-write - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/136428913)
>
> [6.S081/Fall 2020 lab6 Copy-on-Write Fork for xv6 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/372428507) <= 实验

​	许多操作系统都实现了copy-on-write fork(COW fork)。

​	原始fork操作直接申请新page，复制父进程的page内容，但是后续执行exec由释放这些page，重新申请page执行其他代码。这过程显得第一次申请的page毫无意义，反而额外性能浪费。

​	这里有一个优化方法。当创建子进程时，不直接分配新的物理内存page，而是让子进程PTE直接指向父进程原本的物理内存。这里mapping要很小心，为了确保父子进程之间有强隔离性，可以将父进程和子进程的PTE都设置为read only。某个时间点需要修改对应内存内容时，由于尝试向一个只读的PTE写数据，即得到page fault。此时，我们需要拷贝对应的物理page，将原本的物理内存page内容拷贝到新分配的物理page中，将新分配的物理page映射到子进程，新PTE修改成RW，原本父进程的PTE也修改成RW。然后重新指令用户指令，即调用userret函数（前面提过的返回用户空间的方法）。（ps：<u>这里只需要拷贝这个被修改的PTE对应的page，其他PTE暂时不需要拷贝。</u>）

​	**目前在XV6中，除了trampoline page外，一个物理内存page只属于一个用户进程。trampoline page永远也不会被释放**。

​	**COW fork，导致多个虚拟地址PTE都指向了相同的物理内存page，父进程退出时我们要更加小心，因为需要判断是否能立即释放父进程对应的物理page。如果有其他子进程由于COW fork而仍在使用和父进程相同的物理page，这时候如果内核释放这些物理page就会有问题。此时，释放物理page的依据是什么？我们需要对每一个物理page的引用进行计数，当我们释放虚拟page时，对应物理内存page的引用计数间减1，如果引用计数等于0，则释放物理内存page。**

![img](https://pic1.zhimg.com/80/v2-5af6213a1086441ab7fcbfb8e136a10c_1440w.webp)

---

问题：我们如何发现父进程写了这部分内存地址？是与子进程一样的方法吗？

回答：是的，因为子进程的地址空间来自于父进程的地址空间的拷贝。如果我们使用了特定的虚拟地址，因为地址空间是相同的，不论是父进程还是子进程，都会有相同的处理方式。

问题：对于一些没有父进程的进程，比如系统启动的第一个进程，它会对自己的PTE设置为只读吗？还是设置成可读写的，然后在fork的时候在修改成只读的？

回答：这取决于你的实现，在copy-on-write lab中，你可以选择自己的实现方式。当然最简单的方式就是将PTE设置成只读，当你要写这些page时，得到page fault，之后再按照上面的流程进行处理。对于你说的这种情况，也可以这么做，没有理由做特殊处理。

问题：我们经常会拷贝用户进程对应的page，内存硬件有没有特定的指令来完成拷贝？从page a拷贝到page b的需求，有相应的指令吗？

回答：x86有硬件指令可以用来拷贝一段内存，但RISC-V没有这样的指令。当然在一个高性能的实现中，所有这些读写操作都会流水线化，并且按照内存的带宽速度来运行。实际上，我们可能是幸运的，我们节省了负载和stores或copies。在我们这个例子中，我们只需要拷贝1个page。对于一个未修改的XV6系统，我们需要拷贝4个page（说XV6的shell通常申请4个page，所以fork子进程也是4page）。所以这里的方法明显更好，因为内存消耗更少，并且性能会更高，fork会执行更快。

**问题：当发生page fault时，我们其实是在向一个只读的地址进行写操作，内核是如何分辨现在是一个copy-on-write fork的场景，而不是应用程序在向一个正常的只读地址写数据？除了COW fork，由于某些合理的原因page只能读取。是不是说默认情况下， 用户程序的PTE都是read write的，除非COW fork场景下，才会出现只读的PTE？**

**回答：是的，它需要在内核中维护一个invariant，内核必须能够识别这是一个copy-on-write的场景。RISC-V硬件几乎所有的page table硬件都支持了这一点。PTE中最后2bit RSW，保留给supervisor software使用，即内核可以随意使用这2bit。可以做的其中一件事就是将第8 bit标识当作当前是一个copy-on-write page。当内核管理这些page table时，如果是copy-on-write相关的page，就可以设置对应的bit标识位。发生page fault时，就可以根据copy-on-write bit 位判断场景，然后执行相关操作。**

![img](https://pic3.zhimg.com/80/v2-b4e1e2065f5300421905e43f87ea45ca_1440w.webp)

问题：COW fork中，我们需要在哪存储物理page的引用计数？如果我们需要对每个物理page引用计数的话。

回答：对每一页内存都需要做引用计数，即每4096字节，都需要维护一个引用计数。

追问：我们可以将引用计数存在RSW对应的2bit中，并限制不超过4个引用吗？

回答：可以，但是如果引用超过4次，就会是个问题。因为超过4次，你就不能再使用这个优化了。你可以自由地选择实现方式。

问题：真的有必要额外增加1bit标识当前page是copy-on-write的吗？内核可以维护一些有关进程的信息，是不是可以包含这个信息。

回答：你可以在管理用户地址空间时，维护一些其他的元数据信息。这样你就知道这部分虚拟内存地址如果发生了page fault，那必然是copy-on-write的场景。在后续实验中，你们也会接触到扩展XV6管理的元数据。

## 8.7 demand paging

​	回到前面提到的exec，在未修改的XV6中，操作系统会加载程序的text segment、data segment，并且以eager的方式加载到page table中。但根据lazy allocation和zero-fill-on-demand，为什么我们要以eager的方式将程序加载到内存？为什么不等程序实际用到这些指令的时候再加载到内存？毕竟程序的二进制文件也可能非常大。

​	对于exec，我们为text和data分配好地址段，但在ptes中，我们不马上映射它们。我们只会保留其中一页PTE，对于这些PTE，我们只需要将valid bit设置为0就行。位于地址0的指令会触发我们的第一个page fault，因为我们还没有真正加载内存。如何处理这里的page fault？首先我们可以发现这些page是on-demand page，我们需要在某个地方记录这些page对应的程序文件，我们在page fault handler中需要从程序文件中读取page数据加载到内存，之后将内存page映射到page table，最后再重新执行指令。在最坏的情况下，用户程序使用了text和data的所有内容，那么我们会在应用程序的每个page都收到一个page fault。但幸运的情况下，用户程序没有使用所有的text segment或者data segment，那么我们就节省了一些物理内存，同时exec运行也更快了。

​	对应demand paging，如果kalloc返回0，即内存用完了，这时候得到一个page fault。如果是需要从文件加载到内存，但此时没有可用的物理page，该怎么做？即lazy allocaiton中，如果内存耗尽了如何处理？

​	如果内存耗尽了，一个选择是evict a page（撤回一个page），比如将部分page中的内容写回file，然后撤回page。这时候你就可以用刚空间出来的page，然后restarting instruction again，这个操作比较复杂，包含了整个userret函数背后的机制，以及切换回user space等等。这就是常见的操作系统行为。

​	**什么样的page可以evict撤回？并且用什么策略evict？Leat Recently Used（LRU），这是最常用的策略。除了这个策略外，还有一些优化点。比如要撤回一个page，你要在dirty page和non-dirty page中做选择，前者被写过，后者只是被读过但没被写过。选择dirty page，如果后续又写该page，那就相当于对它写了两次。现实中会选择non-dirty page，因为这样就可以不用额外做其他事。你可以重复使用它，标记它是否存在于可分页的文件中，之后将相应的PTE标记成non-valid，这就完成了所有的工作。**之后你可以在另一个page table中重复使用这个page。所以通常会优先选择non-dirty page来撤回。

​	如果你们看PTE，会发现第7bit（D），就是Dirty bit。当硬件向一个page写入数据，就会设置dirty bit。之后操作系统就可以发现这个page被写过。类似的，第6bit（A），Access bit，任何一个page被读或写，都会被设置。

---

问题：对于一个cache，我们可以认为它被修改后但还没被写回memory时是dirty的。但对于内存page，怎么判断dirty？它只存在内存，不存在于其他地方，那它什么时候变成dirty？

回答：例如，如果demand page file page，后面会详细说。如果是memory map files，将文件映射到内存中，然后进行store，就会变成dirty page。

追问：所以dirty，只对memory map files的场景下的page才有效？

回答：是的。（这里错了，只要page被执行过store操作，就会标记为dirty）

**问题：PTE的Access bit，可以帮助我们evict page，连access都没有过，可以直接evict，对吗？**

**回答：是的。或如果你想实现LRU，你需要找到一个在一定时间内没有被访问过的page。那么这个page可以被用来撤回，而被访问过的page则不撤回。Access bit通常用来实现LRU策略**。

**追问：那是不是要定时将Access bit恢复成0？**

**回答：是的，这是一个典型的操作系统行为。如果它不这么做，操作系统会扫描整个内存。这里有一些著名的算法， 比如clock algorithm就是其中一种实现方式**。

**追问：为什么需要恢复Access bit？**

**回答：如果你想知道page最近是否被使用过，你就需要定时这么做，比如每100ms或1s清除Access bit。或者在下一个100ms这个page曾经被访问过，Access bit为0的page则表示在上一个100ms未被使用。进而能够统计每个内存page的使用频度，这是一个成熟的LRU实现的基础。**

## 8.8 memory-mapped files

> [mmap 浅析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/353553313)

 	整体思想即，将完整或者部分文件加载到内存中，这样就可以痛殴内存地址相关的load或者store指令操作文件，而不是使用文件IO相关的read/write指令。

​	为了支持这个功能，一个现代的操作系统会提供一个叫做mmap的系统调用。这个系统调用会接收一个虚拟地址，或一些 virtual address length，protection，flags，file descriptor，offset。 也许你应该把这个文件描述符映射到一个虚拟地址，从文件描述符对应的文件偏移量的位置开始，映射长度为len的内容到虚拟地址VA。同时加上一些保护，比如只读、只写等等。

​	假设文件设置read write，并且内核实现mmap的方式是eager，就像大多数系统会急切地执行copy，内核会从文件offset的位置开始，将数据拷贝到内存，设置好PTE指向物理内存的位置。之后应用程序就可以使用load或者store指令，来修改内存中对应文件的内容。当完成操作之后，会有一个对应unmap的系统调用，参数是VA，len，用来表明应用程序完成了对文件的操作，在unmap时间点，需要将dirty block写回文件中。通过dirty bit为1的PTE可以快速找到ditry block。当然在任何完善的操作系统中，这些都是以lazy的形式实现的，不会立即将文件拷贝到内存，而是先记录PTE属于这个文件描述符，相应的信息通常在VMA（Virtual Memory Area）结构体中存储。例如对于这里的文件f，会有一个VMA，在VMA中我们记录文件描述符、offset偏移量等，用来表示对应内存虚拟地址的实际内容是什么。这样当我们得到一个在VMA地址范围的page fault时，内核可以从磁盘中读数据，并加载到内存中。dirty bit很重要，因为在unmap中，需要向文件回写dirty block。

---

问题：有没有可能多个进程将同一个文件映射到内存中，这些内存映射在二级存储上相同的文件，然后存在同步问题？

回答：这问题等价于，多个进程同时通过read/write系统调用读写同一个文件会怎么样？你需要什么保证吗？这里的行为是不可预知的，write系统调用会以某种顺序出现，如果两个进程向同一个file的同一个block写数据，两个进程只有其中一个的write能生效。这里也是同理，我们不需要考虑冲突的问题。一个更加成熟的Unix操作系统支持file locking，你可以先lock file，保证数据同步。但是默认情况下直接执行write，没有任何同步保证。

问题：mmap的参数中，len和flag是什么意思？

回答：len是文件中需映射到内存的字节数，prot是 read write x flag，map上可以看到，表示这个区域是private的还是shared的，shared则多个进程之间共享。

问题：如果其他进程直接修改了文件内容，那是不是意味着修改的内容不会体现在这里的内存中？

回答：是的，但如果文件mapped shared，那么你应该同步这些变更。

追问：它们应该使用相同的文件描述符吧？

回答：我记不大清楚在mmap中，文件共享会发生什么。

## 8.9 小结

+ page table如何工作
+ trap机制
+ page fault以及基于其的一些功能/机制

你能在Linux中发现包含以上的内容且有更多有趣的地方。

# Lecture9 中断Interrupts

> [top命令的常用方式_勿视人非的博客-CSDN博客_top指令](https://blog.csdn.net/u011939453/article/details/124515892) <= 课程开头简单介绍了下top指令
>
> [理解virt、res、shr之间的关系（linux系统篇） (baidu.com)](https://baijiahao.baidu.com/s?id=1743908545937632735&wfr=spider&for=pc)
>
> [中断机制_百度百科 (baidu.com)](https://baike.baidu.com/item/中断机制/2928852?fr=aladdin)
>
> 中断机制是现代计算机系统中的基本机制之一，它在系统中起着通信网络的作用，以协调系统对各种外部事件的响应和处理，中断是实现[多道程序设计](https://baike.baidu.com/item/多道程序设计/10804195?fromModule=lemma_inlink)的必要条件，中断是CPU 对系统发生的某个事件作出的一种反应。引起中断的事件称为中断源。中断源向CPU 提出处理的请求称为中断请求。发生中断时被打断程序的暂停点称为断点。CPU暂停现行程序而转为响应中断请求的过程称为中断响应。处理中断源的程序称为中断处理程序。CPU执行有关的中断处理程序称为中断处理。而返回断点的过程称为中断返回。中断的实现由软件和硬件综合完成，硬件部分叫做硬件装置，软件部分称为软件处理程序。
>
> [Linux中的硬中断与软中断 - zed99 - 博客园 (cnblogs.com)](https://www.cnblogs.com/zed99/p/16645015.html) <= 对软硬中断介绍蛮清楚的
>
> 1. 问：对于软中断，I/O操作是否是由内核中的I/O设备驱动程序完成？
>
>    答：对于I/O请求，内核会将这项工作分派给合适的内核驱动程序，这个程序会对I/O进行队列化，以可以稍后处理（通常是磁盘I/O），或如果可能可以立即执行它。通常，当对硬中断进行回应的时候，这个队列会被驱动所处理。当一个I/O请求完成的时候，下一个在队列中的I/O请求就会发送到这个设备上。
>
> 2. 问：软中断所经过的操作流程是比硬中断的少吗？换句话说，对于软中断就是：进程 ->内核中的设备驱动程序；对于硬中断：硬件->CPU->内核中的设备驱动程序？
>
>    答：**是的，软中断比硬中断少了一个硬件发送信号的步骤。产生软中断的进程一定是当前正在运行的进程，因此它们不会中断CPU。但是它们会中断调用代码的流程**。
>
>    如果硬件需要CPU去做一些事情，那么这个硬件会使CPU中断当前正在运行的代码。而后CPU会将当前正在运行进程的当前状态放到堆栈（stack）中，以至于之后可以返回继续运行。这种中断可以停止一个正在运行的进程；可以停止正处理另一个中断的内核代码；或者可以停止空闲进程。
>
> [你真的理解Linux中断机制嘛 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/440458446) <= 推荐阅读，包括linux指令操作
>
> Linux 将中断处理过程分成了两个阶段，也就是上半部和下半部。
> 上半部用来快速处理中断，它在中断禁止模式下运行，主要处理跟硬件紧密相关的或时间敏感的工作。也就是我们常说的硬中断，特点是快速执行。
> 下半部用来延迟处理上半部未完成的工作，通常以内核线程的方式运行。也就是我们常说的软中断，特点是延迟执行。
>
> [Linux中断_易风尘的博客-CSDN博客_linux中断](https://blog.csdn.net/g498912529/article/details/125828769)
>
> **RTOS:** RTOS一般以线程为调度任务，没有进程概念，但都会提供一套类似标准操作系统的 线程同步机制。此时，在设计中断程序时，可以更好实现“上半部”和“下半部”，特别是“下半部”。以RT-Thread为例，实现一个串口中断接收函数。上半部分负责将数据放入缓存，并通过信号量通知下半部分处理。下半部实现，可以创建一个处理线程，当获得上半部信号量时则调度线程进行处理数据，否则则挂起该线程，节约cpu资源。
>
> **Linux：** 操作系统是多个进程和多个线程执行，宏观上达到并行运行的状态，外设中断则会打断内核中任务调度和运行，及屏蔽其外设的中断响应，如果中断函数耗时过长则使得系统实时性和并发性降低。中断原则是尽可能处理少的事务，而一些设备中往往需要处理大量的耗时事务。为了提高系统的实时性和并发性，Linux内核将中断处理程序分为上半部（top half）和下半部（bottom half）。上半部分任务比较少，处理一些寄存器操作、时间敏感任务，以及“登记中断”通知内核及时处理下半部的任务。下半部分，则负责处理中断任务中的大部分工作，如一个总线通信系统中数据处理部分。
>
> ----
>
> 下面几篇主要讲网卡相关的（包括中断流程）
>
> [Linux中断处理 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA3NzYzODg1OA==&mid=2648464225&idx=1&sn=7cc26da1f9312e5abc00751c7d65805b&scene=21#wechat_redirect)
>
> [深度剖析Linux网卡数据收发过程分析（超详细） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/478380134)
>
> [深度剖析Linux 网络中断下半部处理（看完秒懂） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/478883502)
>
> ---
>
> 下面两个文章都是超级详细版本，两个文章可以说相辅相成了。
>
> [Linux 中断 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/94788008) <= 循序渐进，特别详细，建议阅读
>
> 软中断在什么时候执行呢？
>
> 1. 当前面的硬件中断处理程序执行结束后，会检查当前 CPU 是否有待决的软中断，有的话则会按照次序处理所有的待决软中断，每处理一个软中断之前，就会将其对应的比特位置零，处理完所有软中断的过程，我们称之为一轮循环
>
> - 一轮循环处理结束后，内核还会再检查是否有新的软中断到来（通过位图），如果有的话，一并处理了，这就会出现第二轮循环，第三轮循环
> - 但是软中断不会无休止的重复下去，当处理的轮数超过 MAX_SOFTIRQ_RESTART（通常是 10） 时，就会唤醒软中断守护线程（每个 CPU 都有一个），然后退出
>
> 2. 软中断守护线程负责在软中断过多时，以一个调度实体的形式（和其他进程一样可以被调度），帮着处理软中断请求，在这个守护线程中会重复的检测是否有待决的软中断请求
>
> - 如果没有软中断请求了，则会进入睡眠状态，等待下次被唤醒
> - 如果有请求，则会调用对应的软中断处理程序
>
> [中断 - Linux源代码导读报告 (ustc.edu.cn)](http://home.ustc.edu.cn/~boj/courses/linux_kernel/2_int.html) <= 超详细，包括硬件讲解

## 9.1 硬件中断-概述

> [UART_通信百科 (c114.com.cn)](https://baike.c114.com.cn/view.asp?id=8562-B10829A3)

​	硬件中断，硬件想要马上得到操作系统的关注。硬件中断很常见，比如：

+ 网卡收到一个packets后产生一个interrupt中断

+ 用户按下键盘的按键，键盘产生一个interrupt中断

​	应对中断，操作系统如下运作：

1. 保存当前的工作

2. 处理interrupt中断

3. 恢复之前保存的工作

​	**这里的1和3步骤，可以说system calls系统调用、page fault、interrupt中断，都是用相同机制**。

但是，中断(interrupt)和系统调用(system calls)有3个小不同点：

1. **异步asynchronous**

   **中断处理器处理硬件中断时，异步执行，与当前在CPU运行的进程毫无关联；而处理系统调用，需要进入内核进行处理，在调用进程的背景下运行（running in the context of the calling procss）**。

2. 并发性concurency

   产生中断的设备和CPU是并行的，例如网卡独立处理来自网络的packets，然后某个时间点产生中断，此时CPU也是无感知地并行运行中。

3. 编程设备(驱动程序)program devices

   外部设备，网卡(network cards)、UART等，这些设备需要被编程，类似RISC-V有指令和寄存器的手册，每个设备都有一个编程手册，比如有什么寄存器、能执行什么操作、在读写寄存器时设备如何响应等。

---

​	后面会围绕两个例子讲解背后的机制：

+ 控制台的提示符"$"如何被展示出来
+ 指令`ls`如何将内核输出到console中

## 9.2 CPU处理硬件中断-概述

> [9.2 Interrupt硬件部分 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/339796375) <= 图片出处

​	这里我们只讨论外部设备的中断（external interrupt），不考虑定时器中断（time interrupt）或软件中断（software interrupt）。外设中断来自主板上的设备。

​	后面主要讲解，当设备产生中断时，CPU会做什么，以及如何从设备读写数据。

![img](https://pic1.zhimg.com/80/v2-a795a05234581e4c3b3b61cd24944050_1440w.webp)

​	**所有设备都可以连接到处理器上，处理器通过平台级中断处理器(platform level interrupt controller，PLIC)来处理设备中断**。PLIC会管理来自外部设备的中断。

​	![img](https://pic1.zhimg.com/80/v2-132ffc10da778559f69dcc919c6cdad8_1440w.webp)

​	**下面是教学芯片设备的PLIC示意图，有53条来自不同设备的中断线(interrupt lines)，每个设备可能都拥有自己的中断线，他们到达PLIC后，PLIC会路由这些中断(route interrupts)。PLIC是可编程的，PLIC可以中断到其中一个CPU核心，或者到第一个可处理中断的核心core**。

​	在一种情况下，即没有任何core可以处理中断(none of the cores can take an interruptive at the point)，比如它们正在处理另一个中断，就会先关闭这个中断(disabled interrupts) 。

​	**PLIC会保留中断，直到有一个CPU核能处理中断。PLIC需要保存一些内部数据来跟踪中断状态**。RISC-V中具体的中断流程，如下：

1. PLIC会通知当前有一个待处理的中断(a interrupt pending)
2. 其中一个CPU核会声称(claim)接收中断
3.  这时PLIC就会把这个中断交给该CPU核
4. CPU核处理中断
5. CPU通知PLIC中断处理完毕
6. PLIC不再保存这个中断信息(被处理完毕的)

![img](https://pic2.zhimg.com/80/v2-929e11e9092c20b0ec071f3fad47dfa9_1440w.webp)

---

**问题：when each heart holds the PLIC，PLIC有没有什么机制保证公平性(ensure fairness)？**

**回答：这取决于内核以什么方式对PLIC进行编程。PLIC只是分发中断，是内核对PLIC编程，告诉它按什么策略分发中断。实际上中断是有优先级的，内核可以决定什么中断更重要，这有很大的灵活性**。

## 9.3 设备驱动driver

​	接下去介绍和硬件中断有关的软件部分内容。**通常来说，管理设备的代码被称为驱动(driver)**。在很多操作系统中，驱动代码加起来可能比内核还大，因为每个设备，都需要一个驱动。

​	所有的驱动代码都在内核中，我们今天要看的是UART设备的驱动，`uart.c`的代码。

​	**大多数驱动，都有一个结构，即分为底部(bottom)和顶部(top)两个部分。**

+ **bottom：最下面的部分是中断处理器(interrupt handler)，当CPU接收到一个来自设备的中断，会调用设备相应的中断处理器**。正如"9.1 硬件中断-概述"所说，中断处理器并不运行在任何特定的进程上下文(any context of any specific process)中，它只是处理中断。
+ **top：通常是用户进程或者内核的其他部分调用的接口**。对于UART来说，这里有read/write接口，可以被更高层级的代码调用。驱动的upper part上部通常与用户进程交互，并进行数据的读写。

​	通常驱动driver会在top和bottom之间维护一些队列，top的代码从队列读写数据，而bottom中断处理器要么放入codes，要么发送或接收。假设是接收，bottom的中断处理器同时会向队列里写数据。

​	**driver维护的队列将top和bottom分开，并允许设备与CPU上的代码并行运行**。

​	由于中断处理器没有运行在任何可调用读写的进程context中，所以通常有一些限制，比如页表的使用。

​	

![img](https://pic4.zhimg.com/80/v2-54f1d34dce65574f158797bc459e5c07_1440w.webp)

## 9.4 驱动编程programming device

> [9.3 设备驱动概述 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/339796706) <= 图片来源

​	**通常来说，驱动编程是通过memory mapped I/O完成的**。

​	正如前面在RISC-V或者SiFive看到的，这些设备出现在物理地址的特定区间内，具体区间由设备或者主板制造商决定。**操作系统需要通过普通的load/store指令对这些设备所在的物理地址区间进行编程**。**这里load/store实际就是读写设备的控制寄存器(They read and write control register of the device)**。

​	比如当你在网卡的控制寄存器(control register)存储数据时，就会触发网卡发送packet。这里load/store指令不是简单地读写内存，而是侧面影响了设备（或者说操作设备）。你需要阅读设备的文档，了解具体流程。

​	下图是SiFive主板中设备的内存映射或物理地址(the memory map or physical address to devices)。`0x02000000`对应CLINT；`0x0C000000`对应PLIC，平台级的中断控制器；`0x10000000`对应UART0。（QEMU模拟中，和这里说的不完全相同）

![img](https://pic1.zhimg.com/80/v2-4d8d5d90771e57020ec040cddcc3f7f0_1440w.webp)

​	下面是UART的文档，QEMU用这个模拟的设备来与键盘和Console进行交互。下面的表格展示芯片的控制寄存器，例如对于控制寄存器000，当你在执行load加载指令时，它会保存数据；如果你执行store指令，它会将数据写到寄存器中，并复制传输到其他地方（outside on the wire）。UART基本上是一个可以让你通过串口(cereal line)发送bits的设备，有单独的发送线sent line和单独的接收线receive line。基本上你取一个字节时，它们会在这一行(this single line)上进行复制或者序列化(multiplexed or serialized)，然后送到线路另一侧的UART chip，另一侧的UART chip能够将数据bits重新组成一个字节。

​	这里还有很多其他可操作的东西，但对我们最重要的是寄存器。比如控制寄存器001，即中断允许寄存器(interrupt enabled register)，我们可以对它进行编程，使UART产生中断。

​	实际上对于一个寄存器，其中每个bit都有不同的作用，对于001寄存器，即IER寄存器，用于接收线中断(line status interrupt)和传输保持寄存中断(transmit hoding registering interrupt)。

![img](https://pic4.zhimg.com/80/v2-119bfc940c0cb9f7d836ab13599e818f_1440w.webp)

---

<u>问题：如果你把数据写到trasmit holding register，然后再次写入，那么能保证其那一个数据不会被覆盖吗？</u>

<u>回答：这是我们需要注意的事情之一，我们通过load将数据写入寄存器中，之后UART chip会通过串口线将字节发送出去。当send完成，UART会生成一个中断，告知内核已经处理完当前接收的字节，可以给我下一个字节了。所以内核和设备之间需要遵守一些协议，才能确保工作流程正常。比如我们正在用的这个特殊的UART，内部有一个FIFO，它可以buffer最多16字符。但是如果阻塞16字符后，再次写入还是会造成数据覆盖</u>。

## 9.5 XV6中断机制初始化

> [9.4 在XV6中设置中断 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/339797175) <= 图片出处
>
> [波特率_百度百科 (baidu.com)](https://baike.baidu.com/item/波特率/2153185?fr=aladdin)
>
> 在电子通信领域，波特（Baud）即调制速率，指的是有效数据讯号调制载波的速率，即单位时间内载波调制状态变化的次数。波特率表示单位时间内传送的[码元](https://baike.baidu.com/item/码元/10525003?fromModule=lemma_inlink)符号的个数，它是对符号传输速率的一种度量，它用单位时间内[载波](https://baike.baidu.com/item/载波/3441949?fromModule=lemma_inlink)调制状态改变的次数来表示，波特率即指一个单位时间内传输符号的个数。

​	比如`$ ls`，是如何工作的。

1. 它将`$`放入了UART设备的寄存器中
2. UART生成一个中断（字符被发送时，就产生一个中断）
3. QEMU模拟的线路的另一端的UART chip连接到Console

​	对于`ls`，这个是用户输入的字符：

1. 当你键盘按下一个按键
2. 由于键盘连接到了UART的receive line，UART chip接收，序列化字符
3. 当前UART chip通过串口线发送到另一端UART芯片

4. 另一端UART chip将数据bits合并成一个Byte，然后产生中断
5. 产生的中断告知处理器这里有一个来自键盘的字符
6. interrupt handler处理来自UART的字符

---

​	RISC-V有很多与中断相关的寄存器，这里介绍一些：

+ SIE（Supervisor Interrupt Enable）寄存器

  + 1bit (E)专门针对外设中断(external interrupts)，例如UART的中断
  + 1bit (S)专门针对软件中断(software interrupts)，软件中断可能由一个CPU核触发给另一个CPU核。
  + 1bit (T)专门针对定时器中断(timer interrupts)

  这节课只关注外设中断。

+ SSTATUS（Supervisor Status）寄存器

  + 1bit专门用于当前core关闭或打开中断(disabled and enable interrupts on this particular core)

+ SIP（Supervisor Interrupt Pending）寄存器

  当发生中断时，处理器查看SIP，看实际是什么中断。

+ SCAUSE寄存器

  用于表明当前状态的原因是中断

+ STVEC寄存器

  当trap、page fault、interrupt发生时，保存当前在CPU运行的用户程序的PC程序计数器。后面才能正常恢复程序运行。

​	当然，每个CPU核有各自的SIE、SSTATUS寄存器。除了通过SIE寄存器来单独控制特定的中断，还可以通过SSTATUS寄存器的1bit来控制所有的中断。

​	SCAUSE、STVEC之前讨论过了，这里不会再详细讲述。对于system calls、page fault、interrupt或其他场景，SCAUSE、STVEC的工作模式基本都是一样的。

---

​	接下来看看XV6是如何对其他寄存器进行编程，使得CPU处于一个能接受中断的状态。

​	首先看看`start.c`的start函数。当机器启动时，调用start函数，它在mmode下运行。start中马上就关闭了页表(disable paging)，因为后续内核会设置页表。这里基本上所有异常的中断都设置在supervisor mode。然后设置SIE寄存器来接收外设中断、软件中断、定时器中断。

![img](https://pic2.zhimg.com/80/v2-36feba2f8500100adca1be3cfba9bc85_1440w.webp)

​	接下去看看main函数如何处理外设中断。我们的第一个外设是console，可以看到调用consoleinit进行初始化。下面调用plicinit函数初始化PLIC。再往下，调用plicinithart函数。最后，程序调用了scheduler函数。

![img](https://pic1.zhimg.com/80/v2-961956d1043694c646c3ec1db476f584_1440w.webp)

​	查看位于`console.c`的consoleinit函数，先初始化锁，然后调用位于`uart.c`的uartinit函数。

![img](https://pic2.zhimg.com/80/v2-f267078c7bb415c4524b716793b9638d_1440w.webp)

​	这个函数实际上就是，配置好UART chip，然后使其能够被正常使用。这里先关闭中断，然后设置波特率(board rate)，设置字符长度为8bit，重制UART内部的FIFO，然后再重新打开中断。

​	运行完uartinit函数之后，原则上UART就可以生成中断了。虽然我们还没有具体编程PLIC，所以就算有中断也不能被CPU感知。

![img](https://pic2.zhimg.com/80/v2-2f696f280bea21a8e0f0c3705a7e2399_1440w.webp)

​	PLIC在物理内存的`0XC0000000`，我们写PLIC的方式，基本上就是取PLIC的数字或者地址。下面`uint32`因为PLIC寄存器是32bit的。第一行使PLIC能接受UART的中断。PLIC会路由中断，所以这里实际就是中断会从左边传到PLIC中，然后PLIC会接收这些中断。第二行，设置PLIC接收来自IO磁盘的中断（本节不介绍）。

​	plicint由0号CPU运行。

![img](https://pic4.zhimg.com/80/v2-a7c25537e62eea1bb521e787624d9073_1440w.webp)

​	后续每个核心都在调用plicinithart函数，表明自己对来自UART、VIRTIO的中断感兴趣。然后因为我们忽略中断的优先级，下一行代码我们将优先级设置为0。

​	每个CPU核都必须向PLIC表明它对接收中断感兴趣。至此，对于PLIC，我们有了生成中断的外设，PLIC可以传递中断到单个CPU核，但CPU核还没有设置好接收中断，因为我们还没有设置好SSTATUS寄存器。

![img](https://pic4.zhimg.com/80/v2-8de86c2181e73d18d7b1a9ea7d25017f_1440w.webp)

​	`proc.c`的scheduler函数。现在整个机器基本上就是一个处理器。scheduler函数主要是运行进程。但是在实际运行进程之前，会执行intr_on函数来使得CPU能接收中断。

![img](https://pic4.zhimg.com/80/v2-864eee7c4e0297ad50846e64c2d0829b_1440w.webp)

​	intr_on函数，在SSTATUS寄存器设置中断标志位。之后，如果PLIC正好有pending的中断，CPU核就能接收到中断。

![img](https://pic3.zhimg.com/80/v2-e4d9f2eb238358eb89d8093456cad912_1440w.webp)

---

问题：什么是波特率(board rate)？

回答：串口线的传输速率。

问题：哪些核在intr_on之后打开了中断？

回答：每个核都会这么做。

## 9.6 UART驱动top部分(用户接口)

> [9.5 UART驱动的top部分 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/339830755) <= 图片出处

+ 在Unix系统中，设备用文件表示。

---

​	接下来，我们讨论如何在Shell程序中输出提示符`$`到console。

​	我们再看看init.c的main函数，这是系统启动后运行的第一个进程。

​	首先这个进程的main函数里，通过mdnod创建了一个代表console的设备，对应文件描述符0，再用dup创建stdout和stderr。最终文件描述符0，1，2都用来表示console。

![img](https://pic4.zhimg.com/80/v2-ab510a4dbf60ffa1b3d6413b8a07e69b_1440w.webp)

​	接下去看看`sh.c`，Shell程序打开文件描述符2，打印提示符`$`。

​	尽管console背后的是UART设备，但从应用程序角度看，shell程序就是再操作一个普通文件，只是向文件描述符2写了数据。

![img](https://pic4.zhimg.com/80/v2-1ddc59e9168e6c08179079218bbba29f_1440w.webp)

​	这里我们再看看fpintf的具体实现。其内部请求了write系统调用。在这个例子中，fd对应文件描述符2，c则是字符"$"。因此我们将保存"$"的内存地址，传递给文件描述符进行写入1个字符。

![img](https://pic3.zhimg.com/80/v2-57c4f6fb1b03d91a3238cb184bcc0fb2_1440w.webp)

​	基本上Shell输出的每个字符都会触发一个write系统调用。回顾之前所学，最后write系统调用内部会执行到`sysfile.c`的sys_write函数。包含字符"$"的地址，成为filewrite。

![img](https://pic2.zhimg.com/80/v2-a33597c62cabd6e30d21fd2693c947a1_1440w.webp)

​	filewrite函数中，会先判断文件描述符的类型，如果这是一个mknod生成的device，它会为这个特定的divice调用write函数，所以它就会在console调用write函数。

![img](https://pic3.zhimg.com/80/v2-4a69f0681b29afddd855e8ca14e6211e_1440w.webp)

​	被调用的consolewrite函数，先通过either_copyin将字符复制，然后调用uartputc函数将字符写到UART设备。这里可以把console想象成驱动，而我们看的consolewrite是驱动的top部分的代码，实际是`uart.c`的uartputc函数在打印字符。

![img](https://pic3.zhimg.com/80/v2-4df7bf7d48d18ed4403a706212941bd2_1440w.webp)

​	看看uartputc函数实现，在UART的top内部有一个32字符用来发送数据的buffer，同时还有一个读指针、一个写指针，用来构建一个环形的buffer。即一个是为消费者提供的读指针，一个是为生产者提供的写指针。在我们例子中，Shell是生产者，它在函数中做的第一件事就是判断环形buffer是否满了。如果读写指针相同，那么buffer是空的；如果写指针加1等于读指针，那么buffer满了。buffer满了再写数据是无意义的，因为显然UART还在忙，需要先尝试发送前面的31个字符。当buffer满了，shell就会休眠一段时间，内核先运行其他程序。当然就我们现在的情况而言，buffer必然没有满，毕竟"$"是第一个字符。驱动程序将字符放进buffer，写指针更新，之后调用uartstart函数，通知设备执行操作。

![img](https://pic2.zhimg.com/80/v2-f19b098758a418400c56886312aeae4d_1440w.webp)

​	uartstart首先检查当前设备是否空间，如果设备忙则休眠；否则从buffer中读取字符，放到THR（Transmission Holding Register）传输寄存器中。

![img](https://pic3.zhimg.com/80/v2-c5800f34a5af77cdad5bb917a432896e_1440w.webp)

​	基本上shell请求系统调用就是告诉设备这里有一个字节需要发送，一旦数据到了设备，系统调用会返回到用户程序，shell继续运行，然后请求read system call，从键盘读取输入。与此同时，UART设备发出数据，在某些时间点收到中断，因为之前设置了要处理UART设备中断。

## 9.7 UART驱动bottom部分(中断处理器)

​	在我们向console输出字符的时候，如果发生了中断，RISC-V会怎么处理？

​	之前我们已经在SSTATUS寄存器打开了中断，所以处理器会被中断。假设键盘生成了一个中断发送给PLIC，PLIC会路由中断给一个特定CPU核。接收中断的核心如果SIE寄存器设置了E bit，会发生以下事情：

1. 清除SIE寄存器相应的bit，阻止CPU核被其他中断打扰。如果后续想接收其他中断，可以再次恢复SIE寄存器相应的bit。
2. 设置SEPC寄存器保存当前的PC程序计数器。 
3. 保存mode，我们例子Shell是user mode
4. 切换mode为supervisor mode
5. 修改PC寄存器的值为STVEC的值。在XV6中，STVEC保存的是uservec或者kernelvec函数的地址，具体取决于发生中断时运行的程序在用户空间还是内核空间。在我们例子中shell在user space，所以这里包含uservec函数的地址。前面trap的课程可以知道uservec函数会丢调用usertrap函数。



![img](https://pic2.zhimg.com/80/v2-cf6f4f91cc2c2842dc9c91a6636a33cd_1440w.webp)

​	接下来看看`trap.c`的usertrap函数的代码片段，其调用的devintr函数，通过SCAUSE寄存器判断当前中断是否属于来自外设的中断，如果是的，再调用plic_claim函数获取中断。

​	回到PLIC，我们看看这个plic_claim。基本上归结起来，就是特定的CPU比如CPU0和CPU1告诉PLIC，CPU1正在请求这个特定的中断，PLIC会进行返回，来获取实际进来的中断的IRQ。这种情况下，对于UART来说， 返回的中断号是10。

​	然后下面if判断，如果是UART中断，则执行uartintr函数。uart中断函数，会让字符脱离UART。我们基本是在第一个寄存器接收到UART字符的，然后它调用consoleintr完成剩余工作。如果在读端有一个字符，那么我们会调用console中断，但现在读端没有字符，因为我们还没有向键盘输入数据。我们只是在传送一个字符，所以这里uartgetc返回-1。

​	基本上代码会直接运行到uartstart函数，这个函数会将shell存储在buffer的任意字符送出。实际上在提示符“$”之后，Shell还会输出一个空格字符，write系统调用可以在UART发送提示符"$"的同时，并发的将空格字符写入到buffer中。所以UART的发送中断触发时，可以发现在buffer中还有一个空格字符，之后会将这个空格字符送出。

![img](https://pic3.zhimg.com/80/v2-343de51adf96fa15186c737cb4606312_1440w.webp)

![img](https://pic2.zhimg.com/80/v2-ea2bff943a57009f9264d172417661e5_1440w.webp)

![img](https://pic1.zhimg.com/80/v2-5a3125d8aa016f547467e6fc0eb22924_1440w.webp)

---

问题：我知道UART对于键盘来说很重要，来自于键盘的字符通过UART走到CPU再到我们写的代码。但是我不太理解UART对于Shell输出字符究竟有什么作用？因为在这个场景中，并没有键盘的参与。

回答：显示器和UART也是连接的。UART连接了键盘和显示器，即console。QEMU通过模拟的UART，与console进行交互。而console的作用就是将字符展示到显示器。

## 9.8 中断的并发设计

​	前面提到的UART设备和CPU是并行运行的。 

+ 设备和CPU工作是并行运行的，这种并行类型被称为生产者-消费者并行(producer consumer parallelism)。

+ 中断会停止当前运行的程序。用户进程或内核进程都如此，这意味着即使内核代码也不是串行运行的。两个内核指令之间，也可能被中断打断执行，这取决于中断是否打开。如果有些代码执行不允许被中断，内核就需要临时关闭中断，确保这段代码的原子性。
+ 驱动的top和bottom部分是并行运行的。比如，shell再次调用write系统调用传输字符，代码会走到UART驱动的top部分，将字符写到UART的buffer中，与此同时另一个CPU核可能会收到来自UART的中断，进而执行UART驱动的bottom部分，查看buffer。驱动的top和bottom可以并行在不同CPU核上运行，我们这里通过lock来进行管理并行，因为这里有共享的数据即buffer，我们向确保同一时间buffer只能被一个CPU核操作。

---

​	这里我们主要关注生产者-消费者并行(producer consumer parallelism)。

​	在前面的例子中，shell调用uartputc最后往UART的buffer中写数据，这是producer的操作；而Intercept handler中断处理器通过uartintr函数作为consumer从buffer中读数据，再通过UART设备发送。

![img](https://pic3.zhimg.com/80/v2-a92356157b2acd3248c20dd0354b681a_1440w.webp)

​	以上就是Shell输出提示符"$"的全部内容。如你们所见，过程还挺复杂的，许多代码一起工作才将这两个字符传输到了console。

---

问题：这里UART的buffer对所有CPU核都是共享的吗？

回答：buffer在内存中只有一份，所有CPU核都是并行地与这一份数据交互，所以我们才需要lock。

问题：uartputc中的sleep，它怎么知道该让shell去sleep？这里只有个地址而已。`sleep(&uart_tx_r, &uart_tx_lock);`

回答：sleep会将当前运行的进程存放于sleep数据中，它传入的参数是需要等待的信号，在这种情况下，地址基本上只有一个channel ID，那里会显示是什么sleeping了。这个例子传入的是uart_tx_r的地址，在uartstart函数中，一旦buffer中有了空间，它就会醒来，会调用与sleep对应的wakeup函数`wakeup(&uart_tx_r)`，传入相同的地址，任何在这个地址的进程都会被环境。具体实现，以后会说。 <u>这种sleep和wakeup组合使用的现象，有时称为条件同步</u>。

## 9.9 UART读取键盘输入

> [9.8 UART读取键盘输入 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/339831314) <= 图文摘自此博文

​	UART另一侧，shell通过read读取键盘输入，而read底层实现中调用了fileread函数，当读取的文件类型是设备，就会调用设备对应的read函数。

![img](https://pic4.zhimg.com/80/v2-ff7a292ff54dcb1ff7f331a26d5ede0b_1440w.webp)

​	在我们的例子中，read函数就是`console.c`文件中的consoleread函数。

![img](https://pic2.zhimg.com/80/v2-44bddf56bc574ed399a0c18a68bb0e51_1440w.webp)

​	它和uart结构类似，顶部也有一个buffer，其可存储128字符。其他的基本一致，也有生产者与消费者并行，只不过这里shell是消费者，因为shell从buffer中读取数据，键盘作为生产者，将数据写到buffer。

![img](https://pic1.zhimg.com/80/v2-5358a6201ea009b2953ca56b72d5b840_1440w.webp)

​	看consoleread函数实现，可知读写指针一致时（buffer为空），进程就会休眠。如果shell打印完"$"提示符后，键盘没有后续输入，shell进程就会休眠。内核会让shell进程休眠直到键盘输入字符。

![img](https://pic3.zhimg.com/80/v2-8944b5673721ee58d4b4e8952409da66_1440w.webp)

​	某个时间点，假设用户通过键盘输入了“l”，这会导致“l”被发送到主板上的UART芯片，产生中断之后再被PLIC路由到某个CPU核，之后会触发devintr函数，devintr可以发现这是一个UART中断，然后通过uartgetc函数获取到相应的字符，之后再将字符传递给consoleintr函数。

​	默认情况下，字符会通过consputc，输出到console上给用户查看。之后，字符被存放在buffer中。在遇到换行符的时候，唤醒之前sleep的进程，也就是Shell，再从buffer中将数据读出。

​	所以这里也是通过buffer将consumer和producer之间解耦，这样它们才能按照自己的速度，独立的并行运行。如果某一个运行的过快了，那么buffer要么是满的要么是空的，consumer和producer其中一个会sleep并等待另一个追上来。

## 9.10 中断机制的演变

> [9.9 Interrupt的演进 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/339831441) <= 更详细

+ Unix初期，中断机制使得能够处理外设数据，当时硬件情况下速度够用。
+ 现在硬件速度很快，中断机制显得很慢。因为中断处理器就是保存或者恢复寄存器，且需要很多步骤次啊能真正处理到中断数据。如果一个设备在高速产生中断，处理器就很难跟上。
+ 为减轻处理器的负担，现代设备在中断机制上比以前处理更多事情。在产生中断之前，设备会先执行大量操作，以减轻CPU处理负担。

​	比如，现代千兆网卡，收到大量小packets，如果每接收一个包都发起中断，那CPU会被疯狂占用以处理中断。流量大时，可能甚至需要CPU每us处理一次中断。

​	可以用polling解决上诉网卡问题。除了依赖Interrupt，CPU可以一直读取外设的控制寄存器，来检查是否有数据。对于UART来说，我们可以一直读取RHR寄存器，来检查是否有数据。现在，CPU不停轮询设备，直到设备有了数据。轮询看上去有点浪费CPU周期，对于一个慢设备，轮询不合适。但是如果对于一个快设备，比起处理中断的开销，轮询更省性能。

​	polling轮询机制：

+ 对于慢设备：浪费CPU周期
+ 对于快设备：节省中断开销

​	对于高性能网卡，如果有大量小包流入，则使用轮训机制。

​	一些精心设计的驱动，能够在轮询polling和中断interrupt之间动态切换。

---

问题：我看到uartinit只被调用了一次，是这个导致所有的CPU核都共用一个buffer吗？

回答：一个buffer只针对一个UART设备，而这个buffer会被所有的CPU核共享。驱动通过lock锁确保同一时间只有一个CPU核能调用uartputc，进而使得多个CPU核串行向console打印输出。

问题：用锁，是因为多个CPU核，但只有一个console，对吗？

回答：驱动top和bottom部分可以并行运行，所以一个CPU核可以执行uartputc函数，另一个CPU核可执行uartintr函数。我们需要确保它们不会混在一起，锁确保它们操作buffer时是串行执行的。

追问：上面用锁，是不是意味着有时候所有CPU核需要等待某个CPU核的处理？因为中断需要等待，而且没有其他需要做的了。

回答：假设没有其他进程在运行这本身不太可能。这里不是死锁。如果有多个uartputc函数被调用，并且buffer是满的，那么中断时，它们就会释放锁。举个例子，我们回到uartputc，它调用sleep，而sleep将锁作为一个参数， 内部在将进程置于休眠状态之前，会释放锁。然后休眠结束回来执行时，会重新锁上。

问题：当UART触发中断时，所有CPU核都能收到中断吗？

回答：这取决于对PLIC如何编程。对于XV6，所有CPU核都能收到中断，但只有一个CPU核会claim对应的中断。所以如果你回到PLIC，当你得到一个中断，你会call this PLIC claim，那个特定CPU会得到IRQ，然后PLIC会记得IRQ现在正在被服务，所以不会给其他核。

问题：之前看到打印的输出经常时交错的，是不是因为锁只在putc周围，但是来自多个CPU对putc交错调用。仅保证单个打印可以原子性？

回答：是的，只保证单个打印原子性。

问题：书上说计时器中断是在machine mode下处理的，我们在做trap实验时，它在哪里被处理了？

回答：在机器启动时，有uartstart在machine mode下运行，它给timer chip编程，有timerinit函数，该函数基本是为CLINT编程，也就是本地中断器(local interruptor)，用于在时钟中断发生时产生中断。machine mode的trap handler是一个被叫做timervec的函数，当一个time interrupt发生时被调用。所以内核在user mode或supervisor mode运行并且CLINT产生了一个中断时，它就会切换到machine mode，然后调用timervec函数。流程和supervisor mode、user mode类似。如果你看一下`kernelvec.s`就能找到timervec函数实现，其重新编程来产生中断，然后给supervisor传输中断，进入supervisor mode，最后在mret。我们假设内核被一个定时器芯片中断，我们就进入了machine mode，后面又从machine mode回到supervisor mode。 

追问：一开始进入machine mode有什么特殊理由吗？

回答：为什么计时器芯片会进入machine mode。从我们角度上看，如果将计时器中断直接放到supervisor mode，而不用处理machine mode，那就太好了，但对于这个特殊的芯片来说是行不通的。

# Lecture10 多处理器和锁(Multiprocessor and Locks)

## 10.1 为什么需要锁Lock

> [10.1 为什么要使用锁？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343285126) <= 图来源，部分文字摘抄

​	应用利用多核处理器并行执行系统调用时，往往涉及内核中共享的数据结构的访问，比如proc、ticks、buffer cache等等。我们使用锁来协调对共享数据的访问（比如一个核正要写入，一个核正要读取），以确保数据的一致性。Lock for correct sharing。

​	但是实际的情况有些令人失望，<u>因为我们想要通过并行来获得高性能，我们想要并行的在不同的CPU核上执行系统调用，但是如果这些系统调用使用了共享的数据，我们又需要使用锁，而锁又会使得这些系统调用串行执行，所以最后锁反过来又限制了性能</u>。Lock can limit performance。

![img](https://pic1.zhimg.com/80/v2-ca9ed0890af83d313f17acb12bc2c064_1440w.webp)

+ 绿，时钟频率到达瓶颈；进而，蓝，单核性能达到瓶颈
+ 深红，晶体管数量不断增加；进而，深蓝，核数增加

​	处理器单核性能提升缓慢，转而向多核发展。同时操作系统也必须能适配多核处理器，使应用能充分利用多核的性能。

​	**回到前文提到用锁保证数据一致性。当一份共享数据同时被读写时，如果没有锁的话，可能会出现race condition，进而导致程序出错**。

​	`kalloc.c`文件中的kfree函数会将释放的page保存于freelist中

![img](https://pic2.zhimg.com/80/v2-6141b03f42a0570f389f03deb085aac1_1440w.webp)

​	freelist是XV6中的一个非常简单的数据结构，它会将所有的可用的内存page保存于一个列表中。这样当kalloc函数需要一个内存page时，它可以从freelist中获取。从函数中可以看出，这里有一个锁kmem.lock，在加锁的区间内，代码更新了freelist。**现在我们将锁的acquire和release注释上，这样原来在上锁区间内的代码就不再受锁保护，并且不再是原子执行的**。

![img](https://pic3.zhimg.com/80/v2-4ee279f4e25f818ddddc543c23d74d5e_1440w.webp)

​	修改代码后，重新make qemu编译XV6，运行usertest。

​	这里通过qemu模拟了3个CPU核，这3个核是并行运行的。race condition不一定会发生，因为当每一个核在每一次调用kfree函数时，对于freelist的更新都还是原子操作，这与有锁是一样，这个时候没有问题。<u>有问题的是，当两个处理器上的两个线程同时调用kfree，并且交错执行更新freelist的代码</u>。

![img](https://pic3.zhimg.com/80/v2-21d10ac4cf9204432b52b849b5dc49de_1440w.webp)

​	可以看出来某些test没有通过。

## 10.2 什么是锁的抽象

​	有一个structure结构为lock。其有两个直观的锁方法：

+ `acquire(&lock)`：接收一个lock作为参数，其内部实现强保证同一时间只有一个进程能获得锁。即同一时间只有一个CPU核能获得锁。
+ `release(&lock)`：接收一个lock作为参数，同一时间只有持有当前lock的进程调用release释放锁后，其他进程才有机会通过acquire获取锁。

​	**acquire和release之间的代码，通常被称为临界区(critical section)**。位于临界区(critical section)的代码，要么都执行，要么都不执行。所以永远也不可能看到位于critical section中的代码，如同在race condition中一样在多个CPU上交织的执行，进而能够避免race condition。

​	**现在的程序通常会有许多锁**。实际上，XV6中就有很多的锁。为什么会有这么多锁呢？因为锁序列化了代码的执行。如果两个处理器想要进入到同一个critical section中，只会有一个能成功进入，另一个处理器会在第一个处理器从critical section中退出之后再进入。所以这里完全没有并行执行。

​	<u>如果内核中只有一把大锁，我们暂时将之称为big kernel lock。基本上所有的系统调用都会被这把大锁保护而被序列化。系统调用会按照这样的流程处理：一个系统调用获取到了big kernel lock，完成自己的操作，之后释放这个big kernel lock，再返回到用户空间，之后下一个系统调用才能执行。这样的话，如果我们有一个应用程序并行的调用多个系统调用，这些系统调用会串行的执行，因为我们只有一把锁</u>。所以通常来说，例如XV6的操作系统会有多把锁，这样就能获得某种程度的并发执行。**如果两个系统调用使用了两把不同的锁，那么它们就能完全的并行运行**。

## 10.3 什么时候锁When to lock

> [10.3 什么时候使用锁？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343285775) <= 图出处，部分文字摘自该博文

​	锁能保证数据一致性，但同时也会限制并发性。那**什么时候需要锁？一个保守且简单的规则即：如果两个进程访问了一个共享的数据结构，并且其中一个进程会更新共享的数据结构，那么就需要对于这个共享的数据结构加锁**。

​	*不加锁的程序通常称为lock-free program，不加锁的目的是为了获得更好的性能和并发度，不过lock-free program比带锁的程序更加复杂一些。不是本节要讲解的*。

​	矛盾的是，有时候这个规则太过严格，而有时候这个规则又太过宽松了。除了共享的数据，在一些其他场合也需要锁，例如对于printf，如果我们将一个字符串传递给它，XV6会尝试原子性的将整个字符串输出，而不是与其他进程的printf交织输出。尽管这里没有共享的数据结构，但在这里锁仍然很有用处，因为我们想要printf的输出也是序列化的。

​	那么我们能通过自动的创建锁来自动避免race condition吗？如果按照刚刚的简单规则，一旦我们有了一个共享的数据结构，任何操作这个共享数据结构都需要获取锁，那么对于XV6来说，每个结构体都需要自带一个锁，当我们对于结构体做任何操作的时候，会自动获取锁。

​	可是如果我们这样做的话，结果就太过严格了，所以不能自动加锁。接下来看一个具体的例子。

​	假设我们有一个对于rename的调用，这个调用会将文件从一个目录移到另一个目录，我们现在将文件d1/x移到文件d2/y。

​	如果我们按照前面说的，对数据结构自动加锁。现在我们有两个目录对象，一个是d1，另一个是d2，那么我们会先对d1加锁，删除x，之后再释放对于d1的锁；之后我们会对d2加锁，增加y，之后再释放d2的锁。这是我们在使用自动加锁之后的一个假设的场景。

​	在我们完成了第一步，也就是删除了d1下的x文件，但是还没有执行第二步，也就是创建d2下的y文件时。其他的进程会看到什么样的结果？是的，其他的进程会看到文件完全不存在。这明显是个错误的结果，因为文件还存在只是被重命名了，文件在任何一个时间点都是应该存在的。但是如果我们按照上面的方式实现锁的话，那么在某个时间点，文件看起来就是不存在的。

​	所以这里正确的解决方法是，我们在重命名的一开始就对d1和d2加锁，之后删除x再添加y，最后再释放对于d1和d2的锁。

![img](https://pic3.zhimg.com/80/v2-e3f7e375891f644b01322677752172a2_1440w.webp)

​	<u>在这个例子中，我们的操作需要涉及到多个锁，但是直接为每个对象自动分配一个锁会带来错误的结果。在这个例子中，锁应该与操作而不是数据关联，所以自动加锁在某些场景下会出问题</u>。

![img](https://pic4.zhimg.com/80/v2-ba5b913c1838be49df3b1a5df53acc8f_1440w.webp)

---

问题：有没有可能两个进程同时acquire锁，然后同时修改数据？
回答：不会。之后会看acquire具体实现，其实现需要保证同一时间只能有一个进程获取到锁。

问题：可不可以在访问某个数据结构的时候，就获取所有相关联的数据结构的锁？
回答：这是一种实现方式。但是这种方式最后会很快演进成big kernel lock，这样你就失去了并发执行的能力，但是你肯定想做得更好。这里就是使用锁的矛盾点了，如果你想要程序简单点，可以通过coarse-grain locking（注，也就是大锁），但是这时你就失去了性能。

## 10.4 锁的特性

> [10.4 锁的特性和死锁 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343285964) <= 图出处，部分文字摘自该博文

1. 避免丢失更新(lock help avoid lost updates)

   如果你回想我们之前在`kalloc.c`中的例子，丢失更新是指我们丢失了对于某个内存page在kfree函数中的更新。如果没有锁，在出现race condition的时候，内存page不会被加到freelist中。但是加上锁之后，我们就不会丢失这里的更新。

2. 确保多个操作具有原子性(locks make multi-step operations atomic)

   比如前面提到临界区，在critical section的所有操作会都会作为一个原子操作执行。

3. 保证共享数据的不变性(lock help maintain an invariant)

   <u>如果某个进程acquire了锁并且做了一些更新操作，共享数据的不变性暂时会被破坏，但是在release锁之后，数据的不变性又恢复了</u>。你们可以回想一下之前在kfree函数中的freelist数据，所有的free page都在一个单链表上。但是在kfree函数中，这个单链表的head节点会更新。freelist并不太复杂，对于一些更复杂的数据结构可能会更好的帮助你理解锁的作用。

## 10.5 死锁Deadlock

> [10.4 锁的特性和死锁 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343285964) <= 图出处，部分文字摘自于该博文

​	正确的使用锁，能够避免race contraction；但不恰当的使用会导致一些锁持有的问题，其中最典型的问题就是**死锁**。

​	死锁场景举例1，假设有一个不可重入锁lock被同一进程acquire两次，则造成死锁。因为该锁未被release就重新被同一个进程acquire，导致当前进程无限等待锁释放。这是死锁的一个最简单的例子，XV6会探测这样的死锁，如果XV6看到了同一个进程多次acquire同一个锁，就会触发一个panic。

![img](https://pic2.zhimg.com/80/v2-2c814d181b839818cfad7185d8593101_1440w.webp)

​		死锁场景举例2：两个CPU以不同的顺序acquire两种锁，两个CPU核第一次lock的锁正是对方第二次lock需要的锁，这也造成了死锁。这种场景也被称为deadly embrace。这里的死锁就没那么容易探测了。**对于这种场景，其中一种解决方法即，要求多种锁以固定顺序获取**（order locks，acquire locks in orders）。

​	以对于一个系统设计者，你需要确定对于所有的锁对象的全局的顺序。例如在这里的例子中我们让d1一直在d2之前，这样我们在rename的时候，总是先获取排序靠前的目录的锁，再获取排序靠后的目录的锁。如果对于所有的锁有了一个全局的排序，这里的死锁就不会出现了。

![img](https://pic2.zhimg.com/80/v2-fc512426e115f9b3f8c73b0cb1d64219_1440w.webp)

​	**<u>不过在设计一个操作系统的时候，定义一个全局的锁的顺序会有些问题，比如破坏代码的模块化设计</u>**。如果一个模块m1中方法g调用了另一个模块m2中的方法f，那么m1中的方法g需要知道m2的方法f使用了哪些锁。<u>因为如果m2使用了一些锁，那么m1的方法g必须集合f和g中的锁，并形成一个全局的锁的排序。这意味着在m2中的锁必须对m1可见，这样m1才能以恰当的方法调用m2</u>。但是这样又**违背了代码抽象的原则**。在完美的情况下，代码抽象要求m1完全不知道m2是如何实现的。但是不幸的是，具体实现中，m2内部的锁需要泄露给m1，这样m1才能完成全局锁排序。<u>所以当你设计一些更大的系统时，锁使得代码的模块化更加的复杂了</u>。

---

问题：有必要对所有的锁进行排序吗？或者是否存在一些锁可以以任意方式排序。

回答：在上面的例子中，这取决于f和g是否共用了一些锁。如果你看XV6的代码，你可以看到会有多种锁的排序，因为一些锁与其他的锁没有任何关系，它们永远也不会在同一个操作中被acquire。如果两组锁不可能在同一个操作中被acquire，那么这两组锁的排序是完全独立的。但**操作同一锁集的函数需要在全局排序上达成一致**。

## 10.6 锁和性能取舍

> [10.5 锁与性能 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343286207) <= 部分文字摘自于该博文

​	前面提到，使用锁可能导致**死锁(deadlock)**，**破坏程序的模块化**("10.5 死锁Deadlock"提到，多进程加锁竞争的问题，可以通过强制加锁顺序解决，但这同时也相当于要求了模块之间需要互相了解对方的加锁逻辑)。

​	<u>基本上来说，如果你想获得更高的性能，你需要拆分数据结构和锁。如果你只有一个big kernel lock，那么操作系统只能被一个CPU运行。如果你想要性能随着CPU的数量增加而增加，你需要将数据结构和锁进行拆分</u>。

​	*那怎么拆分呢？通常不会很简单，有的时候还有些困难。比如说，你是否应该为每个目录关联不同的锁？你是否应该为每个inode关联不同的锁？你是否应该为每个进程关联不同的锁？或者是否有更好的方式来拆分数据结构呢？如果你重新设计了加锁的规则，你需要确保不破坏内核一直尝试维护的数据不变性。*

​	*如果你拆分了锁，你可能需要重写代码。如果你为了获得更好的性能，重构了部分内核或者程序，将数据结构进行拆分并引入了更多的锁，这涉及到很多工作，你需要确保你能够继续维持数据的不变性，你需要重写代码。通常来说这里有很多的工作，并且并不容易。*

​	通常来说，使用锁且改进代码的流程如下：

1. 先使用粗粒度锁(coarse-grained lock)完成代码开发。
2. 代码测试，看看多核并发性能
3. 如果多核竞争锁严重，最后其实导致代码串行化执行时间长，性能提高不高甚至更低，那么需要重构代码（拆分代码，缩小锁粒度等等）。

​	当然，凡事需要取舍，如果一个程序（代码片段）调用率不高，即使使用粗粒度大锁也不会有严重的并发竞争，那么不需要特地重构代码。因为重构/改进代码本身成本不低。

## 10.7 XV6中UART模块锁使用示例

> [10.6 XV6中UART模块对于锁的使用 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343286372) <= 图出处，部分文字摘自该博文

​	这里举例`uart.c`的代码。

![img](https://pic3.zhimg.com/v2-e6dac7d73a430d4e3d72d73c4306402e_r.jpg)

​	可看到这里只有一个`spinlock uart_tx_lock`。你可以认为对于UART模块来说，现在是一个coarse-grained lock的设计。<u>这个锁保护了UART的的传输缓存；写指针；读指针。当我们传输数据时，写指针会指向传输缓存的下一个空闲槽位，而读指针指向的是下一个需要被传输的槽位。这是我们对于并行运算的一个标准设计，它叫做**消费者-生产者模式(producer consumer parallelism)**</u>。

​	所以现在有了一个缓存，一个写指针和一个读指针。读指针的内容需要被显示，写指针接收来自例如printf的数据。我们前面已经了解到了锁有多个角色。第一个是保护数据结构的特性不变，数据结构有一些不变的特性，例如读指针需要追赶写指针；从读指针到写指针之间的数据是需要被发送到显示端；从写指针到读指针之间的是空闲槽位，锁帮助我们维护了这些特性不变。

![img](https://pic3.zhimg.com/80/v2-a1e3e398fb83a399ee67ec405b0f96c2_1440w.webp)

​	接下来看看`uart.c`中的uartputc函数。函数首先获得了锁，然后查看当前缓存是否还有空槽位，如果有的话将数据放置于空槽位中；写指针加1；调用uartstart；最后释放锁。如果两个进程在同一个时间调用uartputc，那么这里的锁会确保来自于第一个进程的一个字符进入到缓存的第一个槽位，接下来第二个进程的一个字符进入到缓存的第二个槽位。这就是锁帮助我们避免race condition的一个简单例子。<u>如果没有锁的话，第二个进程可能会覆盖第一个进程的字符</u>。

![img](https://pic4.zhimg.com/80/v2-55f11ebe1e7a905d7ffbe1953bb95657_1440w.webp)

​	接下来我们看一下uartstart函数。如果uart_tx_w不等于uart_tx_r，那么缓存不为空，说明需要处理缓存中的一些字符。锁确保了我们可以在下一个字符写入到缓存之前，处理完缓存中的字符，这样缓存中的数据就不会被覆盖。**锁确保了一个时间只有一个CPU上的进程可以写入UART的寄存器，THR。所以这里锁确保了硬件寄存器只有一个写入者**。

![img](https://pic3.zhimg.com/80/v2-c93afaec489ed00e8f0170ba93048afe_1440w.webp)

​	当UART硬件完成传输，会产生一个中断。在前面的代码中我们知道了uartstart的调用者会获得锁以确保不会有多个进程同时向THR寄存器写数据。但是<u>UART中断本身也可能与调用printf的进程并行执行。如果一个进程调用了printf，它运行在CPU0上；CPU1处理了UART中断，那么CPU1也会调用uartstart。因为我们想要确保对于THR寄存器只有一个写入者，同时也确保传输缓存的特性不变（注，这里指的是在uartstart中对于uart_tx_r指针的更新），我们需要在中断处理函数中也获取锁</u>。

​	所以，在XV6中，驱动的bottom部分（注，也就是中断处理程序）和驱动的top部分（注，uartputc函数）可以完全的并行运行，所以<u>中断处理程序也需要获取锁</u>。

![img](https://pic1.zhimg.com/80/v2-6de298caec565c1db4ba29670e74fb38_1440w.webp)

---

问题：UART的缓存中，读指针是不是总是会落后于写指针？
回答：从读指针到写指针之间的字符是要显示的字符，UART会逐次的将读指针指向的字符在显示器上显示，同时printf可能又会将新的字符写入到缓存。<u>读指针总是会落后于写指针直到读指针追上了写指针，这时两个指针相同，并且此时缓存中没有字符需要显示</u>。

## 10.8 锁的实现-理论篇

> [10.7 自旋锁（Spin lock）的实现（一） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343286617) <= 图出处，部分文字摘自该博文

​	**锁的特性就是只有一个进程可以获取锁，在任何时间点都不能有超过一个锁的持有者**。我们接下来看一下锁是如何确保这里的特性。

​	我们先来看一个有问题的锁的实现，这样我们才能更好的理解这里的挑战是什么。实现锁的主要难点在于锁的acquire接口，在acquire里面有一个死循环，循环中判断锁对象的locked字段是否为0，如果为0那表明当前锁没有持有者，当前对于acquire的调用可以获取锁。之后我们通过设置锁对象的locked字段为1来获取锁。最后返回。

![img](https://pic2.zhimg.com/80/v2-61502787dc655f1094a8e22513b6b755_1440w.webp)

​	如果锁的locked字段不为0，那么当前对于acquire的调用就不能获取锁，程序会一直spin。也就是说，程序在循环中不停的重复执行，直到锁的持有者调用了release并将锁对象的locked设置为0。在这个实现里面会有什么样的问题？答案即这里存在race condition。如果CPU0和CPU1同时到达A语句，它们会同时看到锁的locked字段为0，之后它们会同时走到B语句，这样它们都acquire了锁。这样我们就违背了锁的特性。

![img](https://pic2.zhimg.com/80/v2-d2104e4975b5fdae43880ca39d79bf51_1440w.webp)

---

​	为了解决以上的锁实现问题，有很多实现正确锁的方法。**最为常见的即依赖底层硬件指令支持，以实现加锁原子性**。

​	硬件提供指令保证test-and-set操作具有原子性。在RISC-V上，这个特殊的指令就是amoswap（atomic memory swap）。这个指令接收3个参数，分别是address，寄存器r1，寄存器r2。这条指令会先锁定住address，将address中的数据保存在一个临时变量中（tmp），之后将r1中的数据写入到地址中，之后再将保存在临时变量中的数据写入到r2中，最后再对于地址解锁。

![img](https://pic4.zhimg.com/80/v2-d0c5cd07e52a0301c31c417ecb3e149b_1440w.webp)

​	通过这里的加锁，可以确保address中的数据存放于r2，而r1中的数据存放于address中，并且这一系列的指令打包具备**原子性**。<u>大多数的处理器都有这样的硬件指令，因为这是一个实现锁的方便的方式</u>。

​	**这里我们通过将一个软件锁转变为硬件锁最终实现了原子性**。不同处理器的具体实现可能会非常不一样，处理器的指令集通常像是一个说明文档，它不会有具体实现的细节，具体的实现依赖于内存系统是如何工作的，比如说：

- **多个处理器共用一个内存控制器**，内存控制器可以支持这里的操作，比如给一个特定的地址加锁，然后让一个处理器执行2-3个指令，然后再解锁。因为所有的处理器都需要通过这里的内存控制器完成读写，所以内存控制器可以对操作进行排序和加锁。
- 如果内存位于一个共享的总线上，那么需要**总线控制器(bus arbiter)**来支持。总线控制器需要以原子的方式执行多个内存操作。
- 如果处理器有缓存，那么**缓存一致性协议(cache coherence protocol)**会确保对于持有了我们想要更新的数据的cache line只有一个写入者，相应的处理器会**对cache line加锁**，完成两个操作。

​	<u>硬件原子操作的实现可以有很多种方法。但是基本上都是对于地址加锁，读出数据，写入新数据，然后再返回旧数据</u>（注，也就是实现了atomic swap）。

## 10.9 XV6的acquire和release的实现

> [10.7 自旋锁（Spin lock）的实现（一） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343286617)
> [10.8 自旋锁（Spin lock）的实现（二） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343287063)
>
> 图片出自以上两博文，文字部分摘自以上两博文。

​	看看XV6中acquire和release的代码实现。先看看`spinlock.h`

![img](https://pic3.zhimg.com/80/v2-c6ffa1ad56156c140fbcbb6635d28bc6_1440w.webp)

​	locked表示锁是否被持有；name表示锁的名字（调试用）；cpu表示持锁的CPU核。

​	接下来看看`spinlock.c`的acquire具体代码实现。函数中通过while循环，执行test-and-set操作尝试加锁。实际上C的标准库已经定义了这些原子操作，所以C标准库中已经有一个函数`__sync_lock_test_and_set`，它里面的具体行为与之前描述的硬件指令行为是一样的。因为**大部分处理器都有的test-and-set硬件指令**，所以这个函数的实现比较直观。我们可以通过查看`kernel.asm`来了解RISC-V具体是如何实现的。下图就是atomic swap操作。

![img](https://pic3.zhimg.com/80/v2-fa01dc352abf23244cf48554c768679a_1440w.webp)

​	这里比较复杂，总的来说，一种情况下我们跳出循环，另一种情况我们继续执行循环。C代码就要简单的多。<u>如果锁没有被持有，那么锁对象的locked字段会是0，如果locked字段等于0，我们调用test-and-set将1写入locked字段，并且返回locked字段之前的数值0。如果返回0，那么意味着没有人持有锁，循环结束。如果locked字段之前是1，那么这里的流程是，先将之前的1读出，然后写入一个新的1，但是这不会改变任何数据，因为locked之前已经是1了。之后`__sync_lock_test_and_set`会返回1，表明锁之前已经被人持有了，这样的话，判断语句不成立，程序会持续循环（spin），直到锁的locked字段被设置回0</u>。

![img](https://pic4.zhimg.com/80/v2-b6e9c4a78446b77c0514495e5399e51b_1440w.webp)

​	接下来我们看一下release的实现，首先看一下`kernel.asm`中的指令。

![img](https://pic2.zhimg.com/80/v2-82c9113767e7d26bcccd49e7ecca8cb5_1440w.webp)

​	可以看出release也使用了atomic swap操作，将0写入到了s1。下面是对应的C代码，它基本确保了将`lk->locked`中写入0是一个原子操作。

![img](https://pic4.zhimg.com/80/v2-b56cf7241672ed6013b3c14723383147_1440w.webp)

---

​	关于这里lock的实现，有3个细节需要注意：

1. 为什么release函数中不直接使用一个store指令将锁的locked字段写为0？

   **可能有两个处理器或者两个CPU同时在向locked字段写入数据。这里的问题是，对于很多人包括我自己来说，经常会认为一个store指令是一个原子操作，但实际并不总是这样，这取决于具体的实现**。<u>例如，对于CPU内的缓存，每一个cache line的大小可能大于一个整数，那么store指令实际的过程将会是：首先会加载cache line，之后再更新cache line。所以对于store指令来说，里面包含了两个微指令。这样的话就有可能得到错误的结果。所以为了避免理解硬件实现的所有细节，例如整数操作不是原子的，或者向一个64bit的内存值写数据是不是原子的，我们直接使用一个RISC-V提供的确保原子性的指令来将locked字段写为0</u>。

   amoswap并不是唯一的原子指令，下图是RISC-V的手册，它列出了所有的原子指令。

   ![img](https://pic2.zhimg.com/80/v2-fb5cbc22524fb4392c91c3cce982c985_1440w.webp)

2. acquire函数实现的开头，为什么要先关闭中断？

   回到`uart.c`中。我们先来假设acquire在一开始并没有关闭中断。在uartputc函数中，首先会acquire锁，如果不关闭中断会发生什么呢？

   ![img](https://pic4.zhimg.com/80/v2-99f9f940a1dcbd19622886c17279034b_1440w.webp)

   <u>uartputc函数会acquire锁，UART本质上就是传输字符，当UART完成了字符传输它会做什么？是的，它会产生一个中断之后会运行uartintr函数，在uartintr函数中，会获取同一把锁，但是这把锁正在被uartputc持有。如果这里只有一个CPU的话，那这里就是**死锁**</u>。中断处理程序uartintr函数会一直等待锁释放，但是CPU不出让给uartputc执行的话锁又不会释放。在XV6中，这样的场景会触发panic，因为同一个CPU会再次尝试acquire同一个锁。

   所以spinlock需要处理两类并发，一类是**不同CPU之间的并发**，一类是**相同CPU上中断和普通程序之间的并发**。**<u>针对后一种情况，我们需要在acquire中关闭中断。中断会在release的结束位置再次打开，因为在这个位置才能再次安全的接收中断</u>**。

3. memory ordering，以及指令重排序问题

   假设我们先通过将locked字段设置为1来获取锁，之后对x加1，最后再将locked字段设置0来释放锁。下面将会是在CPU上执行的指令流：

   ![img](https://pic1.zhimg.com/80/v2-52d738002e1345d6b088517289166720_1440w.webp)

   但是编译器或者处理器可能会重排指令以获得更好的性能。对于上面的**串行指令流**，如果将`x<-x+1`移到`locked<-0`之后可以吗？这会改变指令流的正确性吗？并不会，因为x和锁完全相互独立，它们之间没有任何关联。如果他们还是按照串行的方式执行，`x<-x+1`移到锁之外也没有问题。所以在一个**串行执行**的场景下是没有问题的。**实际中，处理器在执行指令时，实际指令的执行顺序可能会改变。编译器也会做类似的事情，编译器可能会在不改变执行结果的前提下，优化掉一些代码路径并进而改变指令的顺序**。

   <u>但是对于**并发执行**，很明显这将会是一个灾难。如果我们将critical section与加锁解锁放在不同的CPU执行，将会得到完全错误的结果</u>。**<u>所以指令重新排序在并发场景是错误的。为了禁止，或者说为了告诉编译器和硬件不要这样做，我们需要使用memory fence或者叫做synchronize指令，来确定指令的移动范围。对于synchronize指令，任何在它之前的load/store指令，都不能移动到它之后。锁的acquire和release函数都包含了synchronize指令</u>**。

   这样前面的例子中，`x<-x+1`就不会被移到特定的memory synchronization点之外。我们也就不会有memory ordering带来的问题。这就是为什么在acquire和release中都有`__sync_synchronize`函数的调用。

   ![img](https://pic4.zhimg.com/80/v2-b56cf7241672ed6013b3c14723383147_1440w.webp)

---

问题：有没有可能在锁acquire之前的一条指令被移到锁release之后？或者说这里会有一个界限不允许这么做？

回答：在这里的例子中，acquire和release都有自己的界限（注，也就是`__sync_synchronize`函数的调用点）。所以发生在锁acquire之前的指令不会被移到acquire的`__sync_synchronize`函数调用之后，这是一个界限。在锁的release函数中有另一个界限。所以在第一个界限之前的指令会一直在这个界限之前，在两个界限之间的指令会保持在两个界限之间，在第二个界限之后的指令会保持在第二个界限之后。

## 10.10 锁-小结

+ 锁保证正确性，但同时降低性能

  并发运行的代码通过锁保证正确性，但同时锁限制了并发性，在临界区的代码其实是多个CPU串行运行。

+ 锁增加编程的复杂度

  + 除非必要，不然不加锁。程序如果需要并发执行，那么一般需要锁。
  + 如果不需要多个进程之间共享数据，那就不会有race condition，也就不需要锁。
  + 对于共享的数据结构，可以先从coarse-grained lock开始，然后基于测试结果，向fine-grained lock演进。

+ 使用race detector来找到race condition，如果你将锁的acquire和release放置于错误的位置，那么就算使用了锁还是会有race。

​	最后的课程还会介绍lock free program，并看一下如何在内核中实现它。

---

问题：fence指令不是没必要吗，因为amoswap指令可以保证acquire release顺序？

回答：sync instruction同步指令对编译器和硬件都适用。

问题：如何只对编译器禁止重排序？

回答：编译器知道正在编译的是什么体系结构的，所以我们知道它什么时候需要确保合适的fence，无论是在哪种体系结构上或哪种内存一致性模型上运行。这可以引申一些更复杂的讨论。每个硬件都有一个memory model内存模型，编译器决定在特定架构的内存模型上可做什么和不可做什么。

问题：我的问题是fence看似没必要。如果你调用amoswap，比如`.w.rl`，并在其中加入sync。但后续还有编译器重排序，内存排序和无序排序。机器也是用fence指令，fence指令只在执行`.rl`下是不需要的。但它似乎不会检测到这一点，那该怎么做？所以编译器最终会强制排序。但是你已经用指令覆盖了它。

回答：有一些复杂的需求实现，比如RISC-V专门的acquire和release实现，它们可能做比我们想的更复杂的事。是用fence指令时一个coarse grain的实现。RISC-V内存模型很复杂，如果你感兴趣，可以看看操作手册中非特权指令的部分，里面有整整一章内容讲memory ordering，以及对应该做什么，以及特定情况下编译器应该有的行为。

追问：所以意思是编译器会注意到这个事实，而我们只是把汇编指令放在这里，而它本身不会对任何内存访问进行重新排序？

回答：synchronize是一个库函数，它可以通过不同方式实现，这是其中一个特殊的实现。并且库函数由编译器提供。

追问：但是编译器是否可以进行优化，让它自己移动loads和stores指令？

回答：编译器能做到。

追问：那为什么不在是用fence时，阻止这种情况发生呢？

回答：也许，synchronized指令同时告知编译器和硬件，但编译器实际上可以以不同方式实现`__sync_synchronize`。它知道它不能移动东西，但它没有RISC-V上的issue和fence指令。它知道它以一种特殊的方式在RISC-V上运行。

问题：如果多个线程而仅有一个CPU核，那它处理方式和多核处理器是否一致？

回答：差不多吧，如果你有多个线程，但是只有一个CPU，那么你还是会想要特定内核代码能够原子执行。所以你还是需要有critical section的概念。你或许不需要锁，但是你还是需要能够对特定的代码打开或者关闭中断。**如果你查看一些操作系统的内核代码，通常它们都没有锁的acquire，因为它们假定自己都运行在单个处理器上，但它们确实有类似锁的东西，即基本上都进行开关中断的操作**。

# Lecture11 线程切换Thread Switching

> [Linux多线程(线程的创建，等待，终止，分离) - 卖寂寞的小男孩 - 博客园 (cnblogs.com)](https://www.cnblogs.com/lonely-little-boy/p/16726452.html)

## 11.1 线程概述

> [11.1 线程（Thread）概述 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346701754) <= 部分文字摘自该博文
>
> [事件驱动编程_百度百科 (baidu.com)](https://baike.baidu.com/item/事件驱动编程/6417369?fr=aladdin)
>
> [状态机_百度百科 (baidu.com)](https://baike.baidu.com/item/状态机/6548513?fr=aladdin)

​	为什么计算机需要运行多线程，主要可归结以下原因：

+ 人们希望计算机能同时执行多个任务，即计算机可能需要支持时分复用(time sharing)

  例如MIT的公共计算机系统Athena允许多个用户同时登陆一台计算机，并运行各自的进程。甚至在一个单用户的计算机或者在你的iphone上，你会运行多个进程，并期望计算机完成所有的任务而不仅仅只是一个任务。

+ 一定程度上减少代码实现复杂度

  比如在第一个lab中prime number部分，通过多个进程可以更简单，方便，优雅地组织代码。

+ 在多核CPU上并行计算，获得更快的处理速度

  常见的方式是将程序进行拆分，并通过线程在不同的CPU核上运行程序的不同部分。如果你足够幸运的话，你可以将你的程序拆分并在4个CPU核上通过4个线程运行你的程序，同时你也可以获取4倍的程序运行速度。你可以认为XV6就是一个多CPU并行运算的程序。

---

​	虽然人们对于线程有很多不同的定义，在这里，我们认为**线程就是单个串行执行代码的单元，它只占用一个CPU并且以普通的方式一个接一个的执行指令**。

​	为了切换线程时，仍然能恢复以前线程执行的工作，我们需要为线程保存一些状态。线程的状态包括：

+ 程序计数器（Program Counter），即当前线程执行的指令的位置。重中之重
+ 寄存器，保存线程当时使用中的变量值
+ 调用栈Stack，通常每个线程维护独立的Stack，其记录函数调用情况

---

​	操作系统需要管理多个线程的运行，通常多线程运行的策略有：

1. multi-core：在多核处理器上的多个CPU核上运行线程，一个CPU核同一时间能运行一个线程。
2. switch：在一个CPU核中切换多个线程执行。

​	本节主要讲解第二种策略。与大多数操作系统一样，XV6同时支持两种策略，即每个CPU核都能运行单个线程，并且多个线程能够在一个CPU核上切换。

----

​	**不同线程系统之间的一个主要的区别就是，线程之间是否会共享内存**。一种可能是你有一个地址空间，多个线程都在这一个地址空间内运行，并且它们可以看到彼此的更新。比如说共享一个地址空间的线程修改了一个变量，共享地址空间的另一个线程可以看到变量的修改。所以当多个线程运行在一个共享地址空间时，我们需要用到上节课讲到的锁。

​		**XV6内核共享了内存，并且XV6支持内核线程的概念，对于每个用户进程都有一个内核线程来执行来自用户进程的系统调用。所有的内核线程都共享了内核内存，所以XV6的内核线程的确会共享内存**。

​	另一方面，<u>XV6还有另外一种线程。每一个用户进程都有独立的内存地址空间，并且包含了一个线程，这个线程控制了用户进程代码指令的执行。**所以XV6中的用户线程之间没有共享内存，你可以有多个用户进程，但是每个用户进程都是拥有一个线程的独立地址空间**。XV6中的进程不会共享内存</u>。

​	**在一些其他更加复杂的系统中，例如Linux，允许在一个用户进程中包含多个线程，进程中的多个线程共享进程的地址空间。当你想要实现一个运行在多个CPU核上的用户进程时，你就可以在用户进程中创建多个线程。Linux中也用到了很多我们今天会介绍的技术，但是在Linux中跟踪每个进程的多个线程比XV6中每个进程只有一个线程要复杂的多**。

---

​	还有一些其他的方式可以支持在一台计算机上交织的运行多个任务，我们不会讨论它们，但是如果你感兴趣的话，你可以去搜索<u>**事件驱动编程(event-driven programming)**或者**状态机(state machine)**，这些是在一台计算机上不使用线程但又能运行多个任务的技术。在所有的支持多任务的方法中，线程技术并不是非常有效的方法，但是线程通常是最方便，对程序员最友好的，并且可以用来支持大量不同任务的方法</u>。

## 11.2 实现多线程系统的难点

> [11.2 XV6线程调度 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346702277) <= 部分文字摘自该博文

+ 如何实现多个线程之间的切换。这里停止一个线程的运行并启动另一个线程的过程通常被称为**线程调度(Scheduling)**。

  XV6为每个CPU核都创建了一个线程调度器scheduler。

+ 如何保存和恢复线程状态。

  切换到另一个线程前，需要决定上一个线程哪些状态需要保存，怎么保存（保存到哪）。

+ **如何处理运算密集型线程(compute bound threads)**。

  对于线程切换，很多直观的实现是由线程自己自愿的保存自己的状态，再让其他的线程运行。但是如果我们有一些程序正在执行一些可能要花费数小时的长时间计算任务，这样的线程并不能自愿的出让CPU给其他的线程运行。所以这里需要能从长时间运行的运算密集型线程撤回对于CPU的控制，将其放置于一边，稍后再运行它。

---

​	这里先针对第3个难点，即对"如何处理运算密集型线程(compute bound threads)"进行讲解。

​	其实关于这个问题的处理，之前的章节提到过了，即利用**定时器中断(timer interrupts)**。

​	**在每个CPU核上都存在一个定时产生中断的硬件设备**。XV6与其他所有的操作系统一样，将这个中断传输到了内核中。所以即使我们正在用户空间计算π的前100万位，定时器中断仍然能在例如每隔10ms的某个时间触发，并将程序运行的控制权从用户空间代码切换到内核中的中断处理程序（注，因为中断处理程序优先级更高）。哪怕这些用户空间进程并不配合工作（注，也就是用户空间进程一直占用CPU），内核也可以从用户空间进程获取CPU控制权。

​	**位于内核的定时器中断处理程序，会自愿的将CPU出让（yield）给线程调度器，并告诉线程调度器说，你可以让一些其他的线程运行了。这里的出让其实也是一种线程切换，它会保存当前线程的状态，并在稍后恢复**。

​	在之前的课程中，你们已经了解过了中断处理的流程。这里的基本流程是，定时器中断将CPU控制权给到内核，内核再自愿的出让CPU。

​	这样的处理流程被称为**抢占式调度(pre-emptive scheduling)**。<u>pre-emptive的意思是，即使用户代码本身没有出让CPU，定时器中断仍然会将CPU的控制权拿走，并出让给线程调度器</u>。与之相反的是**自愿调度(voluntary scheduling)**。

​	**<u>实际上，在XV6和其他的操作系统中，线程调度是这么实现的：定时器中断会强制的将CPU控制权从用户进程给到内核，这里是pre-emptive scheduling，之后内核会代表用户进程（注，实际是内核中用户进程对应的内核线程会代表用户进程出让CPU），使用voluntary scheduling</u>**。

---

​	在执行线程调度时，操作系统需要区分几种不同状态的线程：

+ 当前在CPU上运行的线程
+ 一旦CPU有空闲时间就想要运行在CPU上的线程
+ 不想运行在CPU上的线程，因为这些线程可能在等待I/O或者其他事件

​	这里不同的线程是由状态(state)区分，但是实际上线程的完整状态会要复杂的多（注，线程的完整状态包含了程序计数器，寄存器，栈等等）。下面是我们将会看到的一些线程状态：

+ RUNNING，线程当前正在某个CPU上运行
+ RUNABLE，线程还没有在某个CPU上运行，但是一旦有空闲的CPU就可以运行
+ SLEEPING，线程可能在等待一些I/O事件，在I/O事件发生了之后运行。这节课我们不会介绍，下节课会重点介绍。

​	今天这节课，我们主要关注RUNNING和RUNABLE这两类线程。**<u>前面介绍的定时器中断或者说pre-emptive scheduling，实际上就是将一个RUNNING线程转换成一个RUNABLE线程</u>。通过出让CPU，pre-emptive scheduling将一个正在运行的线程转换成了一个当前不在运行但随时可以再运行的线程**。因为当定时器中断触发时，这个线程还在好好的运行着。

​	**<u>对于RUNNING状态下的线程，它的程序计数器和寄存器位于正在运行它的CPU硬件中。而RUNABLE线程，因为并没有CPU与之关联，所以对于每一个RUNABLE线程，当我们将它从RUNNING转变成RUNABLE时，我们需要将它还在RUNNING时位于CPU的状态拷贝到内存中的某个位置，注意这里不是从内存中的某处进行拷贝，而是从CPU中的寄存器拷贝。我们需要拷贝的信息就是程序计数器（Program Counter）和寄存器</u>**。

​	当线程调度器决定要运行一个RUNABLE线程时，这里涉及了很多步骤，但是其中一步是<u>将之前保存的程序计数器和寄存器拷贝回调度器对应的CPU中</u>。	

## 11.3 XV6线程切换理论

> [11.3 XV6线程切换（一） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346702653)  <= 图片出自该博文，部分文字摘自该博文
>
> [线程调度器_百度百科 (baidu.com)](https://baike.baidu.com/item/线程调度器/2154102?fr=aladdin)

+ 用户空间中每个运行的用户进程有自己的程序计数器(PC)、用户程序栈(user stack)、寄存器状态值。
+ **当用户程序运行时，实际上是用户进程中一个用户线程在运行**。
+ 程序中执行了一个系统调用或响应中断而切换到内核中，那么**当前用户空间的状态会被保存到trapframe中**，之后CPU被切换到内核栈上运行。之后内核会运行一段时间处理系统调用或者执行中断处理程序。在处理完成之后，如果需要返回到用户空间，trapframe中保存的用户进程状态会被恢复。
+ 当内核决定从一个用户进程切换到另一个用户进程时，实际上会先从当前用户进程对应的内核线程切换到另一个用户进程对应的内核线程，最后再用另一个内核线程切换到另一个用户进程。

​	举例，XV6系统下从CC程序的内核线程切换到LS程序的内核线程时，大致经过以下几个步骤：

1. XV6将内核寄存器保存在CC内核线程的上下文(context)中。
2. 为了切换到RUNABLE的LS程序，XV6会恢复LS程序的内核线程的context对象，即恢复LS内核线程的寄存器。
3. 。之后LS会继续在它的内核线程栈上，完成系统调用。
4. LS内核线程通过系统调用流程，从trapframe中恢复用户状态，内核返回到用户空间的LS程序中，继续执行LS用户程序。

​	上面这里可能漏了很多细节描述，但**重点是：XV6中，我们从不直接进行用户进程的切换**。

​	在XV6中，从一个用户进程切换到另一个用户进程，需要大致经过以下步骤：

1. 保存当前用户进程的状态
2. 切换到运行当前用户进程对应的内核线程
3. 切换到另一个用户进程的内核线程，暂停当前用户进程的内核线程
4. 恢复另一个用户进程的用户寄存器（用户进程状态）

​	**真正实现比这曲折些，但用户进程之间的切换总是间接的，需要通过内核线程切换来辅助**。线程切换，最终效果就是从一个用户进程切换到了另一个用户进程。

![img](https://pic2.zhimg.com/80/v2-cab2275b2d3176c92f29a4768dd614f9_1440w.webp)

## 11.4 XV6线程调度器Scheduler

> [11.4 XV6线程切换（二） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346703833) <= 图片出处，部分文字摘自该博文

+ 每个CPU核有属于自己的调度器线程，在操作系统启动的时候就初始化了

+ 用户进程P1（用户空间）=> 用户进程P1对应内核线程 => **当前CPU的调度器线程** => 另一个进程P2的内核线程 => 另一个进程P2（用户空间）

---

![img](https://pic1.zhimg.com/80/v2-f3cf22a2384e64bc270777b3c70ae92c_1440w.webp)

​	这里假设CPU0有一个RUNNING的用户进程P1，此时模拟CPU0从用户进程P1切换到用户进程P2的流程：

1. 假设这里发生定时器中断，于是强制CPU从P1用户空间切换到内核，而trampoline代码段将用户寄存器保存到用户空间的trapframe中。
2. 之后在内核中运行usertrap，来实际执行相应的中断处理程序。这时，CPU正在进程P1的内核线程和内核栈上，执行内核中普通的C代码；
3. 假设进程P1对应的内核线程决定它想出让CPU，它会做很多工作，这个我们稍后会看，但是最后它会调用名为switch的函数，这是整个线程切换的核心函数之一；
4. <u>switch函数会保存用户进程P1对应内核线程的寄存器至context对象</u>。**所以目前为止有两类寄存器：用户寄存器存在trapframe中，内核线程的寄存器存在context中**。

​	**实际上switch函数并不是直接从一个内核线程切换到另一个内核线程**。<u>XV6中，一个CPU上运行的内核线程可以直接切换到的是这个**CPU对应的调度器线程**</u>。所以如果我们运行在CPU0，switch函数会恢复之前为CPU0的调度器线程保存的寄存器和stack pointer，之后就在调度器线程的context下执行schedulder函数。

​	在schedulder函数中会做一些清理工作，例如将进程P1设置成RUNABLE状态。之后再通过进程表单找到下一个RUNABLE进程。假设找到的下一个进程是P2（虽然也有可能找到的还是P1），**schedulder函数会再次调用switch函数**，完成下面步骤：	

1. **先保存自己的寄存器到调度器线程的context对象**
2. 找到进程P2之前保存的context，恢复其中的寄存器
3. 因为进程P2在进入RUNABLE状态之前，如刚刚介绍的进程P1一样，必然也调用了switch函数。所以之前的switch函数会被恢复，并返回到进程P2所在的系统调用或者中断处理程序中（注，因为P2进程之前调用switch函数必然在系统调用或者中断处理程序中）。
4. 不论是系统调用也好中断处理程序也好，在从用户空间进入到内核空间时会保存用户寄存器到trapframe对象。所以当内核程序执行完成之后，trapframe中的用户寄存器会被恢复。
5. 最后用户进程P2就恢复运行了。

---

​	**每一个CPU都有一个完全不同的调度器线程。调度器线程也是一种内核线程，它也有自己的context对象。任何运行在CPU1上的进程，当它决定出让CPU，它都会切换到CPU1对应的调度器线程，并由调度器线程切换到下一个进程**。

​	这里有一个术语需要解释一下。<u>当人们在说context switching，他们通常说的是从一个线程切换到另一个线程，因为在切换的过程中需要先保存前一个线程的寄存器，然后再恢复之前保存的后一个线程的寄存器，这些寄存器都是保存在context对象中</u>。在有些时候，context switching也指从一个用户进程切换到另一个用户进程的完整过程。偶尔你也会看到context switching是指从用户空间和内核空间之间的切换。**对于我们这节课来说，context switching主要是指一个内核线程和调度器线程之间的切换**。

​	这里有一些有用的信息可以记住。每一个CPU核在一个时间只会做一件事情，每个CPU核在一个时间只会运行一个线程，它要么是运行用户进程的线程，要么是运行内核线程，要么是运行这个CPU核对应的调度器线程。所以在任何一个时间点，CPU核并没有做多件事情，而是只做一件事情。线程的切换创造了多个线程同时运行在一个CPU上的假象。类似的每一个线程要么是只运行在一个CPU核上，要么它的状态被保存在context中。线程永远不会运行在多个CPU核上，线程要么运行在一个CPU核上，要么就没有运行。

​	**在XV6的代码中，context对象总是由switch函数产生，所以context总是保存了内核线程在执行switch函数时的状态。当我们在恢复一个内核线程时，对于刚恢复的线程所做的第一件事情就是从之前的switch函数中返回**（注，有点抽象，后面有代码分析）。

---

**问题：context保存在哪？**

回答：对于线程切换(thread switch)而言，每一个用户进程有一个对应的内核线程，它的context对象保存在用户进程的`proc->context`结构体中。**每一个调度器线程，它也有自己的context对象，但是它没有关联的进程，所以调度器线程的context对象保存在cpu核的结构体中。在内核中，有一个cpu结构体的数组，每个cpu结构体对应一个CPU核，每个结构体中都有一个context字段**。

**问题：为什么不直接将context保存到当前进程的trapframe中？**

**回答：context可以保存在trapframe中，因为每一个进程都只有一个内核线程对应的一组寄存器，我们可以将这些寄存器保存在任何一个与进程一一对应的数据结构中。对于每个进程来说，有一个proc结构体，有一个trapframe结构体，所以我们可以将context保存于trapframe中。但是或许出于简化代码或者让代码更清晰的目的，trapframe还是只包含进入和离开内核时的数据。而context结构体中包含的是在内核线程和调度器线程之间切换时，需要保存和恢复的数据**。

问题：出让CPU是由用户发起的还是由内核发起的？

回答：对于XV6来说，并不会直接让用户线程出让(yield)CPU或者完成线程切换(switching)，而是由内核在合适的时间点做决定。有的时候你可以猜到特定的系统调用会导致出让CPU，例如一个用户进程读取pipe，而它知道pipe中并不能读到任何数据，这时你可以预测读取会被阻塞，而内核在等待数据的过程中会运行其他的进程。<u>内核会在两个场景下出让CPU。当定时器中断触发了，内核总是会让当前进程出让CPU，因为我们需要在定时器中断间隔的时间点上交织执行所有想要运行的进程。另一种场景就是任何时候一个进程调用了系统调用并等待I/O，例如等待你敲入下一个按键，在你还没有按下按键时，等待I/O的机制会触发出让CPU</u>。

问题：用户进程调用sleep函数是不是会调用某个系统调用，然后将用户进程的信息保存在trapframe，然后触发进程切换，这时就不是定时器中断决定，而是用户进程自己决定了吧？

回答：如果进程执行了read系统调用，然后进入到了内核中。而read系统调用要求进程等待磁盘，这时系统调用代码会调用sleep，而sleep最后会调用switch函数。switch函数会保存内核线程的寄存器到进程的context中，然后切换到对应CPU的调度器线程，再让其他的线程运行。这样在当前线程等待磁盘读取结束时，其他线程还能运行。所以，这里的流程除了没有定时器中断，其他都一样，只是这里是因为一个系统调用需要等待I/O。

**问题：每一个CPU的调度器线程有自己的栈吗？**

**回答：是的，每一个调度器线程都有自己独立的栈。实际上调度器线程的所有内容，包括栈和context，与用户进程不一样，都是在系统启动时就设置好了。如果你查看XV6的start.s文件，你就可以看到为每个CPU核设置好调度器线程**。

**问题：我们这里一直在说线程，但是从我看来XV6的实现中，一个进程就只有一个线程，有没有可能一个进程有多个线程？**

回答：我们这里的用词的确有点让人混淆。在XV6中，一个进程要么在用户空间执行指令，要么是在内核空间执行指令，要么它的状态被保存在context和trapframe中，并且没有执行任何指令。这里该怎么称呼它呢？你可以根据自己的喜好来称呼它，对于我来说，**每个进程有两个线程，一个用户空间线程，一个内核空间线程，并且存在限制使得一个进程要么运行在用户空间线程，要么为了执行系统调用或者响应中断而运行在内核空间线程 ，但是永远也不会两者同时运行**。

## 11.5 XV6进程切换示例程序

> [11.5 XV6进程切换示例程序 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346708655) <= 图片出处，部分文字摘自该博文

​	先看看`proc.h`中代表进程的proc结构体。

```c
enum procstate { UNUSED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };

// Per-process state
struct proc {
  struct spinlock lock;
  
  // p->lock must be held when using these:
  enum procstate state;					// Proces state
  struct proc *parent;					// Parent process
  void *chan;										// If non-zero, sleeping on chan
  int killed;										// If non-zero,	have been killed
  int xstate;										// Exit status to be returned to parent's wait
  int pid;											// Process ID
  
  // these are private to the process, so p->lock need not be held.
  uint64 kstack;								// Virtual address of kernel stack
  uint64 sz;										// Size of process memory (bytes)
  pagetable_t pagetable; 				// User page table
  struct trapframe *trapframe;	// data page for trampoline.S
  struct context context;				// swtch() here to run process
  struct file *ofile[NOFILE];		// Open files
  struct inode *cwd;						// Current directory
  char name[16];								// Process name (debugging)
}
```

+ 首先是保存了用户空间线程寄存器的trapframe字段
+ **其次是保存了内核线程寄存器的context字段**
+ **还有保存了当前进程的内核栈的kstack字段，这是进程在内核中执行时保存函数调用的位置**
+ state字段保存了当前进程状态，要么是RUNNING，要么是RUNABLE，要么是SLEEPING等等
+ lock字段保护了很多数据，目前来说至少保护了对于state字段的更新。举个例子，因为有锁的保护，两个CPU的调度器线程不会同时拉取同一个RUNABLE进程并运行它

​	接下来看一个简单的示例C代码`spin.c`，演示进程切换。

![img](https://pic4.zhimg.com/80/v2-5329d69bac38c40c6fa358b859aec5c7_1440w.webp)

​	这个程序中会创建两个进程，两个进程会一直运行。代码首先通过fork创建了一个子进程，然后两个进程都会进入一个死循环，并每隔一段时间生成一个输出表明程序还在运行。但是它们都不会很频繁的打印输出（注，每隔1000000次循环才打印一个输出），并且它们也不会主动出让CPU（注，因为每个进程都执行的是没有sleep的死循环）。所以我们这里有了两个运算密集型进程，并且因为我们接下来启动的XV6只有一个CPU核，它们都运行在同一个CPU上。为了让这两个进程都能运行，有必要让两个进程之间能相互切换。

![img](https://pic2.zhimg.com/80/v2-a8aa01e1ba851e727c241a6ebaf3128d_1440w.webp)

​	可以看到一直有字符在输出，一个进程在输出“/”，另一个进程在输出"\"。从输出看，虽然现在XV6只有一个CPU核，但是每隔一会，XV6就在两个进程之间切换。“/”输出了一会之后，**定时器中断**将CPU切换到另一个进程运行然后又输出“\”一会。所以在这里我们可以看到定时器中断在起作用。

​	接下来，我在`trap.c`的devintr函数中的207行设置一个断点，这一行会识别出当前是在响应定时器中断。

![img](https://pic3.zhimg.com/80/v2-0303a5608f563ece7b4f972a860a857a_1440w.webp)

![img](https://pic1.zhimg.com/80/v2-9ffdd45219c0def3b1f64fa612edb338_1440w.webp)

​	之后在gdb中continue。立刻会停在中断的位置，因为定时器中断还是挺频繁的。现在我们可以确认我们在usertrap函数中，并且usertrap函数通过调用devintr函数来处理这里的中断（注，从下图的栈输出可以看出）。

![img](https://pic1.zhimg.com/80/v2-27369419f8893832248aa9ea599e0070_1440w.webp)

​	因为devintr函数处理定时器中断的代码基本没有内容，接下来我在gdb中输入finish来从devintr函数中返回到usertrap函数。当我们返回到usertrap函数时，虽然我们刚刚从devintr函数中返回，但是我们期望运行到下面的yield函数。所以我们期望devintr函数返回2。

![img](https://pic1.zhimg.com/80/v2-844fd949b258f9af5fb760f8b2749150_1440w.webp)

​	可以从gdb中看到devintr的确返回的是2。

![img](https://pic2.zhimg.com/80/v2-36141abfa4956a985d587f95a4f4e595_1440w.webp)

​	**在yield函数中，当前进程会出让CPU并让另一个进程运行**。这个我们稍后再看。现在让我们看一下当定时器中断发生的时候，用户空间进程正在执行什么内容。我在gdb中输入print p来打印名称为p的变量。变量p包含了当前进程的proc结构体。打印`p->name`来获取进程的名称，打印`p->pid`获取当前运行进程号，当前运行spin程序，对应进程号3。进程切换之后，我们预期进程ID会不一样。

![img](https://pic1.zhimg.com/80/v2-9e78e2055f16c02f40135e0535f2cc20_1440w.webp)

![img](https://pic4.zhimg.com/80/v2-455847e37b2ca26c0473e4c42d8393ef_1440w.webp)

​	我们还可以通过打印变量p的trapframe字段获取表示用户空间状态的32个寄存器，这些都是我们在Lec06中学过的内容。这里面最有意思的可能是trapframe中保存的用户程序计数器。

![img](https://pic1.zhimg.com/80/v2-1e53690cca7f5456419e07b4c0098f38_1440w.webp)

​	我们可以查看spin.asm文件来确定对应地址的指令。

![img](https://pic2.zhimg.com/80/v2-de0f060fb480ada8eee8339bfaed4449_1440w.webp)

​	可以看到定时器中断触发时，用户进程正在执行死循环的加1，这符合我们的预期。

----

**问题：怎么区分不同进程的内核线程？**

回答：每一个进程都有一个独立的内核线程。实际上有两件事情可以区分不同进程的内核线程，其中一件是，**每个进程都有不同的内核栈，它由proc结构体中的kstack字段所指向**；另一件就是，**任何内核代码都可以通过调用myproc函数来获取当前CPU正在运行的进程。内核线程可以通过调用这个函数知道自己属于哪个用户进程。myproc函数会使用tp寄存器来获取当前的CPU核的ID，并使用这个ID在一个保存了所有CPU上运行的进程的结构体数组中，找到对应的proc结构体**。这就是不同的内核线程区分自己的方法。

## 11.6 XV6的yield和sched函数

> [11.6 XV6线程切换 --- yield/sched函数 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346709314) <= 图文出处

​	回到devintr函数返回到usertrap函数中的位置。在gdb里面输入几次step走到yield函数的调用。**yield函数是整个线程切换的第一步**，下面是yield函数的内容：

![img](https://pic4.zhimg.com/80/v2-eeeffab386b42fe9f1a6afa152353bbf_1440w.webp)

​	**yield函数只做了几件事情，它首先获取了进程的锁**。实际上，在锁释放之前，进程的状态会变得不一致，例如，yield将要将进程的状态改为RUNABLE，表明进程并没有在运行，但是实际上这个进程还在运行，代码正在当前进程的内核线程中运行。<u>所以这里加锁的目的之一就是：即使我们将进程的状态改为了RUNABLE，其他的CPU核的调度器线程也不可能看到进程的状态为RUNABLE并尝试运行它。否则的话，进程就会在两个CPU核上运行了，而一个进程只有一个栈，这意味着两个CPU核在同一个栈上运行代码</u>（注，因为XV6中一个用户进程只有一个用户线程）。

​	**接下来yield函数中将进程的状态改为RUNABLE**。<u>这里的意思是，当前进程要出让CPU，并切换到调度器线程。当前进程的状态是RUNABLE意味着它还会再次运行，因为毕竟现在是一个定时器中断打断了当前正在运行的进程</u>。

​	之后**yield函数中调用了位于`proc.c`文件中的sched函数**。我们进入到sched函数中，

![img](https://pic3.zhimg.com/80/v2-46da31a9abb91035947de82dda068882_1440w.webp)

​	可以看出，**sched函数基本没有干任何事情，只是做了一些合理性检查，如果发现异常就panic，然后最后调用了一下switch**。为什么会有这么多检查？因为这里的XV6代码已经有很多年的历史了，这些代码经历过各种各样的bug，相应的这里就有各种各样的合理性检查。

## 11.7 XV6的switch函数

> [11.7 XV6线程切换 --- switch函数 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346709521) <= 图文出处

![img](https://pic3.zhimg.com/80/v2-46da31a9abb91035947de82dda068882_1440w.webp)

​	switch函数会将当前的内核线程的寄存器保存到`p->context`中。switch函数的另一个参数`c->context`，c表示当前CPU的结构体。CPU结构体中的context保存了当前CPU核的调度器线程的寄存器。所以**switch函数在保存完当前内核线程的内核寄存器之后，就会恢复当前CPU核的调度器线程的寄存器，并继续执行当前CPU核的调度器线程**。

​	接下来，我们快速的看一下我们将要切换到的context（注，也就是调度器线程的context）。因为我们只有一个CPU核，这里我在gdb中`print cpus[0].context`

![img](https://pic3.zhimg.com/80/v2-c9d061cb8568c45b580e459d6ffb90b2_1440w.webp)

​	这里看到的就是之前保存的当前CPU核的调度器线程的寄存器。在这些寄存器中，最有趣的就是ra（Return Address）寄存器，因为<u>ra寄存器保存的是当前函数的返回地址，所以调度器线程中的代码会返回到ra寄存器中的地址</u>。通过查看`kernel.asm`，我们可以知道这个地址的内容是什么。也可以在gdb中输入`x/i 0x80001f2e`进行查看。

![img](https://pic3.zhimg.com/80/v2-8e1bab66a5155372a8d5605a324bfd56_1440w.webp)

​	输出中包含了地址中的指令和指令所在的函数名。所以我们将要返回到scheduler函数中。

​	因为我们接下来要调用switch函数，让我们来看看switch函数的内容。switch函数位于`switch.s`文件中。

![img](https://pic4.zhimg.com/80/v2-31c443237da0f61f0daa2b939562301f_1440w.webp)

​	首先，ra寄存器被保存在了a0寄存器指向的地址。<u>a0寄存器对应了switch函数的第一个参数，从前面可以看出这是当前线程的context对象地址；a1寄存器对应了switch函数的第二个参数，从前面可以看出这是即将要切换到的调度器线程的context对象地址</u>。

​	**所以函数中上半部分是将当前的寄存器保存在当前线程对应的context对象中，函数的下半部分是将调度器线程的寄存器，也就是我们将要切换到的线程的寄存器恢复到CPU的寄存器中**。之后函数就返回了。所以调度器线程的ra寄存器的内容才显得有趣，因为它指向的是switch函数返回的地址，也就是scheduler函数。

​	这里有个有趣的问题，或许你们已经注意到了。switch函数的上半部分保存了ra，sp等等寄存器，但是并没有保存程序计数器pc（Program Counter），为什么会这样呢？学生回答：因为程序计数器不管怎样都会随着函数调用更新。是的，程序计数器并没有有效信息，我们现在知道我们在switch函数中执行，所以保存程序计数器并没有意义。但是我们关心的是我们是从哪调用进到switch函数的，因为当我们通过switch恢复执行当前线程并且从switch函数返回时，我们希望能够从调用点继续执行。ra寄存器保存了switch函数的调用点，所以这里保存的是ra寄存器。我们可以打印ra寄存器，如你们所预期的一样，它指向了sched函数。

![img](https://pic2.zhimg.com/80/v2-c4fbdc8c69f990ee1a58d86ad324b6cd_1440w.webp)

​	另一个问题是，为什么RISC-V中有32个寄存器，但是switch函数中只保存并恢复了14个寄存器？学生回答：因为switch是按照一个普通函数来调用的，对于有些寄存器，switch函数的调用者默认switch函数会做修改，所以调用者已经在自己的栈上保存了这些寄存器，当函数返回时，这些寄存器会自动恢复。所以switch函数里只需要保存Callee Saved Register就行。完全正确！<u>因为switch函数是从C代码调用的，所以我们知道Caller Saved Register会被C编译器保存在当前的栈上。Caller Saved Register大概有15-18个，而我们在switch函数中只需要处理C编译器不会保存，但是对于switch函数又有用的一些寄存器。所以在切换线程的时候，我们只需要保存Callee Saved Register</u>。

​	最后我想看的是sp（Stack Pointer）寄存器。从它的值很难看出它的意义是什么。它实际是当前进程的内核栈地址，它由虚拟内存系统映射在了一个高地址。

![img](https://pic4.zhimg.com/80/v2-f676a77646825298bd3f533bed6007bb_1440w.webp)

​	现在，我们保存了当前的寄存器，并从调度器线程的context对象恢复了寄存器，我直接跳到switch函数的最后，也就是ret指令的位置。

![img](https://pic4.zhimg.com/80/v2-ac18e7d8e03ee43507f3403eb2371503_1440w.webp)

​	在我们实际返回之前，我们再来打印一些有趣的寄存器。首先sp寄存器有了一个不同的值。sp寄存器的值现在在内存中的stack0区域中。这个区域实际上是在启动顺序中非常非常早的一个位置，start.s在这个区域创建了栈，这样才可以调用第一个C函数。所以调度器线程运行在CPU对应的bootstack上。

![img](https://pic4.zhimg.com/80/v2-ad4eb0d83d9a44801ac602ea5af60537_1440w.webp)

​	其次是ra寄存器，现在指向了scheduler函数，因为我们恢复了调度器线程的context对象中的内容。

![img](https://pic4.zhimg.com/80/v2-9b0c0e0f2f4ad254696a28c8ff07fa67_1440w.webp)

​	现在，我们其实已经在调度器线程中了，这里寄存器的值与上次打印的已经完全不一样了。<u>虽然我们还在switch函数中，但是现在我们实际上位于调度器线程调用的switch函数中。**调度器线程在启动过程中调用的也是switch函数**。接下来通过执行ret指令，我们就可以返回到调度器线程中</u>。

## 11.8 XV6的scheduler函数

> [11.8 XV6线程切换 --- scheduler函数 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346709842) <= 图文出处

![img](https://pic4.zhimg.com/80/v2-9d8a5a064e76f65795240df25ce5a8e7_1440w.webp)

​	现在我们正运行在CPU拥有的调度器线程中，并且我们正好在之前调用switch函数的返回状态。之前调度器线程调用switch是因为想要运行pid为3的进程，也就是刚刚被中断的spin程序。

​	<u>虽然pid为3的spin进程也调用了switch函数，但是那个switch并不是当前返回的这个switch。spin进程调用的switch函数还没有返回，而是保存在了pid为3的栈和context对象中。现在返回的是之前调度器线程对于switch函数的调用</u>。

​	在scheduler函数中，因为我们已经停止了spin进程的运行，所以我们需要抹去对于spin进程的记录。我们接下来将`c->proc`设置为0（`c->proc = 0;`）。因为我们现在并没有在这个CPU核上运行这个进程，为了不让任何人感到困惑，我们这里将CPU核运行的进程对象设置为0。

​	<u>之前在yield函数中获取了进程的锁，因为yield不想进程完全进入到Sleep状态之前，任何其他的CPU核的调度器线程看到这个进程并运行它</u>。而现在我们完成了从spin进程切换走，所以现在可以释放锁了。这就是`release(&p->lock)`的意义。<u>现在，我们仍然在scheduler函数中，但是其他的CPU核可以找到spin进程，并且因为spin进程是RUNABLE状态，其他的CPU可以运行它。这没有问题，因为我们已经完整的保存了spin进程的寄存器，并且我们不在spin进程的栈上运行程序，而是在当前CPU核的调度器线程栈上运行程序，所以其他的CPU核运行spin程序并没有问题</u>。但是因为启动QEMU时我们只指定了一个核，所以在我们现在的演示中并没有其他的CPU核来运行spin程序。

​	接下来我将简单介绍一下`p->lock`。从调度的角度来说，这里的锁完成了两件事情。

​	首先，<u>出让CPU涉及到很多步骤，我们需要将进程的状态从RUNNING改成RUNABLE，我们需要将进程的寄存器保存在context对象中，并且我们还需要停止使用当前进程的栈</u>。所以这里至少有三个步骤，而这三个步骤需要花费一些时间。所以锁的第一个工作就是在这三个步骤完成之前，阻止任何一个其他核的调度器线程看到当前进程。<u>锁这里确保了三个步骤的原子性</u>。从CPU核的角度来说，三个步骤要么全发生，要么全不发生。

​	第二，当我们开始要运行一个进程时，`p->lock`也有类似的保护功能。当我们要运行一个进程时，我们需要将进程的状态设置为RUNNING，我们需要将进程的context移到RISC-V的寄存器中。但是，如果在这个过程中，发生了中断，从中断的角度来说进程将会处于一个奇怪的状态。比如说进程的状态是RUNNING，但是又还没有将所有的寄存器从context对象拷贝到RISC-V寄存器中。所以，如果这时候有了一个定时器中断将会是个灾难，因为我们可能在寄存器完全恢复之前，从这个进程中切换走。而从这个进程切换走的过程中，将会保存不完整的RISC-V寄存器到进程的context对象中。所以我们希望启动一个进程的过程也具有原子性。在这种情况下，切换到一个进程的过程中，也需要获取进程的锁以确保其他的CPU核不能看到这个进程。同时在切换到进程的过程中，还需要关闭中断（前面"10.9 XV6的acquire和release的实现"章节可以知道acquire上来就先关闭了中断），这样可以避免定时器中断看到还在切换过程中的进程。（注，这就是为什么468行需要加锁的原因）

​	现在我们在scheduler函数的循环中，代码会检查所有的进程并找到一个来运行。现在我们知道还有另一个进程，因为我们之前fork了另一个spin进程。这里我跳过进程检查，直接在找到RUNABLE进程的位置设置一个断点。

![img](https://pic1.zhimg.com/80/v2-f3b414e83b956f9779d35a950becf488_1440w.webp)

​	<u>在代码的468行，获取了进程的锁，所以现在我们可以进行切换到进程的各种步骤。在代码的473行，进程的状态被设置成了RUNNING。代码的474行将找到的RUNABLE进程记录为当前CPU执行的进程。代码的475行，又调用了switch函数来保存调度器线程的寄存器，并恢复目标进程的寄存器（注，实际上恢复的是目标进程的内核线程）</u>。我们可以打印新的进程的名字来查看新的进程。

![img](https://pic1.zhimg.com/80/v2-57f2a048568eb034ef54e1ebf88dfa68_1440w.webp)

​	可以看到进程名还是spin，但是pid已经变成了4，而前一个进程的pid是3。我们还可以查看目标进程的context对象，其中ra寄存器的内容就是我们要切换到的目标线程的代码位置。虽然我们在代码475行调用的是switch函数，但是我们前面已经看过了switch函数会返回到即将恢复的ra寄存器地址，所以我们真正关心的就是ra指向的地址。

![img](https://pic2.zhimg.com/80/v2-6544f0f9244822768894a8169a20a6cd_1440w.webp)

![img](https://pic3.zhimg.com/80/v2-cd95abe0bbf712ce9818a7af3fda8832_1440w.webp)

​	通过打印这个地址的内容，可以看到switch函数会返回到sched函数中。这完全在意料之中，因为可以预期的是，将要切换到的进程之前是被定时器中断通过sched函数挂起的，并且之前在sched函数中又调用了switch函数。

​	<u>在switch函数的最开始，我们仍然在调度器线程中，但是这一次是从调度器线程切换到目标进程的内核线程</u>。所以从switch函数内部将会返回到目标进程的内核线程的sched函数，通过打印backtrace，我们可以看到，之前有一个usertrap的调用，这必然是之前因为定时器中断而出现的调用。之后在中断处理函数中还调用了yield和sched函数，正如我们之前看到的一样。但是，这里调用yield和sched函数是在pid为4的进程调用的，而不是我们刚刚看的pid为3的进程。

![img](https://pic3.zhimg.com/80/v2-feeeabe802ca4846c486af1c7ccbd9d2_1440w.webp)

​	这里有件事情需要注意，调度器线程调用了switch函数，但是我们从switch函数返回时，实际上是返回到了对于switch的另一个调用，而不是调度器线程中的调用。我们返回到的是pid为4的进程在很久之前对于switch的调用。这里可能会有点让人困惑，但是这就是线程切换的核心。

​	另一件需要注意的事情是，**switch函数是线程切换的核心，但是switch函数中只有保存寄存器，再加载寄存器的操作**。线程除了寄存器以外的还有很多其他状态，它有变量，堆中的数据等等，但是所有的这些数据都在内存中，并且会保持不变。我们没有改变线程的任何栈或者堆数据。所以<u>线**程切换的过程中，处理器中的寄存器是唯一的不稳定状态，且需要保存并恢复**。而所有其他在内存中的数据会保存在内存中不被改变，所以不用特意保存并恢复</u>。我们只是保存并恢复了处理器中的寄存器，因为我们想在新的线程中也使用相同的一组寄存器。

---

问题：如果不是因为定时器中断发生的切换，我们是不是可以期望ra寄存器指向其他位置，例如sleep函数？

回答：是的，我们之前看到了代码执行到这里会包含一些系统调用相关的函数。你基本上回答了自己的问题，如果我们因为定时器中断之外的原因而停止了执行当前的进程，switch会返回到一些系统调用的代码中，而不是我们这里看到sched函数。我记得sleep最后也调用了sched函数，虽然bracktrace可能看起来会不一样，但是还是会包含sched。所以我这里只介绍了一种进程间切换的方法，也就是因为定时器中断而发生切换。但是还有其他的可能会触发进程切换，例如等待I/O或者等待另一个进程向pipe写数据。

问题：看起来所有的CPU核要能完成线程切换都需要有一个定时器中断，那如果硬件定时器出现故障了怎么办？

回答：是的，总是需要有一个定时器中断。用户进程的pre-emptive scheduling能工作的原因是，用户进程运行时，中断总是打开的。XV6会确保返回到用户空间时，中断是打开的。这意味着当代码在用户空间执行时，定时器中断总是能发生。在内核中会更加复杂点，因为内核中偶尔会关闭中断，比如当获取锁的时候，中断会被关闭，只有当锁被释放之后中断才会重新打开，所以如果内核中有一些bug导致内核关闭中断之后再也没有打开中断，同时内核中的代码永远也不会释放CPU，那么定时器中断不会发生。但是因为XV6是我们写的，所以它总是会重新打开中断。XV6中的代码如果关闭了中断，它要么过会会重新打开中断，然后内核中定时器中断可以发生并且我们可以从这个内核线程切换走，要么代码会返回到用户空间。我们相信XV6中不会有关闭中断然后还死循环的代码。

追问：我的问题是，定时器中断是来自于某个硬件，如果硬件出现故障了呢？

回答：你的电脑坏了，你要买个新电脑了。这个问题是可能发生的，因为电脑中有上亿的晶体管，有的时候电脑会有问题，但是这超出了内核的管理范围了。所以我们假设计算机可以正常工作。有的时候软件会尝试弥补硬件的错误，比如通过网络传输packet，总是会带上checksum，这样如果某个网络设备故障导致某个bit反转了，可以通过checksum发现这个问题。**但是对于计算机内部的问题，人们倾向于不用软件来尝试弥补硬件的错误。**

问题：当一个线程结束执行了，比如说在用户空间通过exit系统调用结束线程，同时也会关闭进程的内核线程。那么线程结束之后和下一个定时器中断之间这段时间，CPU仍然会被这个线程占有吗？还是说我们在结束线程的时候会启动一个新的线程？

回答：exit系统调用会出让CPU。尽管我们这节课主要是基于定时器中断来讨论，但是实际上XV6切换线程的绝大部分场景都不是因为定时器中断，比如说一些系统调用在等待一些事件并决定让出CPU。exit系统调用会做各种操作然后调用yield函数来出让CPU，这里的出让并不依赖定时器中断。

问题：我不知道我们使用的RISC-V处理器是不是有一些其他的状态？但是我知道一些Intel的X86芯片有floating point unit state等其他的状态，我们需要处理这些状态吗？

回答：你的观点非常对。在一些其他处理器例如X86中，线程切换的细节略有不同，因为不同的处理器有不同的状态。所以我们这里介绍的代码非常依赖RISC-V。其他处理器的线程切换流程可能看起来会非常的不一样，比如说可能要保存floating point寄存器。我不知道RISC-V如何处理浮点数，但是XV6内核并没有使用浮点数，所以不必担心。但是是的，**线程切换与处理器非常相关**。

**问题：为什么switch函数要用汇编来实现，而不是C语言？**

**回答：C语言中很难与寄存器交互。可以肯定的是C语言中没有方法能更改sp、ra寄存器。所以在普通的C语言中很难完成寄存器的存储和加载，唯一的方法就是在C中嵌套汇编语言。所以我们也可以在C函数中内嵌switch中的指令，但是这跟我们直接定义一个汇编函数是一样的。或者说switch函数中的操作是在C语言的层级之下，所以并不能使用C语言。**

## 11.9 XV6第一次switch调用

> [11.9 XV6线程第一次调用switch函数 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346710168) <= 图文出处

问题：当调用switch函数的时候，实际上是从一个线程对于switch的调用切换到了另一个线程对于switch的调用。所以线程第一次调用switch函数时，需要伪造一个"另一个线程"对于switch的调用，是吧？因为也不能通过switch函数随机跳到其他代码去。

回答：是的。我们来看一下第一次调用switch时，"另一个"调用switch函数的线程的context对象。proc.c文件中的allocproc函数会被启动时的第一个进程和fork调用，allocproc会设置好新进程的context。

![img](https://pic3.zhimg.com/80/v2-beb3f4f282b535632b32f6dcd0e407be_1440w.webp)

​	实际上大部分寄存器的内容都无所谓。但是ra很重要，因为这是进程的第一个switch调用会返回的位置。同时因为进程需要有自己的栈，所以ra和sp都被设置了。**这里设置的forkret函数就是进程的第一次调用switch函数会切换到的“另一个”线程位置**。

问题：所以当switch函数返回时，CPU会执行forkret中的指令，就像forkret刚刚调用了switch函数并且返回了一样？

回答：是的，从switch返回就直接跳到了forkret的最开始位置。

追问：我们会在其他场合调用forkret吗？还是说它只会用在这？

回答：是的，它只会在启动进程的时候以这种奇怪的方式运行。下面是forkret函数的代码。

![img](https://pic2.zhimg.com/80/v2-c5720f05080904ce16c5f7584175fe91_1440w.webp)

​	从代码中看，它的工作其实就是释放调度器之前获取的锁。函数最后的usertrapret函数其实也是一个假的函数，它会使得程序表现的看起来像是从trap中返回，但是对应的trapframe其实也是假的，这样才能跳到用户的第一个指令中。

问题：与之前的context对象类似的是，对于trapframe也不用初始化任何寄存器，因为我们要去的是程序的最开始，所以不需要做任何假设，对吧？

回答：我认为程序计数器还是要被初始化为0的。

问题：在fortret函数中，`if(first)`是什么意思？

回答：文件系统需要被初始化，具体来说，需要从磁盘读取一些数据来确保文件系统的运行，比如说文件系统究竟有多大，各种各样的东西在文件系统的哪个位置，同时还需要有crash recovery log。完成任何文件系统的操作都需要等待磁盘操作结束，但是XV6只能在进程的context下执行文件系统操作，比如等待I/O。所以初始化文件系统需要等到我们有了一个进程才能进行。而这一步是在第一次调用forkret时完成的，所以在forkret中才有了`if(first)`判断。

![img](https://pic3.zhimg.com/80/v2-2ba143d56c4c6a6f8da7dc8f123263c6_1440w.webp)

​	因为fork拷贝的进程会同时拷贝父进程的程序计数器，所以我们唯一不是通过fork创建进程的场景就是创建第一个进程的时候。这时需要设置程序计数器为0。

---

问题：操作系统都带了线程的实现，如果想要在多个CPU上运行一个进程内的多个线程，那需要通过操作系统来处理而不是用户空间代码，是吧？那这里的线程切换是怎么工作的？是每个线程都与进程一样了吗？操作系统还会遍历所有存在的线程吗？比如说我们有8个核，每个CPU核都会在多个进程的更多个线程之间切换。同时我们也不想只在一个CPU核上切换一个进程的多个线程，是吧？

回答：**Linux是支持一个进程包含多个线程，Linux的实现比较复杂，或许最简单的解释方式是：几乎可以认为Linux中的每个线程都是一个完整的进程。Linux中，我们平常说一个进程中的多个线程，本质上是共享同一块内存的多个独立进程。所以Linux中一个进程的多个线程仍然是通过一个内存地址空间执行代码。如果你在一个进程创建了2个线程，那基本上是2个进程共享一个地址空间。之后，调度就与XV6是一致的，也就是针对每个进程进行调度。**

追问：用户可以指定将线程绑定在某个CPU上吗？操作系统如何确保一个进程的多个线程不会运行在同一个CPU核上？要不然就违背了多线程的初衷了。

回答：这里其实与XV6非常相似，假设有4个CPU核，Linux会找到4件事情运行在这4个核上。如果并没有太多正在运行的程序的话，或许会将一个进程的4个线程运行在4个核上。或者如果有100个用户登录在Athena机器上，内核会随机为每个CPU核找到一些事情做。<u>如果你想做一些精细的测试，有一些方法可以将线程绑定在CPU核上，但正常情况下人们不会这么做</u>。

追问：所以说Linux中一个进程中的多个线程会有相同的page table？

回答：是的，如果你在Linux上，你为一个进程创建了2个线程，我不确定它们是不是共享同一个的page table，还是说它们是不同的page table，但是内容是相同的。

追问：有没有原因说这里的page table要是分开的？

回答：我不知道Linux究竟用了哪种方法。

# Lecture12 Q&A for Labs

​	教师提到自己的COW fork(copy-on-write fork)的实现，经过测试，大概减少90%的字节复制量，猜测主要是因为instruction pages的复制量减少了，因为指令页从不修改，也就没必要复制。同时COW减少了RAM的使用，节省fork时间。（当然如果程序后续修改了 copy-on-write pages，那就不得不再对这些pages进行复制。）

