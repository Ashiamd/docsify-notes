# MIT6.S081网课学习笔记-03

> [MIT6.S081 操作系统工程中文翻译 - 知乎 (zhihu.com)](https://www.zhihu.com/column/c_1294282919087964160) => 知乎大佬视频中文笔记，很全
>
> [6.S081 / Fall 2020 (mit.edu)](https://pdos.csail.mit.edu/6.S081/2020/schedule.html) => 课表+实验超链接等信息

# Lecture19 虚拟机(Virtual Machines)

## 19.1 虚拟机概述(Virtual Machine)

> [19.1 Why Virtual Machine? - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/365569925) <= 图文出处
>
> [虚拟机管理器_百度百科 (baidu.com)](https://baike.baidu.com/item/虚拟机管理器/5421878?fr=aladdin)
>
> [虚拟机监视程序_百度百科 (baidu.com)](https://baike.baidu.com/item/虚拟机监视程序/20839241?fromtitle=VMM&fromid=7047240&fr=aladdin)
>
> 虚拟化系统下的I/O访问需要在客户操作系统、[VMM](https://baike.baidu.com/item/VMM?fromModule=lemma_inlink)、[设备驱动程序](https://baike.baidu.com/item/设备驱动程序?fromModule=lemma_inlink)、[I/O设备](https://baike.baidu.com/item/I%2FO设备?fromModule=lemma_inlink)共同参与下才能完成。所谓的虚拟设备就是由VMM创建的，提供给客户操作系统进行I/O访问的虚拟I/O设备。客户操作系统只能观察到属于它的虚拟I/O设备，客户操作系统的所有I/O访问都被发往它的虚拟I/O设备，然后VMM软件从虚拟I/O设备中获取客户操作系统的访问请求，继而完成真正的I/O访问。
>
> [hypervisor_百度百科 (baidu.com)](https://baike.baidu.com/item/hypervisor/3353492?fr=aladdin)
>
> **Hypervisor**，又称**虚拟机监视器**（英语：virtual machine monitor，缩写为 VMM），是用来建立与执行[虚拟机器](https://baike.baidu.com/item/虚拟机器?fromModule=lemma_inlink)的软件、固件或硬件。
>
> 被Hypervisor用来执行一个或多个虚拟机器的电脑称为主体机器（host machine），这些虚拟机器则称为客体机器（guest machine）。hypervisor提供虚拟的作业平台来执行客体操作系统（guest operating systems），负责管理其他客体操作系统的执行阶段；这些客体操作系统，共同分享虚拟化后的硬件[资源](https://baike.baidu.com/item/资源?fromModule=lemma_inlink)。
>
> Hypervisor——一种运行在基础物理服务器和[操作系统](https://baike.baidu.com/item/操作系统?fromModule=lemma_inlink)之间的[中间软件](https://baike.baidu.com/item/中间软件?fromModule=lemma_inlink)层，可允许多个操作系统和应用共享硬件。也可叫做VMM（ virtual machine monitor ），即[虚拟机](https://baike.baidu.com/item/虚拟机?fromModule=lemma_inlink)[监视器](https://baike.baidu.com/item/监视器?fromModule=lemma_inlink)。

今天讨论的话题是虚拟机。今天的内容包含三个部分:

- 第一个部分是Trap and Emulate，这部分会介绍如何在RISC-V或者QEMU上构建属于自己的Virtual Machine Monitor（注，有些场合也称为Hypervisor）。
- 第二部分会描述最近在硬件上对于虚拟化的支持。
- 最后是讨论一下今天的[论文](https://pdos.csail.mit.edu/6.828/2020/readings/belay-dune.pdf)，它使用了第二部分中硬件上的支持。

​	首先什么是虚拟机？你可以认为这是对于计算机的一种模拟，这种模拟足够能运行一个操作系统。QEMU可以认为是虚拟机的一个例子（注，QEMU应该是属于VMM/Hypervisor）。

​	**在架构的最底层，位于硬件之上存在一个Virtual Machine Monitor（VMM），它取代了标准的操作系统内核**。<u>VMM的工作是模拟多个计算机用来运行Guest操作系统。VMM往上一层，如果对比一个操作系统的架构应该是用户空间，但是现在是叫做Guest空间。所以在今天的架构图里面，上面是Guest空间，下面是Host空间（注，也就是上面运行Guest操作系统，下面运行VMM）</u>。

![img](https://pic3.zhimg.com/80/v2-37ca0ba3f131c846729423d338dd574e_1440w.webp)

​	<u>在Guest空间，会有一个或者多个Guest操作系统内核，或许其中一个是Linux kernel</u>。这里的Linux kernel会觉得自己就是个普通的内核，并在自己之上还运行一堆用户进程，例如VI，C Compiler。我们或许还有另一个Guest运行了Windows操作系统，同时也包含了Windows用户进程。所以，**在Host空间运行的是VMM，在Guest空间运行的是普通的操作系统**。

​	除此之外，**在Guest空间又可以分为Guest Supervisor Mode，也就是Guest操作系统内核运行的模式，和Guest User Mode**。

![img](https://pic4.zhimg.com/80/v2-bbbb7c7ffd678ff9758728873259ef43_1440w.webp)

​	**VMM的主要目的是提供对计算机的模拟**，这样你可以不做修改就启动普通的Linux，普通的Windows系统，并运行在虚拟机内，并且不用担心任何奇怪的事情发生。所以，<u>VMM必须要能够完全按照实际硬件的行为来模拟Guest Supervisor Mode和Guest User Mode，尽管实际上不可能完全一样，我们之后会讨论VMM对于这两种模式的模拟</u>。

​	*那么人们为什么会想要使用虚拟机呢？实际中有很多原因使得人们会在一个计算机上运行多个相互独立的操作系统。在一个大公司里面，你需要大量的服务器，例如DNS，Firewall等等，但是每个服务器并没有使用太多的资源，所以单独为这些服务器购买物理机器有点浪费，但是将这些低强度的服务器以虚拟机的形式运行在一个物理机上可以节省时间和资金*。

​	**虚拟机在云计算中使用的也非常广泛**。云厂商，例如AWS，不想直接出借物理服务器给用户，因为这很难管理。它们想向用户出借的是可以随意确定不同规格的服务器。或许有两个用户在一台物理服务器上，但是他们并没有太使用计算机，这样AWS可以继续向同一个物理服务器上加入第三或者第四个用户。这样可以不使用额外的资金而获得更高的收益。所以，<u>虚拟机提供了额外的灵活性，这里借助的技术是：将操作系统内核从之前的内核空间上移至用户空间，并在操作系统内核之下增加新的一层（注，也就是虚拟机的内核是运行在宿主机的用户空间，虚拟机的内核通过新增加的一层VMM来对接底层硬件）以提供这里的灵活性</u>。

​	还有一些其他的原因会使得人们使用虚拟机。第一个是开发内核，这就是为什么我们在课程中一直使用QEMU。能够在虚拟环境而不是一个真实的计算机运行XV6，使得这门课程对于你们和我们来说都要方便的多。同时<u>对于调试也更容易</u>，因为相比在物理计算机上运行XV6，在QEMU提供的虚拟机环境中运行可以更容易的提供gdb的访问权限。

​	**最后一个人们使用虚拟机的原因是，通过新增的VMM提供的抽象可以实现更多的功能。例如，你可以为整个操作系统和其中的用户进程做一个快照，并在磁盘中保存下来**。稍后再恢复快照，并将操作系统和其中的用户进程恢复成做快照时的状态。这可以增加运行的可靠性，或者用来调试，或者用来拷贝虚拟机的镜像并运行多次。除此之外，还可以将一个Guest操作系统迁移到另一个计算机上。如果你在一个物理计算机上运行了一个Guest操作系统，现在需要关闭并替换该物理计算机，你可以在不干扰虚拟机运行的前提下，将它迁移到另一个物理计算机，这样你就可以安全的关闭第一个物理计算机。

​	以上就是人们喜欢使用虚拟机的原因。虚拟机实际上应用的非常非常广泛，并且它也有着很长的历史。虚拟机最早出现在1960年代，经过了一段时间的开发才变得非常流行且易用。

​	对于这们课程来说，我们之所以要学习虚拟机是因为**VMM提供了对于操作系统的一种不同视角**。在操作系统的架构中，内核之上提供的封装单元（注，视频中说的是container，但是container还有容器的意思，所以这里说成是封装单元）是我们熟悉的进程，内核管理的是多个用户进程。而**在VMM的架构中，VMM之上提供的封装单元是对计算机的模拟。VMM的架构使得我们可以从另一个角度重新审视我们讨论过的内容，例如内存分配，线程调度等等，这或许可以给我们一些新的思路并带回到传统的操作系统内核中**。所以，在虚拟机场景下，大部分的开发设计研究工作，从传统的内核移到了VMM。**某种程度上来说，传统操作系统内核的内容下移了一层到了VMM**。

​	今天课程的第一部分我将会讨论如何实现我们自己的虚拟机。这里假设我们要模拟的是RISC-V，并运行针对RISC-V设计的操作系统，例如XV6。我们的目的是让运行在Guest中的代码完全不能区分自己是运行在一个虚拟机还是物理机中，因为我们希望能在虚拟机中运行任何操作系统，甚至是你没有听说过的操作系统，这意味着对于任何操作系统的行为包括使用硬件的方式，虚拟机都必须提供提供对于硬件的完全相同的模拟，这样任何在真实硬件上能工作的代码，也同样能在虚拟机中工作。

​	**除了不希望Guest能够发现自己是否运行在虚拟机中，我们也不希望Guest可以从虚拟机中逃逸**。很多时候人们使用虚拟机是因为它为不被信任的软件甚至对于不被信任的操作系统提供了**严格的隔离**。假设你是Amazon，并且你出售云服务，通常是你的客户提供了运行在虚拟机内的操作系统和应用程序，所以有可能你的客户运行的不是普通的Linux而是一个特殊的修改过的Linux，并且会试图突破虚拟机的限制来访问其他用户的虚拟机或者访问Amazon用来实现虚拟机隔离的VMM。所以Guest不能从虚拟机中逃逸还挺重要的。<u>Guest可以通过VMM使用内存，但是不能使用不属于自己的内存。类似的，Guest也不应该在没有权限的时候访问存储设备或者网卡。所以这里我们会想要非常严格的隔离</u>。**虚拟机在很多方面比普通的Linux进程提供了更加严格的隔离。Linux进程经常可以相互交互，它们可以杀掉别的进程，它们可以读写相同的文件，或者通过pipe进行通信。但是在一个普通的虚拟机中，所有这些都不被允许**。运行在同一个计算机上的不同虚拟机，彼此之间是通过VMM完全隔离的。所以出于安全性考虑人们喜欢使用虚拟机，这是一种可以运行未被信任软件的方式，同时又不用担心bug和恶意攻击。

​	前面已经指出了**虚拟机的目标是提供一种对于物理服务器的完全准确的模拟**。但是<u>实际中出于性能的考虑，这个目标很难达到。你将会看到运行在Guest中的Linux与VMM之间会相互交互，所以**实际中Linux可以发现自己是否运行在VMM之上**</u>。**<u>出于效率的考虑，在VMM允许的前提下，Linux某些时候知道自己正在与VMM交互，以获得对于设备的高速访问权限。但这是一种被仔细控制的例外，实现虚拟机的大致策略还是完全准确的模拟物理服务器</u>**。

## 19.2 trap-and-emulate之trap

> [19.2 Trap-and-Emulate --- Trap - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/365570440) <= 图文出处
>
> [关于trap and emulated的解释 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/382937263) <= 推荐阅读，可以先大致了解下trap-and-emulate指什么
>
> 运行在hypervisor之上的虚拟机操作系统是作为用户级的进程运行的。他们并没有和运行在bare metal上的Linux os具有一样的特权。但是由于这些虚拟机操作系统的代码未经任何修改，它们不知道它们不具有在bare metal上才能做的一些特权命令。即，当虚拟机操作系统执行某些特权指令时，要求它们必须运行在bare metal上，同时位于privileged mode或者kernel mode。这些命令会创造一个trap，即进入hypervisor，然后hypervisor仿真虚拟机需要的功能。这就是trap and emulate。
>
> 每个操作系统都以为它运行在bare metal上，会像在bare metal上一样发出指令。这意味着，它会试图执行一些特权指令，以为其具有正确的privilege。但是其并不具有相应的特权，因为它们是在hypervisor之上，作为用户级进程运行的。因此，当它们想要做一些需要高级别特权的操作时，it will result in a trap into the hypervisor，然后hypervisor会模仿该操作系统需要的功能。

​	**我们该如何构建我们自己的VMM呢？一种实现方式是完全通过软件来实现**，你可以想象写一个类似QEMU的软件，这个软件读取包含了XV6内核指令的文件，查看每一条指令并模拟RISC-V的状态，这里的状态包括了通过软件模拟32个寄存器。<u>你的软件读取每条指令，确定指令类型，再将指令应用到通过软件模拟的32个寄存器和控制寄存器中。实际中有的方案就是这么做的，虽然说考虑到细节还需要做很多工作，但是这种方案从概念上来说很简单直观</u>。

​	但是**纯软件解析的虚拟机方案应用的并不广泛，因为它们很慢**。如果你按照这种方式实现虚拟机，那么Guest应用程序的运行速度将远低于运行在硬件上，<u>因为你的VMM在解析每一条Guest指令的时候，都可能要转换成几十条实际的机器指令，所以这个方案中的Guest的运行速度比一个真实的计算机要慢几个数量级。在云计算中，这种实现方式非常不实用。所以人们并不会通过软件解析来在生产环境中构建虚拟机</u>。

​	相应的，**<u>一种广泛使用的策略是在真实的CPU上运行Guest指令</u>**。所以如果我们要在VMM之上运行XV6，我们需要先将XV6的指令加载到内存中，之后再跳转到XV6的第一条指令，这样你的计算机硬件就能直接运行XV6的指令。当然，这要求你的计算机拥有XV6期望的处理器（注，也就是RISC-V）。

​	<u>但是实际中你又不能直接这么做，因为当你的Guest操作系统执行了一个privileged指令（注，也就是在普通操作系统中只能在kernel mode中执行的指令，详见3.4）之后，就会出现问题。现在我们在虚拟机里面运行了操作系统内核，而内核会执行需要privileged权限指令，比如说加载一个新的Page Table到RISC-V的SATP寄存器中，而这时就会出现问题</u>。

​	**前面说过，我们将Guest kernel按照一个Linux中的普通用户进程来运行，所以Guest kernel现在运行在User mode，而在User mode加载SATP寄存器是个非法的操作，这会导致我们的程序（注，也就是虚拟机）crash**。<u>但是如果我们蠢到将Guest kernel运行在宿主机的Supervisor mode（注，也就是kernel mode），那么我们的Guest kernel不仅能够修改真实的Page Table，同时也可以从虚拟机中逃逸，因为它现在可以控制PTE（Page Table Entry）的内容，并且读写任意的内存内容。所以**我们不能直接简单的在真实的CPU上运行Guest kernel**</u>。

​	相应的，这里会使用一些技巧。

​	**首先将Guest kernel运行在宿主机的User mode，这是最基本的策略**。这意味着，当我们自己写了一个VMM，然后通过VMM启动了一个XV6系统，VMM会将XV6的kernel指令加载到内存的某处，再设置好合适的Page Table使得XV6看起来自己的内存是从地址0开始向高地址走。之后VMM会使用trap或者sret指令（注，详见6.8）来跳转到位于User mode的Guest操作系统的第一条指令，这样不论拥有多少条指令，Guest操作系统就可以一直执行下去。

![img](https://pic1.zhimg.com/80/v2-06d033bd948cce31db93640ca9208040_1440w.webp)

​	**一旦Guest操作系统需要使用privileged指令，因为它当前运行在User mode而不是Supervisor mode，会使得它触发trap并走回到我们的VMM中**（注，在一个正常操作系统中，如果在User mode执行privileged指令，会通过trap走到内核，但是现在VMM替代了内核），之后我们就可以获得控制权。所以当Guest操作系统尝试修改SATP寄存器，RISC-V处理器会通过trap走回到我们的VMM中，之后我们的VMM就可以获得控制权。并且我们的VMM也可以查看是什么指令引起的trap，并做适当的处理。这里核心的点在于Guest操作系统并没有实际的设置SATP寄存器。

![img](https://pic4.zhimg.com/80/v2-f21a3c967004a9cfcd8309d88aec2343_1440w.webp)

​	在RISC-V上，如果在User mode尝试运行任何一个需要Supervisor权限的指令都会触发trap。这里需要Supervisor权限的指令并不包括与Page Table相关的指令，我们稍后会介绍相关的内容。所以每当Guest操作系统尝试执行类似于读取SCAUSE寄存器，读写STVEC寄存器，都会触发一个trap，并走到VMM，之后我们就可以获得控制权。

---

**问题：VMM改如何截获Guest操作系统的指令？它应该要设置好一个trap handler对吧，但这不是一个拥有privileged权限的进程才能做的事情吗？而VMM又是个宿主机上的用户程序，是吧？**

**回答：<u>我这里假设VMM运行在Supervisor mode。所以在这里的图中，VMM就是宿主机的kernel。这里我们不是启动类似Linux的操作系统，而是启动VMM（注，类似VMware的ESXi）。VMM以privileged权限运行，并拥有硬件的完整控制权限，这样我们就可以在VMM里面设置各种硬件寄存器。有一些VMM就是这么运行的，你在硬件上启动它们，并且只有VMM运行在Supervisor mode。实际上还有很多很多其他的虚拟机方案，比如说在硬件上启动Linux，之后要么Linux自带一个VMM，要么通过可加载的内核模块将VMM加载至Linux内核中，这样VMM可以在Linux内核中以Supervisor mode运行</u>。今天我们要讨论的论文就是采用后者。这里主要的点在于，我们自己写的可信赖的VMM运行在Supervisor mode，而我们将不可信赖的Guest kernel运行在User mode，通过一系列的处理使得Guest kernel看起来好像自己是运行在Supervisor mode**。

## 19.3 trap-and-emulate之emulate

> [19.3 Trap-and-Emulate --- Emulate - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/365570804) <= 图文出处

​	**VMM会为每一个Guest维护一套虚拟状态信息**。所以VMM里面会维护虚拟的STVEC寄存器，虚拟的SEPC寄存器以及其他所有的privileged寄存器。当Guest操作系统运行指令需要读取某个privileged寄存器时，首先会通过trap走到VMM，因为在用户空间读取privileged寄存器是非法的。之后VMM会检查这条指令并发现这是一个比如说读取SEPC寄存器的指令，之后VMM会模拟这条指令，并将自己维护的**虚拟**SEPC寄存器，拷贝到trapframe的用户寄存器中（注，有关trapframe详见Lec06，这里假设Guest操作系统通过类似`sread a0, sepc`的指令想要将spec读取到用户寄存器a0）。之后，VMM会将trapframe中保存的用户寄存器拷贝回真正的用户寄存器，通过sret指令，使得Guest从trap中返回。这时，用户寄存器a0里面保存的就是SEPC寄存器的值了，之后Guest操作系统会继续执行指令。最终，Guest读到了VMM替自己保管的**虚拟**SEPC寄存器。

![img](https://pic1.zhimg.com/80/v2-879fa4bdf30af1ea946575e240dbfea4_1440w.webp)

> 学生提问：**VMM是怎么区分不同的Guest？**
>
> Robert教授：**VMM会为每个Guest保存一份虚拟状态信息**，然后它就像XV6知道是哪个进程一样，VMM也知道是哪个Guest通过trap走到VMM的。XV6有一个针对每个CPU的变量表明当前运行的是哪个进程，类似的VMM也有一个针对每个CPU的变量表明当前是哪个虚拟机在运行，进而查看对应的虚拟状态信息。
>
> 学生提问：VMM可以给一个Guest分配多个CPU核吗？
>
> Robert教授：稍微复杂点的VMM都可以实现。
>
> **学生提问：在实际的硬件中会有对应寄存器，那么为什么我们不直接使用硬件中的寄存器，而是使用虚拟的寄存器？**
>
> Robert教授：这里的原因是，**VMM需要使用真实的寄存器**。举个例子，想象一下SCAUSE寄存器，当Guest操作系统尝试做任何privileged操作时（注，也就是读写privileged寄存器），会发生trap。硬件会将硬件中真实的SCAUSE寄存器设置成引起trap的原因，这里的原因是因为权限不够。但是<u>假设Guest操作系统只是从Guest用户进程执行了一个系统调用，Guest操作系统需要看到SCAUSE的值是系统调用。也就是说Guest操作系统在自己的trap handler中处理来自Guest用户进程的系统调用时，需要SCAUSE的值表明是系统调用</u>。

![img](https://pic4.zhimg.com/80/v2-22aa4be6a2c0645d372025104585fc43_1440w.webp)

> **而实际的SCAUSE寄存器的值却表明是因为指令违反了privilege规则才走到的trap**。<u>通常情况下，VMM需要看到真实寄存器的值，而Guest操作系统需要能看到符合自己视角的寄存器的值。（注，在Guest操作系统中，可能有两种情况会触发trap，一种是Guest用户空间进程的系统调用，也就是正常操作系统中正常的trap流程，另一种是Guest内核空间读取privileged寄存器时，因为Guest内核空间实际上也是在宿主机的用户空间，导致这是个非法操作并触发trap</u>。Robert这边举的例子的流程应该是这样，Guest用户进程执行系统调用，在这一个瞬间SCAUSE寄存器的值是ECALL，也就是8，详见6.6。但是稍后在Guest系统内核的trap handler中需要读取SCAUSE的值，以确定在Guest中引起trap的原因，但是这就触发了第二种trap，SCAUSE的值会变成Illegal Access。我们不能让Guest系统内核看到这个值，所以VMM这里将它变成ECALL并返回。）

​	**在这种虚拟机的实现中，Guest整个运行在用户空间，任何时候它想要执行需要privilege权限的指令时，会通过trap走到VMM，VMM可以模拟这些指令。这种实现风格叫做Trap and Emulate**。你可以完全通过软件实现这种VMM，也就是说你可以只通过修改软件就将XV6变成一个可以运行在RISC-V上的VMM，然后再在之上运行XV6虚拟机。当然，与常规的XV6一样，<u>VMM需要运行在Supervisor mode</u>。

​	**所有以S开头的寄存器，也就是所有的<u>Supervisor控制寄存器</u>都必须保存在虚拟状态信息中**。<u>同时还有一些信息并不能直接通过这些控制寄存器体现，但是又必须保存在这个虚拟状态信息中</u>。**其中一个信息就是mode。VMM需要知道虚拟机是运行在Guest user mode还是Guest Supervisor mode**。例如，<u>Guest中的用户代码尝试执行privileged指令，比如读取SCAUSE寄存器，这也会导致trap并走到VMM。但是这种情况下VMM不应该模拟指令并返回，因为这并不是一个User mode中的合法指令。所以VMM需要跟踪Guest当前是运行在User mode还是Supervisor mode，所以在虚拟状态信息里面也会保存mode</u>。



![img](https://pic3.zhimg.com/80/v2-a32773106f370b82940fcdeb3e6b865e_1440w.webp)

​	**VMM怎么知道Guest当前的mode呢？当Guest从Supervisor mode返回到User mode时会执行sret指令，而sret指令又是一个privileged指令，所以会通过trap走到VMM，进而VMM可以看到Guest正在执行sret指令，并将自己维护的mode从Supervisor变到User**。

​	**虚拟状态信息中保存的另外一个信息是hartid，它代表了CPU核的编号。即使通过privileged指令，也不能直接获取这个信息，VMM需要跟踪当前模拟的是哪个CPU。**

![img](https://pic3.zhimg.com/80/v2-c44b0982a3396d0c9b9a0b66dfe8ccc6_1440w.webp)

​	实际中，在不同类型的CPU上实现Trap and Emulate虚拟机会有不同的难度。不过RISC-V特别适合实现Trap and Emulate虚拟机，因为RISC-V的设计人员在设计指令集的时候就考虑了Trap and Emulate虚拟机的需求。举个例子，设计人员确保了每个在Supervisor mode下才能执行的privileged指令，如果在User mode执行都会触发trap。你可以通过这种机制来确保VMM针对Guest中的每个privileged指令，都能看到一个trap。

> **学生提问：Guest操作系统内核中会实际运行任何东西吗？还是说它总是会通过trap走到VMM？**
>
> Robert教授：如果你只是执行一个ADD指令，这条指令会直接在硬件上以硬件速度执行。如果你执行一个普通的函数调用，代码的执行也没有任何特殊的地方。**所有User代码中合法的指令，以及内核代码中的non-priviledged指令，都是直接以全速在硬件上执行**。
>
> **学生提问：在Guest操作系统中是不是也有类似的User mode和Kernel mode？**
>
> Robert教授：**有的。Guest操作系统就是一个未被修改的普通操作系统，所以我们在Guest中运行的就是Linux内核或者XV6内核**。而XV6内核知道自己运行在Supervisor mode，从代码的角度来说，内核代码会认为自己运行在Supervisor mode，并执行各种privileged指令，并期望这些指令能工作。当Guest操作系统执行sret指令时，它也知道自己将要进入到User空间。**不过在宿主机上，Guest操作系统是运行在User mode，VMM也确保了这里能正常工作。但是从Guest角度来说，自己的内核看起来像是运行在Supervisor mode，自己的用户程序看起来像是运行在User mode**。

​	**所以，当Guest执行sret指令从Supervisor mode进入到User mode，因为sret是privileged指令，会通过trap进入到VMM。VMM会更新虚拟状态信息中的mode为User mode，尽管当前的真实mode还是Supervisor mode，因为我们还在执行VMM中的代码**。<u>在VMM从trap中返回之前，VMM会将真实的SEPC寄存器设置成自己保存在虚拟状态信息中的虚拟SEPC寄存器。因为当VMM使用自己的sret指令返回到Guest时，它需要将真实的程序计数器设置成Guest操作系统想要的程序计数器值（注，**因为稍后Guest代码会在硬件上执行，因此依赖硬件上的程序计数器**）。所以在一个非常短的时间内，真实的SEPC寄存器与虚拟的SEPC寄存器值是一样的。同时，当VMM返回到虚拟机时，还需要切换Page table，这个我们稍后会介绍</u>。

​	**Guest中的用户代码，如果是普通的指令，就直接在硬件上执行。当Guest中的用户代码需要执行系统调用时，会通过执行ECALL指令（注，详见6.3，6.4）触发trap，而这个trap会走到VMM中（注，因为ECALL也是个privileged指令）**。<u>VMM可以发现当前在虚拟状态信息中记录的mode是User mode，并且发现当前执行的指令是ECALL，之后VMM会更新虚拟状态信息以模拟一个真实的系统调用的trap状态。比如说，它将设置虚拟的SEPC为ECALL指令所在的程序地址（注，执行sret指令时，会将程序计数器的值设置为SEPC寄存器的值。这样，当Guest执行sret指令时，可以从虚拟的SEPC中读到正确的值）；将虚拟的mode更新成Supervisor；将虚拟的SCAUSE设置为系统调用；将真实的SEPC设置成虚拟的STVEC寄存器（注，STVEC保存的是trap函数的地址，将真实的SEPC设置成STVEC这样当VMM执行sret指令返回到Guest时，可以返回到Guest的trap handler。Guest执行系统调用以为自己通过trap走到了Guest内核，但是实际上却走到了VMM，这时VMM需要做一些处理，让Guest以及之后Guest的所有privileged指令都看起来好像是Guest真的走到了Guest内核）；之后调用sret指令跳转到Guest操作系统的trap handler，也就是STVEC指向的地址</u>。

## 19.4 trap-and-emulate之page table

​	**这一小节主要讲纯软件技术——影子页表(Shadow page table)如何使得VMM能维护客户虚拟机的页表；而EPT(Extended Page Table)需要依赖硬件支持**。

---

> [19.4 Trap-and-Emulate --- Page Table - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/365571294) <= 图文出处
>
> ---
>
> 下面学生提问提到的**EPT(Extended Page Table)**技术，网上查了一下相关的介绍文章。
>
> [硬件辅助虚拟化 之EPT(内存虚拟化)介绍_方晓山的博客-CSDN博客](https://blog.csdn.net/mad2006/article/details/128459519) <= 推荐阅读，图文并茂，虚拟机的物理地址GPA(guest physical address)到实际宿主机的HPA(host physical address) 转换。
>
> 在内存虚拟化技术下，所有上面提到的物理地址(PA)都将作为虚拟机的物理地址(GPA)，需要把所有的GPA借助于**CPU内部的EPT(Extended Page Table)寄存器**转化为HPA，才能被CPU给访问。所以，过程变为了下面这样：
>
> 虚拟机里面的操作系统使用的CR3寄存器存放的PML4T的起始物理地址(GPA)需要被VMM(Virtual Machine Monitor)转化为HPA，然后才能使用Index4找到PDPT的起始物理地址(GPA)。继续借助于EPT把PDPT的GPA转化为HPA，再利用Index3找到PDT的起始物理地址(GPA)。继续借助于EPT把PDT的GPA转化为HPA，再利用Index2找到PT的起始物理地址(GPA)。继续借助于EPT把PT的GPA转化为HPA，再利用Index1找到PFN的起始物理地址(GPA)。最后借助于EPT把PFN的GPA转化为HPA，加上页内偏移(offset)，得到最后的虚拟地址(VA)对应的HPA。
>
> [内存虚拟化硬件基础——EPT_享乐主的博客-CSDN博客_ept](https://blog.csdn.net/huang987246510/article/details/104650146/) <= 推荐阅读，也是图文并茂
>
> EPT克服了影子页表使用软件维护GVA->HPA地址转换的缺点，它使用**硬件**来维护GPA->HPA，因此效率大大提高。而且，影子页表虽然缩短了地址转换路径，但每次虚机进程访问CR3时，都会引起VMX的模式切换，开销很大。影子页表在每次加载和卸载的时候都会引起模式切换，而EPT减少了这种开销，EPT只在缺页的时候才引起VMX模式切换，一旦页表建立好之后，EPT就不再有模式切换的开销，虚机内存的访问一直在客户态。

​	有关Trap and Emulate的实现还有两个重要的部分，一个是Page Table，另一个是外部设备。

​	Page Table包含了两个部分，第一个部分是Guest操作系统在很多时候会修改SATP寄存器（注，SATP寄存器是物理内存中包含了Page Table的地址，详见4.3），当然这会变成一个trap走到VMM，之后VMM可以接管。**但是我们不想让VMM只是简单的替Guest设置真实的SATP寄存器，因为这样的话Guest就可以访问任意的内存地址，而不只是VMM分配给它的内存地址，所以我们不能让Guest操作系统简单的设置SATP寄存器**。

​	但是我们的确又需要为SATP寄存器做点什么，因为我们需要让Guest操作系统觉得Page Table被更新了。此外，当Guest上的软件运行了load或者store指令时，或者获取程序指令来执行时，我们需要数据或者指令来自于内存的正确位置，也就是Guest操作系统认为其PTE指向的内存位置。所以当Guest设置SATP寄存器时，真实的过程是，我们不能直接使用Guest操作系统的Page Table，VMM会生成一个新的Page Table来模拟Guest操作系统想要的Page Table。

​	所以现在的Page Table翻译过程略微有点不一样，首先是Guest kernel包含了Page Table，但是这里是将Guest中的虚拟内存地址（注，下图中gva）映射到了Guest的物理内存地址（注，下图中gpa）。Guest物理地址是VMM分配给Guest的地址空间，例如32GB。并且VMM会告诉Guest这段内存地址从0开始，并一直上涨到32GB。但是在真实硬件上，这部分内存并不是连续的。所以<u>我们不能直接使用Guest物理地址，因为它们不对应真实的物理内存地址</u>。

​	**相应的，VMM会为每个虚拟机维护一个映射表，将Guest物理内存地址映射到真实的物理内存地址，我们称之为主机物理内存地址（注，下图中的hpa）。这个映射表与Page Table类似，对于每个VMM分配给Guest的Guest物理内存Page，都有一条记录表明真实的物理内存Page是什么**。

![img](https://pic1.zhimg.com/80/v2-eded9289114d498643987d0ca56692bc_1440w.webp)

​	**当Guest向SATP寄存器写了一个新的Page Table时，在对应的trap handler中，VMM会创建一个Shadow Page Table，Shadow Page Table的地址将会是VMM向真实SATP寄存器写入的值**。

​	Shadow Page Table由上面两个Page Table组合而成，所以它将gva映射到了hpa。Shadow Page Table是这么构建的：

- 从Guest Page Table中取出每一条记录，查看gpa。
- 使用VMM中的映射关系，将gpa翻译成hpa。
- 再将gva和hpa存放于Shadow Page Table。

​	在创建完之后，VMM会将Shadow Page Table设置到真实的SATP寄存器中，再返回到Guest内核中（注，这样的效果是，Guest里面看到的Page Table就是一个正常的Page Table，而Guest通过SATP寄存器指向的Page Table，将虚拟内存地址翻译得到的又是真实的物理内存地址）。



![img](https://pic3.zhimg.com/80/v2-b1f9d4eeecbeae8e10e2471133b3ff1e_1440w.webp)

​	**所以，Guest kernel认为自己使用的是一个正常的Page Table，但是实际的硬件使用的是Shadow Page Table**。

​	**<u>这种方式可以阻止Guest从被允许使用的内存中逃逸。Shadow Page Table只能包含VMM分配给虚拟机的主机物理内存地址。Guest不能向Page Table写入任何VMM未分配给Guest的内存地址。这是VMM实现隔离的一个关键部分</u>**。

> **学生提问：如果Guest操作系统想要为一个进程创建一个新的Page Table，会发生什么呢？**
>
> Robert教授：Guest会完全按照Linux或者XV6的行为来执行。首先是格式化Page Table Entries以构造一个Page Table。之后执行指令将Page Table的地址写入到SATP寄存器，这就是Guest操作系统的行为。**但是它又不能设置实际的SATP寄存器，因为这是一个privileged操作，所以设置SATP寄存器会触发trap并走到VMM。VMM会查看trap对应的指令，并发现Guest要尝试设置SATP寄存器，之后VMM会创建一个新的Shadow Page Table**。VMM会查看Guest尝试要设置的Page Table的每一条记录，通过gpa->hpa的映射关系，将gva和hpa的对应关系翻译出来。**如果Guest尝试使用一个不被允许的物理地址，VMM会生成一个真实的Page Fault。之后VMM会将Shadow Page Table设置到真实的SATP寄存器中，并返回到Guest中**。

​	Shadow Page Table是实现VMM时一个比较麻烦的地方。除了设置SATP寄存器，Guest操作系统还有另一种方式可以与Page Table进行交互。XV6有时候会直接修改属于自己的Page Table Entry，或者读取PTE中的dirty bit。如果你读了RISC-V的文档，你可以发现在RISC-V上，如果软件更改了PTE，RISC-V不会做任何事情。如果你修改了PTE，RISC-V并不承诺可以立即观察到对于PTE的修改，在修改那一瞬间，你完全是不知道PTE被修改了（注，这里主要对比的是privileged指令，因为如果在用户空间执行了privileged指令，会立刻触发trap，而这里修改PTE不会有任何的额外的动作）。相应的，文档是这么说的，如果你修改PTE并且希望MMU可以看到这个改动，你需要执行`sfence.vma`指令，这个指令会使得硬件注意到你对Page Table的修改。所以如果你要自己写一个VMM，你在RISC-V上的VMM会完全忽略Guest对于PTE的修改，但是你知道Guest在修改完PTE之后将会执行`sfence.vma`指令，并且这是一个privileged指令，因为它以s开头，所以这条指令会通过trap走到VMM，VMM就可以知道sfence.vma指令被执行了。之后VMM会重新扫描Guest的当前Page Table，查找更新了的Page Table Entry。如果修改合法的话，VMM会将修改体现在Shadow Page Table中，并执行真实的sfence.vma指令来使得真实的硬件注意到Shadow Page Table的改动。最后再会返回到Guest操作系统中。

> 学生提问：所以MMU只使用了一个Page Table，也就是Shadow Page Table，对吧？这里并没有使用EPT（Extended Page Table），对吧？
>
> Robert教授：这里还没有EPT。
>
> 学生提问：所以Guest认为它自己有一个Page Table，也就是`gva->gpa`，但是这里并没有做任何的翻译工作。VMM通过两个映射关系构建了属于自己的Page Table。
>
> Robert教授：是的。这里澄清一下，**EPT是一种非常不一样的虚拟机实现方式，并且需要硬件的支持**。<u>我们这里假设除了对privileged指令触发trap以外，不需要使用任何特殊的硬件支持来构建一个虚拟机</u>。
>
> 学生提问：这里会弄乱direct mapping吗？
>
> Robert教授：这里不会有direct map。Guest会认为自己有一个direct mapping，但这只是在虚拟的世界里的一个direct mapping，在真实的机器上这不是direct mapping。但是这没有关系，因为我们这里欺骗了Guest使得看起来像是direct mapping。
> 学生提问：我们刚刚说过性能的损耗，如果我们使用VMM，对于这里的trap机制看起来也会有大量的性能损耗。
>
> Robert教授：是的。如果你的操作系统执行了大量的privileged指令，那么你也会有大量的trap，这会对性能有大的损耗。这里的损耗是现代硬件增加对虚拟机支持的动机。今天要讨论的论文使用的就是现在硬件对于虚拟机的支持，Intel和AMD在硬件上支持更加有效的trap，或者说对于虚拟机方案，会有少得多的trap。所以是的，性能很重要。但是上面介绍的方案，人们也使用了很多年，它能工作并且也很成功，尽管它会慢的多，但是还没有慢到让人们讨厌的程度，人们其实很喜欢这个方案。

## 19.5 trap-and-emulate之devices

> [19.5 Trap-and-Emulate --- Devices - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/365571452) <= 图文出处

​	接下来我们来看Trap and Emulate的最后一个部分，也就是**虚拟机的外部设备**。外部设备是指，一个普通的操作系统期望能有一个磁盘用来存储文件系统，或者是期望有一个网卡，甚至对于XV6来说期望有一个UART设备来与console交互，或者期望有一张声卡，一个显卡，键盘鼠标等等各种各样的东西。所以我们的虚拟机方案，需要能够至少使得Guest认为所有它需要的外部设备是存在的。

​	这里人们通常会使用三种策略。

​	**第一种是Emulation，模拟一些需要用到的并且使用非常广泛的设备**，例如磁盘。也就是说，Guest并不是拥有一个真正的磁盘设备，只是VMM使得与Guest交互的磁盘看起来好像真的存在一样。这里的实现方式是，Guest操作系统仍然会像与真实硬件设备交互一样，通过Memory Map控制寄存器与设备进行交互。通常来说，操作系统会假设硬件已经将自己的控制寄存器映射到了内核地址空间的某个地址上。**在VMM中不会映射这些内存地址对应的Page，相应的会将这些Page设置成无效。这样当Guest操作系统尝试使用UART或者其他硬件时，一访问这些地址就会通过trap走到VMM**。VMM查看指令并发现Guest正在尝试在UART发送字符或者从磁盘中读取数据。VMM中会对磁盘或者串口设备有一些模拟，通过这些模拟，VMM知道如何响应Guest的指令，之后再恢复Guest的执行。这就是我们之前基于QEMU介绍XV6时，QEMU实现UART的方式。在之前的介绍中，并没有UART硬件的存在，但是QEMU模拟了一个UART来使得XV6正常工作。**这是一种常见的实现方式，但是这种方式可能会非常的低效，因为每一次Guest与外设硬件的交互，都会触发一个trap**。但是对于一些低速场景，这种方式工作的较好。**如果你的目标就是能启动操作系统并使得它们完全不知道自己运行在虚拟机上，你只能使用这种策略**。

![img](https://pic1.zhimg.com/80/v2-40387ba25b2d9fcd80192a358125d738_1440w.webp)

​	**在现代的世界中，操作系统在最底层是知道自己运行在虚拟机之上的。所以第二种策略是提供虚拟设备(Virtual Device)，而不是模拟一个真实的设备**。<u>通过在VMM中构建特殊的设备接口，可以使得Guest中的设备驱动与VMM内支持的设备进行高效交互。现在的Guest设备驱动中可能没有Memory Mapped寄存器了，但是相应的在内存中会有一个命令队列，Guest操作系统将读写设备的命令写到队列中</u>。在XV6中也使用了一个这种方式的设备驱动，在XV6的`virtio_disk.c`文件中，你可以看到一个设备驱动尝试与QEMU实现的虚拟磁盘设备交互。在这个驱动里面要么只使用了很少的，要么没有使用Memory Mapped寄存器，所以它<u>基本不依赖trap，相应的它在内存中格式化了一个命令队列</u>。之后QEMU会从内存中读取这些命令，但是并不会将它们应用到磁盘中，而是将它们应用到一个文件，对于XV6来说就是`fs.image`。**这种方式比直接模拟硬件设备性能要更高，因为你可以在VMM中设计设备接口使得并不需要太多的trap**。

![img](https://pic3.zhimg.com/80/v2-a5a852321c54987f0b1847d523f3ec22_1440w.webp)

​	**第三个策略是对于真实设备的pass-through，这里典型的例子就是网卡**。<u>现代的网卡具备硬件的支持，可以与VMM运行的多个Guest操作系统交互。你可以配置你的网卡，使得它表现的就像多个独立的子网卡，每个Guest操作系统拥有其中一个子网卡。经过VMM的配置，Guest操作系统可以直接与它在网卡上那一部分子网卡进行交互，并且效率非常的高。所以这是现代的高性能方法</u>。**在这种方式中，Guest操作系统驱动可以知道它们正在与这种特别的网卡交互**。

![img](https://pic4.zhimg.com/80/v2-0c7e298e86ae96d72bda8b36ad37b127_1440w.webp)

​	以上就是实现外部设备的各种策略。我认为**在实现一个VMM时，主要的困难就在于构建外部设备和设备驱动，并使得它们能正确的与Guest操作系统配合工作。这里或许是实现VMM的主要工作，尤其是当你使用第一种策略时**。

----

**问题：我并没有太理解策略一emulation和策略二virtual device的区别。**

**回答：它们是类似的。可以这么想，如果你启动了一个完全不知道虚拟机的操作系统，它或许包含了很多磁盘驱动，但是所有的驱动都是为真实硬件提供的。如果你想要在虚拟机中启动这样一个操作系统，你需要选择其中一种真实的硬件，并且以一种非常准确的方式来模拟该硬件。这种方式并没有问题，只是<u>大部分情况下硬件接口并没有考虑Trap and Emulate VMM下的性能。所以真实的设备驱动需要你频繁地读写它的控制寄存器，而VMM需要为每一次写控制寄存器都获取控制权，因为它需要模拟真实的硬件。这意味着每一次写控制寄存器都会触发一次trap走到VMM，并消耗数百个CPU cycles。所以策略一非常的慢且低效</u>。策略二并没有卑微地模仿真实的设备，某些设计人员提出了一种设备驱动，这种设备驱动并不对接任何真实的硬件设备，而是只对接由VMM实现的虚拟设备。这种驱动设计的并不需要很多trap，并且这种驱动与对应的虚拟设备是解耦的，并不需要立即的交互。<u>从功能层面上来说，使用策略一的话，你可以启动任何操作系统，使用策略二的话，如果你想要使用虚拟设备，你只能启动知道虚拟设备的操作系统</u>。<u>实际中，策略二是一种标准，并且很多虚拟机的实现方案都能提供</u>。虽然我们并没有在除了QEMU以外的其他场景测试过，XV6中的`virtio_disk.c`稍作修改或许也可以在其他虚拟机方案上运行。**

**追问：所以对于每一种主板，取决于不同的磁盘，编译XV6都需要不同的磁盘驱动，是吗？**

**回答：是的。我认为或许你可以买到支持`virtio_disk`驱动的真实硬件，但是大部分的磁盘硬件还不支持这个驱动，这时你需要为真实的硬件实现一种新的驱动**。

## 19.6 硬件对虚拟机的支持

> [19.6 硬件对虚拟机的支持 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/365571701) <= 图文出处
>
> [VT-x_百度百科 (baidu.com)](https://baike.baidu.com/item/VT-x/10210913?fr=aladdin)
>
> VT-x是[intel](https://baike.baidu.com/item/intel/125450?fromModule=lemma_inlink)运用[Virtualization](https://baike.baidu.com/item/Virtualization/10102404?fromModule=lemma_inlink)虚拟化技术中的一个[指令](https://baike.baidu.com/item/指令/18765029?fromModule=lemma_inlink)集。VT-x有助于提高基于软件的虚拟化解决方案的灵活性与稳定性。通过按照纯软件虚拟化的要求消除[虚拟机监视器](https://baike.baidu.com/item/虚拟机监视器/18750511?fromModule=lemma_inlink)(VMM）代表客户操作系统来听取、中断与执行特定指令的需要，不仅能够有效减少 VMM 干预，还为 VMM 与客户操作系统之间的传输平台控制提供了有力的硬件支持，这样在需要 VMM干预时，将实现更加快速、可靠和安全的切换。

​	接下来我将讨论硬件对于虚拟机的支持，这里特指的就是Intel的**VT-x**。为什么Intel和其他的硬件厂商会为虚拟机提供直接的硬件支持呢？

- 首先虚拟机应用的非常广泛，硬件厂商的大量客户都在使用虚拟机
- 其次，我们刚刚描述的Trap and Emulate虚拟机方案中，经常会涉及到大量高成本的trap，所以这种方案性能并不特别好。
- 第三个原因或许就没那么有趣了。RISC-V非常适合Trap and Emulate虚拟机方案，但是Intel的x86处理器的一些具体实现使得它可以支持虚拟化，但是又没那么容易。所以Intel也有动力来修复这里的问题，因为它的很多客户想要在x86上运行VMM。

![img](https://pic3.zhimg.com/80/v2-010f77447e7c941dc1502a180c5e82de_1440w.webp)

​	<u>这里硬件上的支持，是为了让人们能够更容易的构建可以更快运行的虚拟机。它已经存在了10年左右了，并且现在在构建虚拟机时使用的非常非常广泛</u>。**<u>在Trap and Emulate方案中，VMM会为每个Guest在软件中保存一份虚拟状态信息，而现在，这些虚拟状态信息会保存在硬件中。这样Guest中的软件可以直接执行privileged指令来修改保存在硬件中的虚拟寄存器，而不是通过trap走到VMM来修改VMM中保存在软件中的虚拟寄存器。所以这里的目标是Guest可以在不触发trap的前提下，执行privileged指令</u>**。

​	**我们还是有一个VMM在内核空间，并且Guest运行在用户空间。当我们使用这种新的硬件支持的方案时，我们的VMM会使用真实的控制寄存器，而<u>当VMM通知硬件切换到Guest mode时，硬件里还会有一套完全独立，专门为Guest mode下使用的虚拟控制寄存器。在Guest mode下可以直接读写控制寄存器，但是读写的是寄存器保存在硬件中的拷贝，而不是真实的寄存器</u>**。

![img](https://pic3.zhimg.com/80/v2-b991eb48a90d47b56c907e7d23cfbb2a_1440w.webp)

​	**硬件会对Guest操作系统的行为做一些额外的操作，以确保Guest不会滥用这些寄存器并从虚拟机中逃逸**。<u>在这种硬件支持的虚拟机方案中，存在一些技术术语，至少Intel是这么叫的，Guest mode被称为non-root mode，Host mode中会使用真实的寄存器，被称为root mode</u>。所以，**硬件中保存的寄存器的拷贝，或者叫做虚拟寄存器是为了在non-root mode下使用，真实寄存器是为了在root mode下使用**。

![img](https://pic1.zhimg.com/80/v2-dbcc57b5a911050c82880d27689129e4_1440w.webp)

​	<u>现在，当我们运行在Guest kernel时，可以在不触发任何trap的前提下执行任何privileged指令。比如说如果想读写STVEC寄存器，硬件允许我们直接读写STVEC寄存器的non-root拷贝</u>。这样，privileged指令可以全速运行，而不用通过trap走到VMM。这对于需要触发大量trap的代码，可以运行的快得多。

​	**现在当VMM想要创建一个新的虚拟机时，VMM需要配置硬件。在VMM的内存中，通过一个结构体与VT-x硬件进行交互**。<u>这个结构体称为VMCS(注，Intel的术语，全称是Virtual Machine Control Structure)。当VMM要创建一个新的虚拟机时，它会先在内存中创建这样一个结构体，并填入一些配置信息和所有寄存器的初始值，之后VMM会告诉VT-x硬件说我想要运行一个新的虚拟机，并且虚拟机的初始状态存在于VMCS中。Intel通过一些新增的指令来实现这里的交互</u>。

- VMLAUNCH，这条指令会创建一个新的虚拟机。你可以将一个VMCS结构体的地址作为参数传给这条指令，再开始运行Guest kernel。
- VMRESUME。在某些时候，Guest kernel会通过trap走到VMM，然后需要VMM中需要通过执行VMRESUME指令恢复代码运行至Guest kernel。
- VMCALL，这条新指令在non-root模式下使用，它会使得代码从non-root mode中退出，并通过trap走到VMM。

![img](https://pic3.zhimg.com/80/v2-3a9c770a3b1f820048319669d53abf86_1440w.webp)

​	通过硬件的支持，Guest现在可以在不触发trap的前提下，直接执行普通的privileged指令。但是还是有一些原因需要让代码执行从Guest进入到VMM中，其中一个原因是调用VMCALL指令，另一个原因是**设备中断，例如定时器中断会使得代码执行从non-root模式通过trap走到VMM**。<u>所以通常情况下设备驱动还是会使得Guest通过trap走回到VMM。这表示着Guest操作系统不能持续占有CPU，每一次触发定时器中断，VMM都会获取控制权。**如果有多个Guest同时运行，它们可以通过定时器中断来分时共享CPU**（注，类似于线程通过定时器中断分时共享CPU一样）</u>。

​	**VT-x机制中的另外一大部分是对于Page Table的支持**。当我们在Guest中运行操作系统时，我们仍然需要使用Page Table。首先Guest kernel还是需要属于自己的Page Table，并且会想要能够加载CR3寄存器，这是Intel中类似于SATP的寄存器(用于存储多级页表中顶级页表的物理地址)。VT-x使得Guest可以加载任何想要的值到CR3寄存器，进而设置Page Table。而<u>硬件也会执行Guest的这些指令，这很好，因为现在Guest kernel可以在不用通过trap走到VMM再来加载Page Table</u>。

![img](https://pic4.zhimg.com/80/v2-0747eda4afcb144775063d6170dbc2cb_1440w.webp)

​	但是我们也不能让Guest任意的修改它的Page Table，因为如果这样的话，Guest就可以读写任意的内存地址。所以VT-x的方案中，还存在另一个重要的寄存器：**EPT (Extended Page Table)。EPT会指向一个Page Table。当VMM启动一个Guest kernel时，VMM会为Guest kernel设置好EPT，并告诉硬件这个EPT是为了即将运行的虚拟机准备的**。

​	**之后，当计算机上的MMU在翻译Guest的虚拟内存地址时，它会先根据Guest设置好的Page Table，将Guest虚拟地址（gva）翻译到Guest 物理地址（gha）。之后再通过EPT，将Guest物理地址（gha）翻译成主机物理地址（hpa）。硬件会为每一个Guest的每一个内存地址都自动完成这里的两次翻译**。

​	EPT使得VMM可以控制Guest可以使用哪些内存地址。Guest可以非常高效的设置任何想要的Page Table，因为它现在可以直接执行privileged指令。但是**Guest能够使用的内存地址仍然被EPT所限制，而<u>EPT由VMM所配置，所以Guest只能使用VMM允许其使用的物理内存Page</u>**（注，EPT类似于19.4中的Shadow Page Table）。

![img](https://pic1.zhimg.com/80/v2-ffc78b205741c259608c6fb50e7cf92c_1440w.webp)

---

问题：我对于硬件中保存的虚拟寄存器有问题，如果你有两个CPU核，然后你想要运行两个虚拟机，你会得到多少虚拟寄存器？

回答：**每一个CPU核都有一套独立的VT-x硬件**。所以每一个CPU核都有属于自己的32个通用寄存器，属于自己的真实的控制寄存器，属于自己的用在Guest mode下的虚拟控制寄存器，属于自己的EPT，所以你可以在<u>两个CPU核上运行两个不同的虚拟机，它们不会共用任何寄存器，每个CPU核都有属于自己的寄存器</u>。

追问：那也需要一个新的VMM吗？

回答：VMM可以像一个普通的操作系统一样。XV6可以支持多个进程，并且为每个进程配备一个proc结构体。而我们的**VMM也会为每个虚拟机配备一个vm结构体，用来跟踪Guest的信息。并且，如我之前所说的，如果你只有一个CPU核，但是有3个Guest，可以通过定时器中断结合VMM在3个Guest之间切换**。

## 19.7 Dune: Safe User-level Access to Privileged CPU Features

> [19.7 Dune: Safe User-level Access to Privileged CPU Features - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/365571886) <= 图文出处

​	今天要讨论的[论文](https://pdos.csail.mit.edu/6.828/2020/readings/belay-dune.pdf)利用了上一节介绍的硬件对于虚拟机的支持(VT-x)，但是却将其用作其他的用途，这是这篇论文的有趣之处，它利用了这种完全是为了虚拟机而设计的硬件，但是却用来做一些与虚拟机完全无关的事情。<u>从一个全局的视角来看这篇论文的内容，它想要实现的是普通的进程。所以现在我们的场景是在一个Linux而不是VMM中，但是我们又用到了硬件中的VT-x。我们将会在Linux中加载Dune可加载模块，所以Dune作为kernel的一部分运行在Supervisor mode（注，又或者叫做kernel mode），除此之外，内核的大部分还是原本的Linux</u>。

​	因为这里运行的是Linux进程，所以我们期望Dune可以支持进程，以及包括系统调用在内的各种Linux进程可以做的事情。**不过现在我们想要使用VT-x硬件来使得普通的Linux进程可以做一些额外的事情。Dune会运行一些进程，或者说允许一个进程切换到Dune模式，这意味着，之前的进程只是被Page Table保护和隔离，现在这个进程完全被VT-x机制隔离开了**。现在进程有了一套完整的虚拟控制寄存器，例如CR3寄存器，并且这些进程可以运行在non-root Supervisor mode，所以**它可以在VT-x管理的虚拟状态信息上直接执行所有的privileged指令**。

​	基于上面的描述，**Dune管理的进程可以通过属于自己的CR3寄存器，设置好属于自己的Page Table。当然Dune也会控制属于这个进程的EPT，EPT会被设置的只包含这个进程相关的内存Page**。所以进程可以向CR3寄存器写入任意的Page Table地址，但是<u>因为MMU会在翻译完正常的Page Table之后再将地址送到EPT去翻译，所以进程不能从分配给它的内存中逃逸。所以进程并不能修改其他进程或者kernel的内存，它只是有了一种更灵活的设置自己内存的方式</u>。

![img](https://pic1.zhimg.com/80/v2-a55971aa7e42fc6b0277444758031f84_1440w.webp)

​	**Dune管理的进程也可以拥有Guest Supervisor mode和Guest User mode，就像一个小的虚拟机一样，并且可以保护运行在Supervisor mode的代码，不受运行在User mode的代码影响**。

![img](https://pic4.zhimg.com/80/v2-681cc1c8320aefb4407a42e7fad315bb_1440w.webp)

​	论文中提到了可以基于Dune做的两件事情：

​	首先，**<u>Dune能够在硬件层面支持进程同时拥有Guest Supervisor mode和Guest User mode，这样进程可以在自己的User mode中运行未被信任的插件代码</u>**。这里的主进程或许是一个网页浏览器，你可以为浏览器下载并运行各种各样的插件，或许是一个新的视频解码器，一个新的广告拦截插件等等。但是我们并不能完全信任这个插件，所以我们希望能够在权限受控的前提下运行它。虽然一个普通的Linux也可以达到这个目的，但是会比较麻烦。<u>通过Dune，我们可以在Guest User mode下运行插件，同时让网页浏览器运行在进程的Guest Supervisor mode下。因为现在可以修改CR3寄存器，所以可以为Guest User mode配置一个不同的Page Table。这样，即使插件是恶意的，进程也可以安全的运行这里的未被信任的插件代码，因为插件代码现在不能任意的读写主浏览器的内存，只能访问网页浏览器指定的某些内存Page。进程的Guest User代码可能会执行系统调用，但是这些系统调用会通过trap走到进程的Guest Supervisor mode，而不是Linux内核，所以这里的插件代码或许会认为自己调用了fork/read/write等系统调用，但是实际上这里尝试运行的系统调用通过trap走到了进程对应的网页浏览器，而网页浏览器可以做任意的事情，它可以选择执行或者不执行系统调用。所以现在网页浏览器对于插件代码有了完全的控制能力</u>。

​	**公平的说，这里提到的隔离效果可以通过Linux中一些非常不一样的技术来实现，但是Dune通过使用VT-x应将，为你可以提供一个特别优雅且有效的实现方式**。

​	**进程可以做的另一个事情是：通过Dune，进程的垃圾回收(Garbage Collect，GC)变得更快了**。在这个场景中，没有了Guest Supervisor mode和Guest User mode。假设我们在运行任意一种带有GC的编程语言，比如说Java或者Python。GC可能会很慢，并且本身有着非常非常多的技术可以使得GC变快。**许多GC都会扫描并找到位于内存中仍然在使用的对象，扫描会从寄存器中保存的对象指针开始，依次找到所有正在使用对象的所有指针。如果在扫描之后没能找到某个对象，那说明这个对象不被任何指针引用，那么它就可以被释放了**。许多GC会同时在主程序的一个线程中运行，所以GC会从寄存器中保存的指针开始，根据指针之间的树或者图的关系，扫描一个个的对象。

![img](https://pic4.zhimg.com/80/v2-3ccb5c6a32606666df3df520d972518f_1440w.webp)

​	<u>但是因为GC与程序本身是并行的在运行，所以程序可能会修改GC已经扫描过的对象，这很糟糕，因为这样的话，GC在扫描完成之后确定的要释放和不能释放的对象清单可能就不再正确了</u>。**Dune使用了Page Table Entry中的一个特性来帮助GC检测这样的修改。Dune管理的进程首先会设置好由VT-x提供的虚拟CR3寄存器，指向属于自己的Page Table，其中的PTE都是有效的。每一条PTE的dirty位，表明对于对应的Page存在写操作。所以如果程序在GC的过程中修改了某些对象，那么对应PTE的dirty位会被设置为1。当GC查找完所有的对象之后，它会查看所有PTE中的dirty位，找到包含了可能修改过的对象的内存Page，然后再重新扫描这些对象**。

​	**实际中，获取PTE dirty位的过程在普通的Linux中既困难又慢，我甚至都不确定Linux是否支持这个操作，在一些其他操作系统中你可以通过系统调用来查询PTE的dirty位**。

​	**但是如果你使用Dune和 VT-x，进程可以很快的使用普通的load和store指令获取PTE，进而获取dirty位。所以这里，Dune使得某些需要频繁触发GC的程序明显变得更快**。

> **学生提问：如果Guest User mode中的插件程序想要运行自己的GC会怎样？**
>
> Robert教授：现在我们使用了Dune，并且有一个进程是被Dune管理的。这个进程通过VT-x实现了Supervisor mode和User mode，我们在User mode运行了一个插件，并且插件也是由带GC的编程语言写的，所以它有属于自己的Page Table，并且其中的PTE也包含了dirty位。但是刚刚说的GC加速在这不能工作，因为Dune会将插件运行在Guest User mode，而就像普通的User mode一样，Guest User mode不允许使用CR3寄存器。所以**在Guest User mode，我们不能快速的访问PTE的dirty位。只有在Guest Supervisor mode，才能通过CR3寄存器访问Page Table。所以，并不能同时使用以上Dune提供的两种功能。**

![img](https://pic2.zhimg.com/80/v2-5bfb0790250f532d49c5cbca7d9fa3c9_1440w.webp)

> 学生提问：如果某人基于Dune写了个浏览器，那么对于不支持Dune的计算机来说就很难使用这样的浏览器，对吗？就像很难直接让Chrome使用Dune，因为不是所有的计算机都有这个内核模块。
>
> Robert教授：首先，这里提到的内容需要运行在支持VT-x的计算机上，也就是说底层的计算机必须支持VT-x，所以需要VT-x来运行Dune。其次Dune需要被加载来运行浏览器以利用前面说到的特性。所以是的，你需要将所有的东西都设置好。并且<u>Dune是一个研究项目，它的目标是使得人们去思考可以部署在真实世界，并且有足够的价值的一些东西。就像Linux一样，Linux有成千上万个功能，如果某人决定将Dune添加到Linux中作为一个标准功能，那么我们就可以依赖这个功能，并且Chrome也可以直接用它了</u>。
>
> 学生提问：所以从整体来看，这里就像是创建了一个VM，但是实际上运行的又是一个进程？
>
> Robert教授：你可以这么描述。这里主要是对进程进行抽象，但是这里没有用Page Table硬件来实现进程间的隔离（注，其实也使用了，但是主要不依赖Page Table硬件），这里使用的是CPU上的硬件来支持进程，这里说的CPU上的硬件就是VT-x，它包含了一些额外的功能，例如设置好属于进程的Page Table。
>
> **学生提问：论文里提到了，如果Dune管理的一个进程fork了，那就会变成一个不被Dune管理的进程，这不会是一个安全漏洞吗？比如说你通过Dune运行了一个进程，并且认为它现在是安全的。但是fork之后的进程因为不被管理所以可能会逃逸。**
>
> **Robert教授：Dune管理的进程的Guest Supervisor mode中，不存在安全的问题。这部分代码已经拥有了相应的权限，通过fork也不会获得更多的权限。但是另一方面，Dune的Guest User mode代码中，我们有未被信任的代码，如果让它在没有Dune管理的情况下运行会有一定的风险。所以这部分代码不能fork，如果它尝试执行fork系统调用，会通过trap走到进程的Guest Supervisor mode**。

![img](https://pic3.zhimg.com/80/v2-4ce4d818ea764343396af088918213a2_1440w.webp)

> 假设进程的Guest Supervisor mode部分代码写的非常的小心，并且不会被欺骗，那么它不会执行fork，所以这时fork不能工作。如果Supervisor mode的代码允许fork，它会调用Linux的fork系统调用，并得到一个fork进程包含了与原进程有相同的内存镜像，所以我们在新进程中包含可能是恶意的插件代码。如果新进程没有意识到Dune已经被关闭了，那么原来的Supervisor mode中的privileged指令会是非法的。所以我们需要假设Dune管理的进程里面的Supervisor mode部分代码能够足够的小心且足够的聪明，来阻止User mode中的插件代码执行fork。
>
> **学生：被Dune管理的进程拥有Supervisor mode并没有不安全，因为它实际上是non-root mode下的Supervisor mode，就像是Guest操作系统中的Supervisor mode一样，你可以让它做任何事情，因为VT-x的存在，进程就像是一个虚拟机一样，并不能伤害到真正的操作系统。**
>
> **Robert教授：是的，进程不能逃逸出来，因为存在EPT，而EPT会限制进程的地址空间。**
>
> **学生提问：在VT-x的方案中，当我们访问Page Table时，因为我们需要通过EPT进行第二层翻译，将Guest物理内存地址翻译到Host物理内存地址，这样从Page Table返回的延时是不是增加了？**
>
> Robert教授：**这里可能会花费更多的时间让硬件MMU来翻译内存地址**。在最坏的情况下，比如在RISC-V中，会有多层Page Table，MMU需要一层一层的去查找PTE，x86下同样也有多层Page Table，所以在x86中首先会查找主Page Table，如果要访问更多的内存地址，**每一次内存地址的访问都需要再次走到EPT，而EPT也是一个多层的Page Table**。所以我并不知道最差情况下需要访问Page Table多少次才能完成翻译，但是很明显在VT-x下会比普通情况下差得多。**不过实际中会有cache所以通常不会走到最坏的情况**。
>
> 学生提问：今天的虚拟机还是普遍会更慢吗？如果是的话，AWS是怎么工作的，因为看起来还挺快的，并且工作的也很好。
>
> Robert教授：我认为他们使用了硬件上的VT-x支持，并且使用了我们讨论过的一些功能，这样使得AWS虚拟机比较快，或者并不比真实的计算机慢多少。
>
> 学生提问：我对于Trap and Emulate中的Shadow Page Table有个问题，每次都会创建Shadow Page Table吗？难道不能记住上次的Shadow Page Table吗？
>
> Robert教授：**VMM需要创建新的Shadow Page Table以供真实的硬件使用**。当然在很多时候都可以增加缓存，对于一个聪明的VMM，它可以注意到Guest更新了一个PTE，VMM可以做相应的有限的工作来更新Shadow Page Table。如果机器是在多个虚拟机上分时复用的，VMM会为还在运行的虚拟机保存Shadow Page Table，这样这些虚拟机可以在恢复时直接重用。
>
> **学生提问：这难道不是意味着VMM为每个虚拟机中的每个进程都保存了Shadow Page Table的拷贝？**
>
> **Robert教授：是的，虚拟机里面有很多很多个Page Table，所以维护Shadow Page Table需要大量的工作。而类似于VT-x的硬件支持使得这部分工作更加的容易了，因为EPT表示你不用构建Shadow Page Table了**。
>
> **学生提问：我有个问题有关GC的，如果有dirty位的话需要重新扫描对象，那么有没有可能会无限扫描？**
>
> **Robert教授：是的，这有个问题，如果一直有对象在更新，扫描能正常结束吗？<u>实际中，GC会先扫描一次，之后它会冻结除了GC线程以外的其他线程，所以这期间不可能再发生任何其他的变更</u>。之后GC才会查看所有PTE的dirty位，但是因为其他所有线程都冻结了，所以不可能会有更多的dirty位了，所以GC查看了所有的dirty位，之后结束GC会结束扫描并创建需要释放对象的列表，最后再恢复所有之前冻结的线程的执行。GC是一个复杂的流程，Dune的论文中并没有足够的篇幅讨论它。**

# Lecture20 内核与高级编程语言Kernels and High-Level-Language(HLL)

## 20.1 操作系统与C语言

> [20.1 C语言实现操作系统的优劣势 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367506082) <= 更详细的网络博主的笔记，部分文字摘自该博文

​	本节课基于该[论文](https://pdos.csail.mit.edu/6.828/2020/readings/biscuit.pdf)进行展开，主要讨论编程操作系统时语言的选择与优劣对比。

​	市面上不少操作系统由C语言实现，比如Windows、Linux、BSD等。

​	为什么它们都是用C实现的呢？

- 首先C提供了大量的控制能力，从我们的实验中你可以看到，C可以完全控制内存分配和释放
- C语言几乎没有隐藏的代码，你几乎可以在阅读C代码的时候想象到对应的RISC-V机器指令是什么
- 通过C可以有**直接内存访问能力**，你可以读写PTE的bit位或者是设备的寄存器
- 使用C会有极少的依赖，因为你不需要一个大的程序运行时。你几乎可以直接在硬件上运行C程序。你们可以在XV6启动过程中看到这一点， 只通过几行汇编代码，你就可以运行C代码

​	以上就是C代码的优点，也是我们喜欢C语言的原因。但是C语言也有一些缺点。

​	比较典型的缺点即，很难用C语言写出安全的代码，就算是高手，也难免遇到以下几个典型安全问题：

+ buffer overrun：数组越界等
+ use-after-free bugs：你可能会释放一些仍然在使用的内存，之后其他人又修改了这部分内存
+ threads sharing dynamic memory：多线程操作动态内存的问题

​	CVEs一个跟踪所有的安全漏洞的组织，如果你查看他们的网站，你可以发现，在2017年有40个Linux Bugs可以让攻击者完全接管机器。很明显，这些都是非常严重的Bugs，这些Bug是由buffer overrun和一些其他memory-safety bug引起。这就太糟糕了，如果你用C写代码，就很难能够完全正确运行。当然，我可以肯定你们在之前的实验中都见过了这些Bug，之前在课程论坛上的一些问题涉及了use-after-free Bug。特别是在copy-on-write lab中，这些问题出现了好几次。

## 20.2 高级编程语言实现操作系统的优劣势

> [20.2 高级编程语言实现操作系统的优劣势 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367506321) <= 图文出处

​	**高级编程语言吸引人的一个原因是它提供了memory-safety，所以上一节中CVEs提到的所有Bugs，都将不再存在**。要么当它们发生时程序运行时会检查数组是否越界，如果越界了就panic；要么高级编程语言不允许你写出引起Bug的代码，所以这些问题完全不可能出现。

​	当然，高级编程语言还有一些其他的优点：

- Type safety：类型安全
- Automatic memory management with garbage collector：通过GC实现了自动的内存管理，所以free更容易了，你都不用去考虑它，GC会为你完成所有的内存释放工作
- Concurrency：对并发更友好
- Abstraction：有更好的抽象，接口和类等面向对象的语法使得你可以写出更加模块化的代码

​	高级编程语言有这么多优势，你不禁会想它有哪些缺点呢？为什么XV6或者Linux没有用Java，Golang，Python来写？

​	这里的原因是高级编程语言通常有更差的性能。高级编程语言通常都有一些额外的代价，这被称为High Level Language Tax。

- Bounds, cast, nil-pointer checks：比如说在索引一个数组元素时检查数据边界，比如说检查空指针，比如说类型转换。
- Garbage collection：除此之外，GC也不是没有代价的，需要花费一些时间来跟踪哪些对象可以被释放。

除了性能之外，**高级编程语言与内核编程本身不兼容**。

- No direct memory access：比如说高级编程语言没有直接访问内存的能力，因为这从原则上违反了Type safety。
- No hand-written assembly：<u>高级编程语言不能集成汇编语言，而在内核中的一些场景你总是需要一些汇编程序，比如说两个线程的context switching，或者系统启动</u>
- Limited concurrency or parallelism：编程语言本身支持的并发与内核需要的并发并不一致，比如我们在调度线程的时候，一个线程会将锁传递给另一个线程。一些并发管理模式在用户程序中不太常见，但是在内核中会出现。

![img](https://pic2.zhimg.com/80/v2-81f7a446f2d9a656ce119e7b53e6ecc1_1440w.webp)

​	今天论文的目标是能够测量出高级编程语言的优劣势，并从safety，programmability和性能损失角度，探索使用高级编程语言而不是C语言实现内核的效果。

​	当然，为了做到这一点，你需要在一个产品级的内核上做实验，而不是在XV6上。XV6现在是由C语言写的很慢的内核，如果你用Golang也写了个很慢的内核，这不能说明C还是Golang更快，这只能说明XV6很慢。所以，你会想要在一个为高性能而设计的内核中完成这里的测量。

​	很奇怪之前并没有一个论文完成了这里的测量。<u>有很多论文研究了在用户程序中高级编程语言的优劣势，但是你知道的，内核与用户程序还是很不一样的，比如内核中需要有更小心的内存管理，内核中的并发或许会略有不同</u>。所以，现在我们想要在内核中而不是用户程序中完成分析，而我们并没有找到之前的任何论文真正做了这个工作。

​	之前的确有很多内核是用高级编程语言写的，这里有很长的历史，甚至可以回溯到最早的计算机中。但是最近的一些基于高级编程语言的内核并不是为了评估High Level Language Tax，而是为了探索新的内核设计和新的内核架构，所以这些内核并没有在保持结构相同的同时，直接对比C语言内核。只有保持系统结构相同，你才可以真正的关注语言本身，而不是一些其他的问题。

![img](https://pic3.zhimg.com/80/v2-3c35d4a6e84590231423ba60bd5d7bd2_1440w.webp)

​	所以我们能做到的最好情况是：

- 用高级编程语言构建内核
- 保留与Linux中最重要的部分对等的功能
- 优化性能使得其与Linux基本接近，即使这里的功能与Linux并不完全一致，但是我们至少可以将它们拉到一个范围内
- 最后我们就可以测量高级编程语言的优劣

​	当然，这种方法的风险在于我们构建的内核与Linux还是略有不同，它不会与Linux完全一样，所以在得出结论时需要非常小心。这就是为什么不能对论文提出的问题（注，也就是应该使用什么样的编程语言实现操作系统）给出一个十分清晰的答案的原因。尽管如此，我们还是可以期望更深入的了解这个问题，而不是完全不知道它的内容。

​	以上就是论文的背景，以及为什么很少有人会做同样的工作的原因。

## 20.3 操作系统编程之go语言

> [20.3 高级编程语言选择 --- Golang - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367506458) <= 图文出处

![img](https://pic3.zhimg.com/80/v2-02e39014a833e11b7796c43891fd876a_1440w.webp)

​	接下来我们看一下对比方法，图中左边是Biscuit，这是一个我们为了[论文](https://pdos.csail.mit.edu/6.828/2020/readings/biscuit.pdf)专门用Golang写的内核，它以大概类似的方式提供了Linux中系统调用的子集。Biscuit和Linux的系统调用有相同的参数和相同的调用方式。并且我们在内核之上运行的是相同的应用程序，这里的应用程序是NGINX，这是一个web server，这里我们将相同的应用程序分别运行在Biscuit和Linux之上，应用程序会执行相同的系统调用，并传入完全相同的参数，Biscuit和Linux都会完成涉及系统调用的相同操作。之后，我们就可以研究高级编程语言内核和Linux之间的区别，并讨论优劣势是什么。以上就是对比方法的核心。

​	因为Linux和Biscuit并不完全一样，它们会有一些差异，所以我们花费了大量的时间来使得这里的对比尽可能的公平。

![img](https://pic3.zhimg.com/80/v2-d3b4a83a76d058a3538b3626b9f71372_1440w.webp)

​	有很多同学可能会问，这里会使用什么样的高级编程语言呢？基于以下原因，我们选用了Golang。

- 这是一个静态编译的编程语言，不像Python这里没有解释器。我们喜欢静态编译的语言的原因是编译语言性能通常更好，实际上Go编译器就非常好，所以基本上来说这是一种高性能编程语言

- 另外，Golang被设计成适合系统编程，而内核就是一种系统编程所以Golang也符合这里的场景。例如：

- - Golang非常容易调用汇编代码，或者其他的外部代码
  - Golang能很好的支持并发
  - Golang非常的灵活

- 另一个原因是Golang带有Garbage Collector。使用高级编程语言的一个优点就是你不需要管理内存，而GC是内存管理的核心。

​	在我们开始写论文的时候，Rust并不十分流行，并且也不是十分成熟和稳定。但是如果你现在再做相同的事情，你或许会想要用Rust来实现。因为Rust也是为系统编程而设计，它有一个小的运行时，它能生成好的代码。不过Rust相比Golang还有一个缺点，Rust认为高性能程序不能有GC，所以Rust不带GC。实际上Rust的类型系统以一种非常聪明且有趣的方式实现，所以GC对于Rust并不是必须的。这里涉及到一个有趣的问题：通过高级编程语言实现内核时，GC的代价到底有多少？而Rust通过不使用GC而跳过了这个问题。

​	这里有一个问题，并且在这节课最后我们会再次回顾这个问题。我们想要使用高级编程语言内核的部分原因是为了避免一类特定的Bug，那么你可以问自己的一个问题的是，你们在实验中遇到的Bug，是否可以通过使用高级编程语言来避免？我肯定你可以回想起一些Bug，它们耗费了你很多的时间，很多精力，现在你可以问自己，如果实验中的XV6是使用某种高级编程语言实现的，你的生活会不会更轻松一些？你是否能有更多时间做一些其他的事情。让我们记住这个问题，并在这节课结束的时候再看这个问题。

---

问题：如果我们这里使用Rust而不是Golang来实现高级编程语言内核，通过一定的优化有没有可能达到比C内核更高的性能？

回答：因为我们没有做过这样的实验，所以我就猜一下。我觉得不会有比C内核更高的性能，但是基本在同一个范围内。因为C是如此的底层，你可以假设你在Rust做的工作，都可以在C中完成。

## 20.4 Biscuit简述

> [20.4 Biscuit - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367506675) <= 图文出处

![img](https://pic4.zhimg.com/80/v2-71ed68f718c78859858244ff51ccf4f3_1440w.webp)

​	接下来我将对Biscuit稍作介绍，包括了Biscuit是如何工作的，以及在实现中遇到的问题。其中有些问题是预期内的，有些问题不在预期之内。

​	就像Linux和XV6一样，Biscuit是经典的**宏内核(monolithic kernel)**。所以它也有用户空间和内核空间，用户空间程序可能是你的编译器gcc，或者论文中主要用到的webserver。这里用户空间程序主要用C实现，尽管原则上它可以是任何编程语言实现的，但是因为这里只是性能测试，我们这里统一选用的是C版本的应用程序。大部分用户程序都是多线程的，所以不像在XV6中每个用户程序只有一个线程，<u>在Biscuit中支持用户空间的多线程</u>。<u>基本上，对于每个用户空间线程，都有一个对应的位于内核的内核线程，这些内核线程是用Golang实现的，在Golang里面被称为goroutine</u>。你可以认为goroutine就是普通的线程，就像XV6内核里的线程一样。区别在于，XV6中线程是由内核实现的，而这里的goroutine是由Go runtime提供。所以Go runtime调度了goroutine，Go runtime支持sleep/wakeup/conditional variable和同步机制以及许多其他特性，所以这些特性可以直接使用而不需要Biscuit再实现一遍。

​	<u>Biscuit中的Go runtime直接运行在硬件上，稍后我将介绍更多这部分内容，但是你现在可以认为当机器启动之后，就会启动Go runtime。这里会稍微复杂，因为Go runtime通常是作为用户空间程序运行在用户空间，并且依赖内核提供服务，比如说为自己的heap向内核申请内存。所以Biscuit提供了一个中间层，使得即使Go runtime运行在裸硬件机器之上，它也认为自己运行在操作系统之上，这样才能让Go runtime启动起来</u>。

​	Biscuit内核本身与XV6非常相似，除了它更加的复杂，性能更高。<u>它有虚拟内存系统可以实现mmap，有更高性能的文件系统，有一些设备驱动，比如磁盘驱动，以及网络协议栈</u>。所以Biscuit比XV6更加完整，它有58个系统调用，而XV6只有大概18-19个系统调用；它有28000行代码，而XV6我认为只有少于10000行代码。所以Biscuit有更多的功能。

> 学生提问：这里的接口与XV6类似对吧，所以进程需要存数据在寄存器中，进程也会调用ECALL。
>
> Frans教授：我稍后会再做介绍，但是这里完全相同。

![img](https://pic2.zhimg.com/80/v2-b7bc7d2487f14c37ddf24beafb0cfe45_1440w.webp)

​	以上是Biscuit的特性，有些我已经提到过了。

- 首先它支持多核CPU。Golang对于并发有很好的支持，所以Biscuit也支持多核CPU。类似的，XV6却只对多核CPU有有限的支持。所以在这里，我们相比XV6有更好的同步协调机制。
- 它支持用户空间多线程，而XV6并没有。
- 它有一个相比XV6更高性能的Journaled File System（注，Journaled就是指log，可以实现Crash Recovery）。如果你还记得EXT3论文，它与EXT3的Journaled File System有点类似。
- 它有在合理范围内较为复杂的虚拟内存系统，使用了VMAs并且可以支持mmap和各种功能
- 它有一个完整的TCP/IP栈，可以与其他的服务器通过互联网连接在一起
- 它还有两个高性能的驱动，一个是Intel的10Gb网卡，以及一个非常复杂的磁盘驱动AHCI，这比virtIO磁盘驱动要复杂的多

![img](https://pic1.zhimg.com/80/v2-c36139d945e8fa2609bd87cffead0a4c_1440w.webp)

​	Biscuit支持的用户程序中：

- 每个用户程序都有属于自己的Page Table
- 用户空间和内核空间内存是由硬件隔离的，也就是通过PTE的User/Kernel bit来区分
- <u>每个用户线程都有一个对应的内核线程，这样当用户线程执行系统调用时，程序会在对应的内核线程上运行。如果系统调用阻塞了，那么同一个用户地址空间的另一个线程会被内核调度起来</u>
- 如之前提到的，内核线程是由Go runtime提供的goroutine实现的。如果你曾经用Golang写过用户空间程序，其中你使用go关键字创建了一个goroutine，这个goroutine就是Biscuit内核用来实现内核线程的goroutine。

![img](https://pic3.zhimg.com/80/v2-68ce3da201fee1197be8018cbf642672_1440w.webp)

​	来看一下系统调用。就像刚刚的问题一样，这里的系统调用工作方式与XV6基本一致：

- 用户线程将参数保存在寄存器中，通过一些小的库函数来使用系统调用接口
- 之后用户线程执行SYSENTER。现在Biscuit运行在x86而不是RISC处理器上，所以进入到系统内核的指令与RISC-V上略有不同
- 但是基本与RISC-V类似，控制权现在传给了内核线程
- 最后内核线程执行系统调用，并通过SYSEXIT返回到用户空间

​	所以这里基本与XV6一致，这里也会构建trapframe和其他所有的内容。

> 学生提问：我认为Golang更希望你使用channel而不是锁，所以这里在实现的时候会通过channel取代之前需要锁的场景吗？
>
> Frans教授：这是个好问题，我会稍后看这个问题，接下来我们有几页PPT会介绍我们在Biscuit中使用了Golang的什么特性，但是我们并没有使用太多的channel，大部分时候我们用的就是锁和conditional variable。所以某种程度上来说Biscuit与XV6的代码很像，而并没有使用channel。<u>我们在文件系统中尝试过使用channel，但是结果并不好，相应的性能很差，所以我们切换回与XV6或者Linux类似的同步机制</u>。

![img](https://pic3.zhimg.com/80/v2-6c4cdd1fb7ffd1cedda7515bff4ef85e_1440w.webp)

​	在实现Biscuit的时候有一些挑战：

- 首先，我们**需要让Go runtime运行在裸硬件机器之上**。我们希望对于runtime不做任何修改或者尽可能少的修改，这样当Go发布了新的runtime，我们就可以直接使用。在我们开发Biscuit这几年，我们升级了Go runtime好几次，所以Go runtime直接运行在裸硬件机器之上是件好事。并且实际上也没有非常困难。<u>Golang的设计都非常小心的不去依赖操作系统，因为Golang想要运行在多个操作系统之上，所以它并没有依赖太多的操作系统特性，我们只需要仿真所需要的特性。大部分这里的特性是为了让Go runtime能够运行起来，一旦启动之后，就不太需要这些特性了</u>。
- 我们需要安排goroutine去运行不同的应用程序。通常在Go程序中，只有一个应用程序，而这里我们要用goroutine去运行不同的用户应用程序，这些不同的用户应用程序需要使用不同的Page Table。**这里困难的点在于，Biscuit并不控制调度器，因为我们使用的是未经修改过的Go runtime，我们使用的是Go runtime调度器，所以在调度器中我们没法切换Page Table**。Biscuit采用与XV6类似的方式，它会在内核空间和用户空间之间切换时更新Page Table。所以当进入和退出内核时，我们会切换Page Table。这意味着像XV6一样，当你需要在用户空间和内核空间之间拷贝数据时，你需要使用copy-in和copy-out函数，这个函数在XV6中也有，它们基本上就是通过软件完成Page Table的翻译工作。
- **另一个挑战就是设备驱动，Golang通常运行在用户空间，所以它并不能从硬件收到中断**。但是现在我们在裸硬件机器上使用它，所以它现在会收到中断，比如说定时器中断，网卡中断，磁盘驱动中断等等，我们需要处理这些中断。<u>然而在Golang里面并没有一个概念说是在持有锁的时候关闭中断，因为中断并不会出现在应用程序中，所以我们在实现设备驱动的时候要稍微小心</u>。我们采取的措施是在设备驱动中不做任何事情，我们不会考虑锁，我们不会分配任何内存，我们唯一做的事情是向一个非中断程序发送一个标志，之后唤醒一个goroutine来处理中断。在那个goroutine中，你可以使用各种各样想要的Golang特性，因为它并没有运行在中断的context中，它只是运行在一个普通goroutine的context中。
- 前三个挑战我们完全预料到了，我们知道在创造Biscuit的时候需要处理它们，而最难的一个挑战却不在我们的预料之中。这就是**heap耗尽的问题**。所以接下来我将讨论一下heap耗尽问题，它是什么，它怎么发生的，以及我们怎么解决的？

## 20.5 Biscuit堆内存耗尽问题(Heap exhaustion)

> [20.5 Heap exhaustion - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367506795) <= 图文出处

​	让我们讨论一下Heap耗尽问题，我不会像[论文](https://pdos.csail.mit.edu/6.828/2020/readings/biscuit.pdf)一样深入讨论，但是至少会演示问题是什么。

![img](https://pic4.zhimg.com/80/v2-da8954f9cc1431709de0e49e5615161f_1440w.webp)

​	假设蓝色的矩形是内核，内核会有一个heap，它会从其中申请动态内存。在XV6中，我们并没有这样一个heap，我们在内核中没有内存分配器，所有内存都是静态分配的。**但是任何其他的内核中，都会有heap，所以你在内核中可以调用malloc和free。可能通过heap分配的对象有socket对象，文件描述符对象和进程对象**。所以，我们在XV6中静态分配的所有结构体，例如struct proc，struct fd，<u>在正常的内核中都是动态分配的。所以当你打开一个新的文件描述符时，内核会通过heap分配一个文件描述符对象</u>。

​	这里的问题是，你可以运行很多个应用程序，它们会打开很多个文件描述符，拥有很多个socket，它们会逐渐填满heap。

![img](https://pic3.zhimg.com/80/v2-e3836171c6f2c85a4c7731e70dea93b2_1440w.webp)

​	在某个时间点，heap会被填满，这时没有额外的空间可以用来分配一个新的对象。如果这时应用程序需要打开一个新的文件描述符，或者调用了fork使得内核想要在heap中分配一个新的proc结构体，heap中没有了空间。这时你该怎么办呢？这是一个不太常见的常见问题，但是如果你使劲用你的电脑，你或许会遇到所有内存都被使用了的情况，你的heap满了，并且没有进程调用free，因为它们都还在运行且想分配更多的内存。所有的内核都会遇到这个问题，不管是C内核也好，Biscuit也好，任何内核都需要解决这个问题。

​	之所以这个问题对于我们来说是个严重的问题，是因为**在很多内核中，你可以对malloc返回错误**，实际上，XV6就是这么做的。<u>但是在Go runtime中，当你调用new来分配一个Go对象，并没有error condition，new总是可以成功。让我们来讨论一些可以解决这里问题的方法</u>。

![img](https://pic2.zhimg.com/80/v2-ebf588c79869dd768ee1382b35f34c5d_1440w.webp)

- 第一种方法我们在XV6中见过。如果XV6不能找到一个空闲的block cache来保存disk block，它会**直接panic**。这明显不是一个理想的解决方案。这并不是一个实际的解决方案，所以我们称之为strawman。
- 另一个strawman方法是，当你在申请一块新的内存时，你会调用alloc或者new来分配内存，你实际上可以在内存分配器中进行等待。这实际上也不是一个好的方案，原因是你**可能会有死锁**。假设内核有把大锁，当你调用malloc，因为没有空闲内存你会在内存分配器中等待，那么这时其他进程都不能运行了。因为当下一个进程想要释放一些内存时，但是因为死锁也不能释放。对于内核中有大锁的情况，这里明显有问题，但是即使你的锁很小，也很容易陷入到这种情况：**在内存分配器中等待的进程持有了其他进程需要释放内存的锁，这就会导致死锁的问题**。
- 下一个strawman方法是，如果没有内存了就返回空指针，你检查如果是空指针就直接失败，这被称为bail out。但是**bail out并不是那么直观，进程或许已经申请了一些内存，那么你需要删除它们，你或许做了一部分磁盘操作，比如说你在一个多步的文件系统操作中间，你只做了其中的一部分，你需要回退。所以实际中非常难做对**。

​	当研究这部分，并尝试解决这个问题，**Linux使用了前面两种方法，但是两种方法都有问题。实际中，内核开发人员很难将这里弄清楚**。如果你对这个问题和相关的讨论感兴趣，可以Google搜索“[too small to fail](https://lwn.net/Articles/627419/)”，会有一篇小的文章讨论释放内存，在内存分配器中等待的复杂性。

​	对于Biscuit来说，strawman 2解决方案不可能实施，因为new不会fail，它总是能成功。除此之外，这里的方案都不理想，所以我们需要有一种更好的方法。

## 20.6 Biscuit堆内存耗尽解决方案

> [20.6 Heap exhaustion solution - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367507100) <= 图文出处

![img](https://pic2.zhimg.com/80/v2-d84dad234bd58a5c8c35786691a75b49_1440w.webp)

​	**Biscuit的解决方案非常直观，当应用程序执行系统调用，例如read，fork时，在系统调用的最开始，跳转到内核之前，它会先调用reserve函数，reserve函数会保留足够的内存以运行系统调用**。所以reserve会保留足够这个系统调用使用的空闲内存，以使得系统调用总是能成功。所以一旦系统调用被执行，且保留了足够的内存，那么它就可以一直运行而不会有内存耗尽和heap exhaustion的问题。

​	**如果reserve函数执行时没有足够的内存，那么程序会在这里wait等待**。<u>因为现在在系统调用的最开始，系统调用现在还没有持有任何的锁，也没有持有任何的资源，所以在这里等待完全没有问题，也不会有死锁的风险</u>。**当程序在等待的时候，内核可以撤回cache并尝试在heap增加空闲空间，比如说kill一个进程来迫使释放一些内存。一旦内存够用了，并且内核决定说是可以满足需要保留的内存，之后内核会让系统调用继续运行，然后执行系统调用需要的操作**。

​	**在最后，当系统调用完成的时候，所有之前保留的内存都返回到池子中，这样后续的系统调用可以继续使用**。

​	这个方案中有一些很好的特性：

- 在内核中没有检查。你不需要检查内存分配是否会失败，在我们的例子中这尤其的好，因为在Golang中内存分配不可能会失败。
- 这里没有error handling代码。
- **这里没有死锁的可能，因为你在最开始还没有持有锁的时候，就避免了程序继续执行**。

​	当然，现在的问题是**如何实现reserve函数，你如何计算运行一个系统调用会需要多少内存**？

![img](https://pic3.zhimg.com/80/v2-d7425fd7bae2850797d2ebb3fe745422_1440w.webp)

​	你保留的内存数量是重要的，你可以为每个系统调用保留一半的内存或者一些其他夸张的内存数量。但是这意味着你限制了可以并发执行的系统调用的个数，所以你这里**尽量精确的计算一个系统调用的内存边界**。

![img](https://pic4.zhimg.com/80/v2-18557d2f0a96944d11f618c6c727f72b_1440w.webp)

​	这里的解决方法是使用了高级编程语言的特性。<u>Golang实际上非常容易做静态分析，Go runtime和Go生态里面有很多包可以用来分析代码，我们使用这些包来计算系统调用所需要的内存</u>。所以你可以想象，如果你有一个read系统调用，我们可以通过系统调用的函数调用图查看比如函数f调用函数g调用函数h等等等等。我们可以做的是弄清楚这里调用的最大深度，对于最大的深度，计算这里每个函数需要的内存是多少。

![img](https://pic1.zhimg.com/80/v2-739c2fdf559ee0e67bf708dde7584050_1440w.webp)

​	比如说函数f调用了new，因为这是一个高级编程语言，我们知道new的对象类型，所以我们可以计算对象的大小。我们将所有的new所需要的内存加起来，得到了一个总和S，这就是这个调用图（或者说系统调用）任何时间可能需要的最大内存。

![img](https://pic4.zhimg.com/80/v2-1d43c0c9c05f909cae61b1d13ac0c983_1440w.webp)

​	**实际中并没有这么简单，会有点棘手。因为函数h可能会申请了一些内存，然后再回传给函数g。所以当h返回时，g会得到h申请的一些内存。这被称为escaping，内存从h函数escape到了函数g**。

![img](https://pic1.zhimg.com/80/v2-be9fa537563131209750d6b26030c0e0_1440w.webp)

​	<u>存在一些标准算法来完成这里的escape分析，以决定哪些变量escape到了函数调用者。当发生escape时，任何由函数h申请的内存并且还在函数g中存活，我们需要将它加到函数g的内存计数中，最后加到S中</u>。

> 学生提问：**某些函数会根据不同的工作负载申请不同量的内存，那么在计算函数消耗的内存时，会计算最差的情况吗？**
>
> Frans教授：**是的。这里的工具会计算最深函数调用时最大可能使用的内存量。所以它会计算出每个系统调用可能使用的最多内存，虽然实际中系统调用可能只会使用少的多的内存。但是保险起见，我们会为最坏情况做准备**。一些系统调用内的for循环依赖于传给系统调用的参数，所以你不能静态的分析出内存边界是什么。所以在一些场景下，我们会标注代码并规定好这是这个循环最大循环次数，并根据这个数字计算内存总量S。类似的，如果有你有递归调用的函数，谁知道会递归多少次呢？或许也取决于一个动态变量或者系统调用的参数。实际中，我们在Biscuit中做了特殊处理以避免递归函数调用。所以最后，我们才可能完成这里的内存分析。

​	所以，这里的内存分析不是没有代价的，也不是完全自动的。这花费了Cody（论文一作）好几天检查代码，检查所有的循环并标注代码。还有一些其他的Golang特有的问题需要处理，例如，向Slice添加元素可能会使内存使用量扩大一倍，所以我们也给Slice标注了最大的容量。但是所有这些工作都是可完成的，在花费了几天时间之后，使用这里的内存分析工具，你可以得到对于系统调用使用的最大内存量的合理评估。以上基本就是Biscuit如何解决heap exhaustion问题。

> 学生提问：这里的静态内存分析工具，如果不是用来构建内核，它们通常会用来干嘛？
>
> Frans教授：Go编译器内部使用它来完成各种各样的优化，并分析得出最优的编译方式。这里正好编译器使用了一个包，我们也可以使用同样的包。在后面你还可以看到，我们还将它用于一些其他特性，有这么一个包非常的方便。

## 20.7 Evaluation: HLL benefits

> [20.7 Evaluation: HLL benefits - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367507447) <= 图文出处
>
> [Linux 内核—— RCU机制介绍 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/517618594) <= 不是很好理解的文章
>
> [深入理解RCU|核心原理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/386422612) <= 特别好理解的文章，推荐阅读
>
> **RCU(Read-Copy Update)，**顾名思义就是读-拷贝修改，它是基于其原理命名的。对于被RCU保护的共享数据结构，读者不需要获得任何锁就可以访问它，但写者在访问它时首先拷贝一个副本，然后对副本进行修改，最后使用一个回调（callback）机制在适当的时机把指向原来数据的指针替换为新的被修改的数据。这个时机就是所有引用该数据的CPU都退出对共享数据的访问。
>
> [openVswitch（OVS）源代码之linux RCU锁机制分析_庾志辉的博客-CSDN博客](https://blog.csdn.net/yuzhihui_no1/article/details/40115559) <= 推荐阅读，对相关RCU的方法讲解通俗易懂

![img](https://pic4.zhimg.com/80/v2-a38629ece135b7cc78e573a207c8e8e7_1440w.webp)

​	Biscuit的实现与其他内核，例如XV6，非常相似，除了Biscuit比XV6性能要高的多。Biscuit采用了很多Linux内核的优化和聪明的设计：

- 我们对于内核文本采用了大页，以避免TLB的代价
- 我们有针对每个CPU的网卡队列，这样可以避免CPU核之间同步
- 我们有RCU实现了不需要读锁的Directory Cache
- ……

​	<u>通常为了高性能而做的优化，编程语言并不会成为阻碍</u>。Golang并没有成为阻碍这些优化实现的因素。这些优化之前是在C和Linux中实现，我们现在只是在Golang中又实现它们。在实现这些优化时有很多的工作，但是这些工作与编程语言本身无关。

![img](https://pic2.zhimg.com/80/v2-059c475cfe2e8ffea9e0f02324faafc1_1440w.webp)

​	今天[论文](https://pdos.csail.mit.edu/6.828/2020/readings/biscuit.pdf)的出发点就是**了解用高级编程语言实现操作系统的收益和代价**。所以我们将分两部分来评估，首先是收益，其次是代价。

![img](https://pic1.zhimg.com/80/v2-01cd438759314f038ba60c1d58a17e38_1440w.webp)

​	有关高级编程语言，我们要回答三个问题：

- 首先，我们有没有作弊？或许我们避免使用了所有Golang提供的高级编程语言中代价较高的功能
- 其次，高级编程语言是否有简化Biscuit代码？
- 最后，高级编程语言是否能阻止前面提到的内核漏洞？

![img](https://pic4.zhimg.com/80/v2-7b0237b53247c7eb8b37e8c057d45903_1440w.webp)

​	首先，我们有没有使用高级编程语言的特性？我们会对比一下Biscuit与其他两个大的Golang项目在使用语言特性上是否类似，这样我们才可使说我们的内核以类似的方式利用了相同的语言特性。这里我们使用了相同的静态分析工具来分析两个大的Golang项目，它们都有超过100万行代码，其中一个项目是Go runtime以及包含的所有包，另一个是一个叫做Moby的系统。

![img](https://pic1.zhimg.com/80/v2-24a68f4866f8484cfb0e427a3165a3ac_1440w.webp)

​	之后我们画出了一些高级语言特性在每1000行代码中的使用量。图中X轴是语言特性：

- allocation对应于new
- maps就是hashtable
- slice是动态数组
- channel是同步的工具，如你所见我们用的很少，Go runtine和Moby也用的很少
- 很明显我们最喜欢的特性就是函数返回多个值
- 我们使用了Closure（闭包）
- 我们稍微使用了defer
- 我们使用了Interface
- 使用了Type assertion来以一种类型安全的方式将一个类型转换成另一个类型
- 同时我们也import了很多包，Biscuit内核是由很多个包构建出来的，而不是一个大的单一的程序

​	如你所见，有些特性Biscuit用的比Go runtime和moby更少，有些特性Biscuit用的更多，这里没有很明显的区别。所以<u>从这张图中可以得出的主要结论是：Biscuit使用了Golang提供的高级编程语言特性，而不是为了得到好的性能而避开使用它们</u>。

> 学生提问：你这里是怎么统计的？是不是使用了静态分析工具？
>
> Frans教授：是的，这里使用的就是静态分析工具。通过写一个小程序利用静态分析工具来查看这些项目的每一行代码，并记录对应的特性是什么，这样就能统计这些特性的使用数量。

![img](https://pic2.zhimg.com/80/v2-02aa868b14bb2ebe7983d85b33423ded_1440w.webp)

​	第二个问题有点主观，高级编程语言有没有简化Biscuit代码？笼统地说我认为有的，我这里会讨论一两个例子。

​	使用Garbage allocation是极好的，你可以回想XV6，当你调用exit时，有大量的数据结构需要被释放回给内核，这样后面的进程才能使用。如果使用Garbage Collector这里的工作着实容易，Garbage Collector会完成这里的所有工作，你基本不用做任何事情。如果你从地址空间申请了一段内存，对应这段内存的VMA会自动被GC释放，所以这里可以简化代码。

​	如之前所说的，函数返回多个值对于代码风格很好。闭包很好，map也很好。XV6中很多地方通过线性扫描查找数据，但是如果你有map和hashtable作为可以直接使用的对象，那么你就不用线性扫描了。你可以直接使用map，runtime会高效的为你实现相应的功能。所以直观上的感受是，你可以得到更简单的代码。

![img](https://pic3.zhimg.com/80/v2-74ce208e84c3a6d1a989cd3fef549366_1440w.webp)

​	但是前面只是定性的评估，下面会介绍一些更具体的例子。当有大量的并发线程，且线程有共享的数据时，GC如何起作用的。

![img](https://pic3.zhimg.com/80/v2-938668abe2334f81c3d23b9f0864cf36_1440w.webp)

​	这里有个最简单的例子。假设你申请了一些动态的对象，比如说buffer，你fork一个线程来处理这个buffer，原线程也会处理同一个buffer。当两个线程都完成了工作，buffer需要被释放，这样内存才可以被后面的内核代码使用。这在C语言里面有点难协调，因为你需要有某种方式来决定buffer不再被使用。如果你使用GC，那么就没什么好决定的，因为当两个线程都处理完buffer之后，没有线程会指向那个buffer。GC会从线程栈开始追踪，并且在任何线程栈中都找不到buffer，因此GC会在稍后某个时间释放内存。所以在一个带GC的编程语言中，你完全不必考虑这个问题。

​	<u>在C中你可以这样解决这个问题，为对象增加引用计数，引用计数需要被锁或者一些原子性操作保护，当引用计数到达0时，你可以释放内存</u>。

![img](https://pic2.zhimg.com/80/v2-5ca17d00d6f79e1ac106e05486037c19_1440w.webp)

​	**实际中锁加上引用计数代价稍微有点高**。如果你想要高性能，并且并发可以扩展到CPU核数，这可能会是个瓶颈，我们在后面介绍RCU的时候会看这部分。所以**，如果你想要高性能，好的并发能力，人们倾向于不给读数据加锁**。

![img](https://pic3.zhimg.com/80/v2-df0d3cf086aec4e3afa07ad102be2b36_1440w.webp)

​	在实际中，我们会使得读数据至少是不需要锁的，这样你就不需要付出额外的代价。上面是我们在Golang中的实现，我们有个get函数，它会读取并返回链表的头结点。这里就没有使用锁，而是使用了`atomic_load`，它会读取头结点，但是又不需要锁。后面的pop函数使用了锁。这种风格在Linux内核中非常常见，写数据需要加锁，读数据不用加锁。这里pop函数会从链表中弹出头结点，这样你就可以重用头结点对应的内存。<u>在C中实现这种风格会有点困难，因为有可能当你释放头结点内存时，其他并发的线程正好读取到了头结点的指针。这样当你做完`atomic_store`，你不能释放指针内容，因为有可能有另一个线程的指针指向了这部分内容。如果你在这里释放了指针内容，你有可能会有use-after-free Bug</u>。

![img](https://pic4.zhimg.com/80/v2-1b3636cb0dc09c95c97ae00aebae8f77_1440w.webp)

​	我们在这门课程的最后一节课会看到，Linux内核对这个问题有一种非常聪明的解决办法，被称为**Read-Copy-Update或者是RCU**。<u>它的工作就是**推迟释放内存**，直到确定指针不再被使用，并且它有一种非常聪明的方案来决定什么时候可以安全释放内存。但是这个方案有各种各样的限制，程序员需要在RCU关键区域内遵守各种规则</u>。比如说你不能在RCU关键区域sleep，也不能切换线程。

​	所以**尽管实际中Linux内核非常成功的使用了RCU，但是RCU还是有点容易出错，并且需要小心编程来使得它能正确工作**。<u>在带有GC的编程语言，例如Golang，这就不是问题了，因为GC会决定某个对象不再被使用，只有这时才释放它。所以现在对于编程人员来说没有限制了，所有的限制都被GC考虑了。这是一种带有GC的编程语言的明显优势</u>。

![img](https://pic4.zhimg.com/80/v2-5074fe8ee0c92993d3eb8f5e42a9671b_1440w.webp)

​	接下来看看CVEs Bugs，这在前面提到过（注，20.1）。

![img](https://pic3.zhimg.com/80/v2-3954f02dfb3f710c7acb036b74975842_1440w.webp)

​	我们手动的检查了所有的CVEs Bug，并尝试确定Golang是否修复了问题。

- 第一行代表我们不能弄清楚这些Bug的结果是什么，它会怎么展现，我们知道如何修复这些问题，但是我们不能确定Golang是否能避免这些问题。
- 有很多逻辑Bug，可以认为Golang会有与C相同的Bug，所以结果是相同的
- 接下来是40个memory-safety Bugs，包括了use-after-free，double-free，out-of-bound。其中8个直接消失了，因为GC考虑了内存释放，32个会产生panic，比如说数组越界。当然panic并不好，因为内核会崩溃，但是或许要比直接的安全漏洞更好。所以在这40个Bug中，高级编程语言有帮到我们。

​	以上就是使用高级编程语言实现内核的优势，接下来讨论一些代价，也就是High Level Language Tax。

## 20.8 Evaluation: HLL performance cost

> [20.8 Evaluation: HLL performance cost(1) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367507690)
>
> [20.9 Evaluation: HLL performance cost(2) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367507950)
>
> 图文出至以上两篇博文
>
> ---
>
> [SLA服务级别协议_百度百科 (baidu.com)](https://baike.baidu.com/item/服务级别协议/10967493?fromtitle=SLA&fromid=2957862&fr=aladdin)

![img](https://pic4.zhimg.com/80/v2-1dc3e30dbb039cb6fda8cc26678732af_1440w.webp)

​	以上是6个问题，我应该不会全部介绍，因为我想在课程的最后留些时间来回顾我们在本节课开始时提出的问题。

![img](https://pic4.zhimg.com/80/v2-292ec622d220549c44d9004423eb6573_1440w.webp)

​	以上就是测试环境，Biscuit运行在裸硬件机器之上，所以我们的测试是在物理服务器而不是QEMU之上。我们使用了三个应用程序来做性能测试，它们分别是，Webserver，K/V store，Mail server benchmark。

![img](https://pic2.zhimg.com/80/v2-6bebeeac0292855fb7b099e95f0948fd_1440w.webp)

​	这三个应用程序都会给内核一定的压力，它们会执行系统调用，内核会做大量的工作。你可以看到，大部分CPU时间都在内核中。

![img](https://pic3.zhimg.com/80/v2-4b7ad172c0d66e373e0e9c5c9b32a942_1440w.webp)

​	首先的问题是，Biscuit是否是一个工业质量的内核？我们将前面的三个应用程序分别运行在Linux和Biscuit上，并进行对比。

![img](https://pic4.zhimg.com/80/v2-c1b1d7fee6953ee4a4f5d1b0356b4a4b_1440w.webp)

​	在Linux中，我们会关闭所有Biscuit不提供的功能，比如Page Table隔离，repoline等等很多功能，这样的对比才会尽可能的公平。有些特性会比较难关闭，但是我们会尽量将它们关闭。

![img](https://pic3.zhimg.com/80/v2-adb3ea1172df7d25ce6d6ecf1971790e_1440w.webp)

​	之后我们会测试吞吐量，如你所见Biscuit总是会比Linux更慢，mailbench可能差10%，nginx和redis差10%到15%。这里的数字并不是绝对的，因为两个系统并不完全一样。但是可以看出两个系统基本在同一个范围内，而不是差个2倍或者10倍。

![img](https://pic1.zhimg.com/80/v2-a067e50a25fffeec1a57911ef2e55998_1440w.webp)

​	接下来我们会分析代码，并找到高级编程语言额外的CPU cycle消耗。我们会找到：

- 哪些CPU cycle是GC使用的
- 哪些是函数调用的Prologue使用的。Golang会为函数调用做一些额外的工作来确保Stack足够大，这样就不会遇到Out-of-Stack的问题
- Write barrier是GC用来跟踪不同空间的指针的方法
- Safety cycles是用在数组边界检查，空指针检查上的CPU cycles

![img](https://pic3.zhimg.com/80/v2-aa74b65a29a306a4a8ce4e41e29459aa_1440w.webp)

​	通过测试上面的应用程序，可以得到测量结果。

- 3%的执行时间用在了GC cycles中，这里我稍后会介绍为什么这很少。同时这也可以说明GC是在运行的，我们并不是用了一大块内存而没有使用GC
- 令人奇怪的是，Prologue占有的CPU时间最多，这基本上跟我们用来检查kernel Stack或者goroutine Stack是否需要增加的方案有关，这里或许更容易降低一些
- Write barrier使用的时间很少
- 2%-3%的CPU时间用在了Safety cycles中

​	这些数据都很好，**High Level Language Tax并不是那么的大**。

![img](https://pic1.zhimg.com/80/v2-eab85047b6ba7d1ae46bd7465d11edf4_1440w.webp)

​	**当然GC的占比可能会更高，因为它完全取决于heap大小和存活对象的数量，GC会跟踪所有的存活对象，并决定哪些对象已经不被使用。如果有大量的存活对象，GC也需要跟踪更多的对象。所以这里的CPU时间完全与存活对象的数量相关**。

![img](https://pic2.zhimg.com/80/v2-81632158b2301c794a4804ba6c873385_1440w.webp)

​	所以我们做了一些其他的实验。我们创建了大量的存活对象，大概有200万个vnode，可以认为这是200万个inode。然后修改heap的headroom，也就是GC可以使用的空闲内存数量，最后再测量GC的代价。

![img](https://pic3.zhimg.com/80/v2-4324a40ae6e0dd7fa494679977a3deaa_1440w.webp)

​	上图就是测量结果，存活对象占了640MB内存，我们在不同内存大小下运行测试。第一次测试时，有320MB空闲内存，是存活对象内存的一半，这时Golang有非常严重的overhead，大概是34%，GC因为没有足够的headroom需要运行很多额外的程序。如果空闲内存是存活对象的2倍，那么GC的overhead就没有那么疯狂，只有9%。所以，为了保持GC的overhead在10%以内，物理内存大小需要是heap大小的三倍。

> 学生提问：什么是write barrier？是设置权限吗？
>
> Frans教授：你还记得Lec17的内容吗？当GC在运行的时候，需要检查指针是否在from空间，如果在from空间你需要拷贝它到to空间。write barrier是非常类似的功能，它的想法是一样的，你需要检查指针看它是否在你需要运行GC的区域内。
>
> 学生提问：当存活对象的内存大于空闲内存的时候，GC该怎么工作呢？
>
> Frans教授：你买一些内存，vnode会使用一些内存，然后还剩下320MB空闲内存。当应用程序申请更多内存时，首先会从空闲内存中申请，直到空闲内存也用光了。与此同时，GC也在运行。所以我们刚刚的测试中是在3个不同配置下运行，在最后一个配置中，空闲内存是存活对象占用内存的两倍。这意味着GC有大量的headroom来与应用程序并行的运行，如果有大量的headroom，GC的overhead就没那么高了，只有10%左右，而不是34%。在第一个配置中，总共是640+320MB内存，而不是只有320MB内存。

![img](https://pic4.zhimg.com/80/v2-e4859ee129bb2878ef9f146ab6defb33_1440w.webp)

​	这一页跳过没讲。

![img](https://pic2.zhimg.com/80/v2-f58e1bbc588755790b29183affd48999_1440w.webp)

​	接下来稍微介绍GC pause。Go的GC是一个带有短暂pause的并发GC，它在一段非常短的时间内停止程序运行来执行write barrier，之后再恢复应用程序的运行，同时GC也会完成自己的工作。<u>Go的GC也是递增的，就像我们在Lec17中介绍过的一样，每次调用new都会做一些GC的工作。所以每次GC做一些工作的时候，应用程序都会有一些延时，这就是代价</u>。

![img](https://pic4.zhimg.com/80/v2-e5896defa5fda9c023da4e98d20022eb_1440w.webp)

​	所以我们做了一些测试，我们找了个应用程序并测试了最大的pause时间。也就是由于GC可能导致应用程序最大的停止时间。

![img](https://pic3.zhimg.com/80/v2-0b027a44aa21d753c15f870800bdd466_1440w.webp)

​	最大的单个pause时间是115微秒，也就是在web server中，因为使用了TCP stack，TCP Connection table中很大一部分需要被标记（注，GC的一部分工作是标记对象），这花费了115微秒。一个HTTP请求最大的pause时间是582微秒，所以当一个请求走到一个机器，最多会有总共582微秒延时来执行这个请求。而超过100微秒的pause发生的非常非常少，只有少于0.3%。

![img](https://pic3.zhimg.com/80/v2-61c4db767ee2895c98dd3626a10cbf5a_1440w.webp)

​	如果你尝试达成某种SLA，其中要求的最长请求处理时间很短，那么582微秒就很严重。但是如果你查看Google论文，[The Tail at Scale](https://research.google/pubs/pub40801/)，其中介绍有关一个请求最长可以有多长处理时间，他们讨论的都是几毫秒或者几十毫秒这个量级。所以Biscuit拥有最大pause时间是582微秒还在预算之内，虽然不理想，但是也不会很夸张。这表明了，Golang的设计人员把GC实现的太好了。并且我们在做Biscuit项目的时候发现，每次我们升级Go runtime，新的runtime都会带一个更好的GC，相应的GC pause时间也会变得更小。

![img](https://pic1.zhimg.com/80/v2-27c0b8b02e0ae0b637d8dd892da86bfc_1440w.webp)

​	之前在Linux和Biscuit之间的对比并不真正的公平，因为Biscuit和Linux实现的是不同的功能。所以我们做了一个额外的测试，我们写了两个完全相同的内核，一个用C实现，另一个用Golang实现。这两个内核实现了完全相同的东西，并且我们会查看汇编代码以检查区别在哪。可能会有一些区别，因为Golang会做一些安全检查，但是对于基本功能来说，汇编代码是一样的。

![img](https://pic2.zhimg.com/80/v2-87805ed3c2b5dc48d39d0988c4934849_1440w.webp)

​	以上是有关测试的一部分，通过pipe来回传输一个字节。我们查看内核中有关将一个字节从pipe的一端传到另一端的代码。Go里面是1.2K行代码，C里面是1.8K行代码。这里没有内存分配和GC，所以这里只有语言上的差异。我们还查看了两种实现语言中花费最多时间的10个地方，这样我们才能确保两种语言实现的代码尽可能的接近。

![img](https://pic4.zhimg.com/80/v2-5aa711d4807507a052795d3fd913a907_1440w.webp)

​	之后我们查看了每秒可以完成的操作数，如你可见Golang要慢15%。如果你查看Golang的Prologue和safety-check，这些指令是C代码所没有的，这些指令占了16%，这与更慢的处理速度匹配的上。**所以这里的主要结论是Golang是更慢，但并不是非常夸张的慢，Golang还是非常有竞争力的。并且这与我们早些时候做的Biscuit和Linux对比结果一致**。

## 20.9 HLL用于编程新OS是否合适

> [20.10 Should one use HLL for a new kernel? - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/367508066) <= 图文出处
>
> [go runtime 简析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/111370792)

​	最后我想讨论我们在最开始问过的一个问题，你应该在一个新内核中使用高级编程语言吗?

![img](https://pic1.zhimg.com/80/v2-7ea6126b32ad1eff44885d1cc37c6144_1440w.webp)

​	与其直接回答这个问题，我在这页有一些我们的结论和一些考虑。或许你们该回退一步，并问自己，你们更喜欢哪种方式？你们是喜欢像在实验中用C写XV6，还是喜欢使用类似Golang的高级编程语言。更具体的说，你们更想避免哪类Bug？或许在这节课的过程中想想你们遇到过什么Bug？我想听听你们的体验，你们是怎么想的？切换到高级编程语言会不会改变你们的体验？

> 一些学生介绍自己的体验，有说C好的，有说C不好的，略过。

​	当然，我们不会将XV6改成Golang或者任何高级编程语言。具体原因刚刚一些同学已经提到了，Golang还是隐藏了太多细节，这门课的意义在于理解系统调用接口到CPU之间的所有内容。举个例子，Golang隐藏了线程，我们并不想隐藏线程，我们想要向你解释线程是如何实现的。所以接下几年，这门课程还是会使用C语言。

​	但是如果你要实现一个新的内核，并且目标不是教育你的学生有关内核的知识，目标是写一个安全的高性能内核。你可以从我们的研究中得出一些结论：

- 如果性能真的至关重要，比如说你不能牺牲15%的性能，那么你应该使用C。
- 如果你想最小化内存使用，你也应该使用C。
- 如果安全更加重要，那么应该选择高级编程语言。
- 或许在很多场景下，性能不是那么重要，那么使用高级编程语言实现内核是非常合理的选择。

​	Cody、Robert和我在实现这个项目的过程中学到的一件事情是，任何一种编程语言就是编程语言，你可以用它来实现内核，实现应用程序，它并不会阻止你做什么事情。

---

问题：我很好奇你们是怎么实现的Biscuit，你们直接在硬件上运行的Go runtime，具体是怎么启动的？

回答：这里有一层中间层设置好了足够的硬件资源，这样当Go runtime为heap请求内存时，我们就可以响应。这是Go runtime依赖的一个主要内容。

问题：我知道你们实现了一些Go runtime会调用的接口，因为你们现在自己在实现内核，所以没有现成的接口可以使用。你们是全用汇编实现的这些接口吗？还是说有些还是用Golang实现，然后只在必要的时候用汇编？

回答：这就是Biscuit中1500行汇编代码的原因，它会准备好一切并运行Go runtime。有一些我们可以用C来实现，但是我们不想这么做，我们不想使用任何C代码，所以我们用汇编来实现。并且很多场景也要求用汇编，因为这些场景位于启动程序。我们的确写了一些Go代码运行在程序启动的最开始，这些Go代码要非常小心，并且不做内存分配。我们尽可能的用Golang实现了，我需要查看代码才能具体回答你的问题，你也可以查看git repo。

**问题：我有个不相关的问题，Golang是怎么实现的goroutine，使得它可以运行成百上千个goroutine，因为你不可能运行成百上千个线程，对吧？**

**回答：运行线程的主要问题是需要分配Stack，而Go runtime会递增的申请Stack，并在goroutine运行时动态的增加Stack。这就是Prologue代码的作用。当你执行函数调用时，如果没有足够的Stack空间，Go runtime会动态的增加Stack。而在线程实现中，申请线程空间会是一种更重的方法，举个例子在Linux中，对应的内核线程也会被创建。**

**问题：goroutine的调度是完全在用户空间完成的吗？**

**回答：大部分都在用户空间完成。Go runtime会申请m个内核线程，在这之上才实现的的Go routine。所有的Go routine会共享这些内核线程。人们也通过C/C++实现了类似的东西。**

问题：C是一个编译型语言，所以它可以直接变成汇编或者机器语言，它可以直接运行在CPU上，所以对于XV6来说就不用加中间层代码。但是我理解Golang也是一种编译型语言，所以它也会变成汇编语言，那么为什么还要中间层（位于机器和Go runtime之间）？XV6有这样的中间层吗？为什么有一些事情不能直接编译后运行在CPU上？

回答：好问题。Go runtime提供了各种你在XV6中运行C时所没有的功能。Go runtime提供了线程，提供了调度器，提供了hashtable，提供了GC。举个例子，为了支持GC，需要一个heap来申请内存，通常是向底层的操作系统来申请内存作为heap。这里说的中间层Go runtime需要用来完成工作的相应功能（比如说响应内存申请）。

问题：我们不能直接将runtime编译到机器代码吗？

回答：Runtime会被编译到机器码，但是当你运行Go代码时，有一部分程序是要提前运行的，这部分程序需要在那。即使C也有一个小的runtime，比如printf就是C runtime的中间层的一部分，或者字符串处理也是C runtime的一部分，它们也会被编译。C runtime有一些函数，但是这个runtime是如此之小，不像Go runtime需要支持许多Go程序所依赖的功能。

问题：看起来这里的中间层像是一个mini的系统层，它执行了一些底层的系统功能。

回答：是的，或许一种理解中间层的方法是，XV6也有一个非常非常小的中间层。当它启动的时候，它做的第一件事情是分配一些Stack这样你才能调用C的main函数。你可以认为这一小段代码是针对XV6的中间层。一旦你执行了这些指令，你就在C代码中了，然后一切都能愉快的运行。Go runtime的中间层稍微要大一些，因为有一些功能需要被设置好，之后Go runtime才能愉快的运行。

# Lecture21 网络(Networking)
