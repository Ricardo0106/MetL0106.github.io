---
title: Linux驱动-构造和运行模块个人学习总结
date: 2023-09-29 10:40:26
Index_img: https://github.com/Ricardo0106/Ricardo0106.github.io/blob/master/yzb/%E5%B0%8F%E7%8B%97%E5%8D%B7%E5%8D%B7.jpg
tags:
  - Linux Kernel Driver
typora-root-url: ./..
---


我发现对于学习任何一门知识，在最开始只能背诵或抄写别人的一些总结，所以tyro阶段我会抄写一些别人的文章，进行复习总结。<br>

我记得小时候看过一部电影《心灵捕手》，虽然主线剧情不是我要说的，但我仍然记者记忆中有个桥段，Will在哈佛附近的酒吧遇到Skylar的时候，Will最铁的哥们查克冒充历史系学生和美女Skylar搭讪，但哈佛大学的学生克拉克看破了查克是冒牌货，于是过来考查克历史学知识，Will不止回答了这些知识，还输出了一顿自己的理解。我一直不明白，同为学生时代的我们，究竟会有什么差距呢，后来看到这一段，我才发现，只有能输出自己的东西才算是在这个领域有所了解了吧？所以tyro阶段我也只能摘抄一些别人消化的知识点，多希望我也能有一天输出自己的东西。

![](/imgs/外设驱动开发/心灵捕手.webp)

<small>提到《心灵捕手》我又想到:<br>
你年轻彪悍，我如果和你谈论战争，你大可以向我大抛莎士比亚，背诵“共赴战场，亲爱的朋友”，但你从未亲临战阵，未试过把挚友的头拥入怀里，看着他吸着最后一口气，凝望着你，向你求助。不要以为，我了解你。也许我可以通过“知识”来看你，但那不是你，除非你愿意谈谈你自己，否则我不知道你到底是谁。有时候喜欢是一种非常简单的事，我没有进入过你的世界，只听你讲过自己的故事，你人生的前25年从未参与过，我喜欢的只是我所认识的25岁的你。我不知道为什么，我真真实实的做了一场梦，很久很久的梦，我把人生的25年从头过了一遍，非常真切的感受，30年的人生所有的故事主角都是你，我才发现，我的这份喜欢似乎是贯彻了这痛苦的一生。</small>

<small>ANY WAY，写这些东西的时候我的状态是非常不好的，最近这两年我失去了爱我的外婆，得了很多很重的疾病，毕业选择的第一家公司在春招结束的时候解散了，还有一个她。在这之前我的世界没有什么情绪，对生活也没有热情，感谢她给我平庸的人生带来了一些阳光。让我捡起来对生活的热爱，谢谢你，u complete me。人生太多变化，不知道还能在一起多久，我们已经是家人了吧，不要忘记我奥，死亡不是真的逝去，遗忘才是永恒的消亡。</small>

# 构造和运行模块

### 内核模块与应用程序的对比

大多数中小规模的应用程序是从头到尾执行单个任务。**模块只是预先注册自己以便服务于将来的某个请求**，然后其初始化函数立即结束，模块的退出函数将在模块被卸载之前调用。**内核模块的编程方式和事件驱动的编程有些类似。**
应用程序在退出时可以不管资源的释放或其他的清除工作。但**模块的退出函数必须仔细撤销初始化函数所作的一切**，否则，在系统重新引导之前某些东西就会残留在系统中。
应用程序可以调用它并未定义的函数，因为连接过程能够解析外部引用从而使用适当的函数库。但**模块仅被链接到内核，它能调用的函数仅仅是由内核导出的那些函数，而不存在任何可链接的函数库。**
模块源文件中不能包含通常的头文件，只能使用作为内核一部分的函数。
应用程序开发过程中的段错误是无害的，并且总是可以使用调试器跟踪到源代码中的问题所在；内核错误即使不影响整个系统，也至少会杀死当前进程。

#### 可重入

可重入性作为多线程编程里面重要的概念，相信大家或多或少都听说过、研究过甚至使用过。目前很多资料都重点在介绍重入锁、CAS等概念，对“可重入性”本身的讲解不是很多，所以JavaCool给大家整理了一些“可重入性”的相关知识，希望能帮助到大家！

* 可重入的定义
简单定义:"可以正确重复使用"，有两个关键：1，可以重复使用；2，并能正确使用。意味着在多次执行的时候能得到正确的值，并不受其他调用的影响。
官方定义：若一个程序或子程序可以“在任意时刻被中断然后操作系统调度执行另外一段代码，这段代码又调用了该子程序不会出错”，则称其为可重入（reentrant或re-entrant）的。即当该子程序正在运行时，执行线程可以再次进入并执行它，仍然获得符合设计时预期的结果。与多线程并发执行的线程安全不同，可重入强调对单个线程执行时重新进入同一个子程序仍然是安全的。
这里也有一段比较好的英文阐释：
A computer program or routine is described as reentrant if it can be safely called again before its previous invocation has been completed (i.e it can be safely executed concurrently)
可重入函数主要用于多任务环境中，一个可重入的函数简单来说就是可以被中断的函数，也就是说，可以在这个函数执行的任何时刻中断它，转入OS调度下去执行另外一段代码，而返回控制时不会出现什么错误；而不可重入的函数由于使用了一些系统资源，比如全局变量区，中断向量表等，所以它如果被中断的话，可能会出现问题，这类函数是不能运行在多任务环境下的。

* 产生背景
可重入概念是在单线程操作系统的时代提出的。一个子程序的重入，可能由于自身原因，如执行了jmp或者call，类似于子程序的递归调用；或者由于操作系统的中断响应。UNIX系统的signal的处理，即子程序被中断处理程序或者signal处理程序调用。所以，可重入也可称作“异步信号安全”。这里的异步是指信号中断可发生在任意时刻。 重入的子程序，按照后进先出线性序依次执行。

* 编写可重入代码注意的条件
若一个函数是可重入的，则该函数应当满足下述条件：

不能含有静态（全局）非常量数据。
不能返回静态（全局）非常量数据的地址。
只能处理由调用者提供的数据。
不能依赖于单实例模式资源的锁。
调用(call)的函数也必需是可重入的。
上述条件就是要求可重入函数使用的所有变量都保存在呼叫堆叠的当前函数栈（frame）上，因此同一执行线程重入执行该函数时加载了新的函数帧，与前一次执行该函数时使用的函数帧不冲突、不互相覆盖，从而保证了可重入执行安全。

多“用户/对象/进程优先级”以及多进程（Multiple processes），一般会使得对可重入代码的控制变得复杂。同时，IO代码通常不是可重入的，因为他们依赖于像磁盘这样共享的、单独的（类似编程中的静态、全域）资源。

* 与线程安全的关系
可重入与线程安全两个概念都关系到函数处理资源的方式。但是，他们有重大区别：可重入概念会影响函数的外部接口，而线程安全只关心函数的实现。大多数情况下，要将不可重入函数改为可重入的，需要修改函数接口，使得所有的数据都通过函数的调用者提供。要将非线程安全的函数改为线程安全的，则只需要修改函数的实现部分。一般通过加入同步机制以保护共享的资源，使之不会被几个线程同时访问。

操作系统背景与CPU调度策略：
可重入是在单线程操作系统背景下，重入的函数或者子程序，按照后进先出的线性序依次执行完毕。

多线程执行的函数或子程序，各个线程的执行时机是由操作系统调度，不可预期的，但是该函数的每个执行线程都会不时的获得CPU的时间片，不断向前推进执行进度。可重入函数未必是线程安全的；线程安全函数未必是可重入的。

例如，一个函数打开某个文件并读入数据。这个函数是可重入的，因为它的多个实例同时执行不会造成冲突；但它不是线程安全的，因为在它读入文件时可能有别的线程正在修改该文件，为了线程安全必须对文件加“同步锁”。
另一个例子，函数在它的函数体内部访问共享资源使用了加锁、解锁操作，所以它是线程安全的，但是却不可重入。因为若该函数一个实例运行到已经执行加锁但未执行解锁时被停下来，系统又启动该函数的另外一个实例，则新的实例在加锁处将转入等待。如果该函数是一个中断处理服务，在中断处理时又发生新的中断将导致资源死锁。fprintf函数就是线程安全但不可重入。
* 可重入锁
可重入锁也叫递归锁，它俩等同于一回事，指的是同一线程外层函数获得锁之后，内层递归函数仍然能获得该锁的代码，同一线程在外层方法获取锁的时候，再进入内层方法会自动获取锁。也就是说，线程可以进入任何一个它已经拥有的锁所同步着的代码块。ReentrantLock 和 synchronized 就是典型的可重入锁！

**操作系统的作用**是

- 为应用程序提供一个对计算机硬件的一致视图
- 负责程序的独立操作并保护资源不受非法访问。该任务只有在CPU能够保护系统软件不受应用程序破坏时才能完成。

人们在CPU中实现不同的操作模式（或者级别）。不同的级别具有不同的功能，在较低的级别中将禁止某些操作。程序代码只能通过有限数目的“门” 从一个级别切换到另一级别。Unix使用了两个这样的级别：内核运行在最高级别（也称作超级用户态），可以进行所有操作；应用程序运行在最低级别（即所谓的用户态），处理器控制着对硬件的直接访问以及对内存的非授权访问。两种模式具有不同的优先权等级，每个模式都有自己的内存映射，即自己的地址空间。

应用程序执行系统调用或者被硬件中断挂起时，Unix将执行模式从用户空间切换到内核空间

- 执行系统调用的内核代码运行在进程上下文中，它代表用户进程执行操作，因此可以访问进程地址空间的所有数据
- 处理硬件中断的内核代码和进程是异步的，与任何一个特定进程无关。

一个驱动程序通常要执行两类任务

- 某些函数作为系统调用的一部分而执行
- 其他函数负责中断处理

### 内核中的并发

内核编程区别于常见应用程序编程的地方在于对并发的处理。即使是最简单的内核模块，都需要在编写时铭记：同一时刻，可能会有许多事情正在发生。

##### 内核编程必须考虑并发的原因

- Linux系统中通常正在运行多个并发进程，他们可能同时使用我们的驱动程序。

- 大多数设备能够中断处理器，而中断处理程序异步运行，而且可能在驱动程序试图处理其他任务时被调用。
- 一些抽象软件，例如内核定时器，也在异步运行。
- Linux可以运行在SMP系统上，可能同时有不止一个CPU运行我们的驱动程序。
- 在2.6中内核代码已经是可抢占的，意味着即使在单处理器系统上也存在许多类似多处理器系统的并发问题。

Linux内核代码（包括驱动程序代码）必须是可重入的，它必须能够同时运行在多个上下文中。对编写正确的内核代码来说，优良的并发管理是必需的。

#### 当前进程

虽然内核模块不像应用程序那样顺序执行，然而内核执行的大多数操作还是和某个特定的进程相关。
内核代码可通过访问全局项current来获得当前进程。它是一个指向struct task_struct的指针。current指针指向当前正在运行的进程。内核代码可以通过current获得与当前进程相关的信息。例如，下面的语句通过访问struct task_struct的某些成员来打印当前进程的ID和命令名：

```c
printk(KERN_INFO "The process is \" %s \" (pid %i)\n",
current->common,current->pid);
```

为了支持SMP系统，内核开发这设计了一种能找到运行在相关CPU上的当前进程的机制。它必须是快速的，因为对current的引用会很频繁。一种不依赖于特定架构的机制通常是，**将指向task_struct结构的指针隐藏在内核栈中**。

## 编译和装载

### 编译模块

内核是一个大的、独立的程序，为了将它的各个片段放在一起，要满足很多详细而明确的要求。

在构造内核模块之前，有一些先决条件首先应该得到满足

- 确保具备正确版本的编译器、模块工具和其他必要的工具（Documentation/Changes文件）。注意：使用太新的工具也偶尔会导致问题。
- 准备内核树，配置并构造内核。

```c
obj-m := hello.o
```

上面的赋值语句（它利用了GNU make的扩展语法）说明了有一个模块需要从目标文件hello.o中构造，而从该目标文件中构造的模块名称为hello.ko。

如果要构造的模块名称为module.ko，并由两个源文件生成（file1.c和file2.c），那么makefile可编写如下

```makefile
obj-m := module.o
module-objs := file1.o file2.o
```

为了让上面这种类型的makefile文件正常工作，**必须在大的内核构造系统环境中调用它们**。如果内核源代码树保存在~/kernel-2.6目录，则用来构造模块的make命令（在包含模块代码和makefile的目录中输入）

```makefile
make -C ~/kernel-2.6 M=`pwd` modules
```

上述命令**首先改变目录到内核源代码目录**，该目录保存由内核的顶层makefile文件。**M=选项**让该makefile在构造modules目标之前返回到模块源代码目录。然后，**modules目标指向obj-m变量中设定的模块**。

另一种makefile方法，可以使得**内核树之外的模块构造**更加容易。

```makefile
# If KERNELRELEASE is defined, we've been invoked from the
# kernel build system and can use its language.
ifneq ($(KERNELRELEASE),)
    hello_world-objs := main.o
    obj-m := hello_world.o

# Otherwise we were called directly from the command
# line; invoke the kernel build system.
else
    KERNELDIR ?= /lib/modules/$(shell uname -r)/build
    PWD := $(shell pwd)

.PHONY: modules
modules:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) modules
endif

.PHONY: clean
clean:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) clean

```

在一个典型的构造过程中，该makefile将被读取两次。当makefile从命令行运调用时，KERNELRELEASE变量并未设置，这个makefile通过已安装的模块目录中指向内核构造树的符号链接，定位内核的源代码目录。如果实际运行的内核并不是要构造的内核，则可以在命令行提供KENELDIR=选项或者设置KERNELDIR环境变量，也可以修改makfile中用来设置KERNELDIR的行。找到内核源代码树之后，这个makefile会调用modules目标，通过之前描述的方法第二次运行make命令，以便运行内核构建系统。

### 装载和卸载模块

**insmod**将模块装入内核。它将模块的代码和数据装入内核，然后使用内核的符号表解析模块中任何为解析的符号。与链接器不同，内核不会修改模块的磁盘文件，仅仅修改内存中的副本。insmod可以接受一些命令行选项，可以在模块链接到内核之前给模块中的整型和字符串变量赋值。一个设计良好的模块，可以在装载时进行配置。

函数**sys_init_module**给模块分配内核内存。有且只有系统调用的名字前带有sys_前缀。

**modprobe**也用来将模块装载到内核中。它与insmod的区别在于，会考虑要装载的模块是否引用了一些当前内核不存在的符号。如果存在这类引用，它会在当前模块搜索路径中查找定义了这些符号的其他模块。如果它找到了这些模块，则会同时将它们装载到内核。

**rmmod**可以从内核中移除模块。如果内核认为模块仍在使用状态或者内核被配置为禁止移除模块，则无法移除该模块。

**lsmod**列出当前装载到内核中的所有模块，它通过读取/proc/modules文件获得这些信息。有关当前已装载模块的信息也可以在/sys/module下找到。

### 版本依赖

在缺少modversion的情况下，我们的**模块代码必须针对要链接的每个版本的内核重新编译**。**模块和特定的内核版本定义的数据结构和函数原型紧密关联**。

内核不会假定一个给定的模块是针对正确的内核版本构造的。在构造过程中，**可以将自己的模块和vermagic.o链接**。该目标文件包含了大量有关内核的信息，在试图装载模块时，这些信息可以用来检查模块和正在运行的内核的兼容性。

如果打算编写一个**能够和多个内核版本一起工作的模块**，则必须**使用宏以及#ifdef**来构造并编译自己的代码。

- UTS_RELEASE，该宏扩展为一个描述内核版本的字符串
- LINUX_VERSION_CODE，该宏扩展为为内核版本的二进制表示，版本发行号中的每一部分对应一个字节
- KERNEL_VERSION(major,minor,release)，该宏以组成版本号的三个部分，创建整数的版本号

**不应随意使用#ifdef条件语句**将驱动程序代码弄得杂乱无章。最好的一个解决方法就是**将所有相关的预处理条件语句集中存放在一个特定的头文件中**。一般而言，**依赖于特定版本的代码应该隐藏在低层宏或函数之中**。

### 平台依赖

如果模块和某个给定内核工作，它也**必须和内核一样了解目标处理器**。通过**链接vermagic.o**，在装载模块时，内核会检查处理器相关的配置选项以便确保模块匹配于运行中的内核。

### 内核符号表

insmod使用**公共内核符号表**来解析模块中未定义的符号。**公共内核符号表**包含了**所有的全局内核项（函数和变量）的地址**，它是实现模块化驱动程序所必须的。

当模块被装入内核后，它所**导出的任何符号**都会变成内核符号表的一部分。通常情况下，模块只需实现自己的功能，而无需导出任何符号。如果其他模块要从某个模块获得好处，则需要导出符号。这样就可以在其他模块上**层叠新的模块**，即**模块层叠技术**。

如果一个模块要向其他模块导出符号，则应该使用下面的宏

```c
EXPORT_SYMBOL(name);
EXPORT_SYMBOL_GPL(name);
```

GPL版本使得要导出的模块只能被GPL许可证下的模块使用。**符号必须在模块文件的全局部分导出**，不能在函数中导出。

### 预备知识

内核是一个特定的环境，对需要和它接口的代码有其自己的一些要求。

所有的模块代码中都包含下面两行代码

```c
#include <linux/module.h> 
#include <linux/init.h>
```

**module.h**包含可装载模块需要的大量符号和函数的定义。**init.h**是为了指定初始化和清除函数。

大部分模块还包含**moduleparam.h**头文件，这样就可以**在装载模块时向模块传递参数**。

尽管不是严格要求，模块应该指定所使用的许可证，例如

```c
MODULE_LICENSE("GPL")
```

内核可以识别

- GPL
- GPL v2
- GPL and additional rights
- Dual BSD/GPL
- Dual MPL/GPL
- Proprietary

如果模块没有显式地标记为上述可识别的许可证，就会被假定是专有的，内核装在这种模块就会被“污染”。

还可包含其他描述性定义

```c
MODULE_AUTHOR
MODULE_DESCRIPTION
MODULE_VERSION
MODULE_ALIAS
MODULE_DEVICE_TABLE
```

这些MODULE_声明，一般**放在文件的最后**。

### 初始化和关闭

模块的**初始化函数**负责**注册**模块所提供的任何**设施**。这里的**设施**指的是一个可以被应用程序访问的新功能，例如一个完整的驱动程序或一个新的软件抽象。

```c
static int __init initialization_function(void)
{
	/*Initialization codes*/
}

module_init(initialization_function);
```

初始化函数被声明为static，它在特定文件之外没有其他意义。但这并不是一个强制性规则，因为一个模块函数如果要对内核其他部分可见，则必须被显式导出。

__init标记表明该函数仅在初始化期间使用。载模块被装载后，模块装载器会将初始化函数丢弃，以释放其占用的内存。__init和 __initdata很值得使用，但请注意，不要在结束初始化之后仍要使用的函数或数据上使用它们。

module_init的使用是强制的。这个宏会在模块的目标代码中增加一个特殊的段，用于说明内核初始化函数所在的位置。

模块可以注册许多不同类型的设施。对每种设施，对应有具体的内核函数用来完成注册。传递到内核注册函数的参数通常是指向用来描述新设施及设施名称的数据结构指针，而数据结构通常包含指向模块函数的指针。这样，模块中的函数就会在恰当的时间被内核调用。

#### 清除函数

每个重要的模块都需要一个清除函数，它在模块被移除前注销接口并向系统返回所有资源。该函数定义如下

```c
static void __exit cleanup_function(void)
{
	/*Cleanup codes*/
}
module_exit(cleanup_function);
```

清除函数没有返回值。
__exit修饰词标记改代码仅用于模块卸载，编译器将把该函数放在特殊的ELF段中。被标记为__exit的函数只能在模块被卸载或者系统关闭时调用。

module_exit声明帮助内核找到模块的清除函数。

如果一个模块未定义清除函数，则内核不允许卸载该模块。

#### 初始化过程中的错误处理

在内核中注册设施时，**时刻铭记注册可能会失败**。模块代码必须始终检查返回值，确保所有的操作已真正成功。

如果在注册设施时遇到任何错误，首先要判断模块是否可以继续初始化。通常在某个注册失败后可以通过降低功能来继续运转。

如果在发生了某个特定类型的错误之后无法继续装载模块，则要将出错之前的任何注册工作撤销掉。

错误恢复的**处理有时使用goto语句**比较有效，可以避免大量复杂的、高度缩进的“结构化”逻辑。

```c
int __init my_init_func(void)
{
	int err;
	/*Register using pointers and names */
	err = register_this(ptr1,"skull");
	if(err)
		goto fail_this;
	err = register_that(ptr2,"skull");
	if(err)
		goto fail_that;
	err = register_those(ptr3,"skull");
	if(err)
		goto fail_those;
	/*Success*/
	return 0;

fail_those:
	unregister_that(ptr2,"skull");
fail_that:
	unregister_this(ptr1,"skull");
fail_this:
	return err;
}
```

以上的代码再出错的时候使用goto语句，它将只撤销出错时刻以前所成功注册的那些设施。
my_init_module的返回值err是一个错误编码，它是定义在<linux/error.h>中的负整数。每次返回合适的错误编码是一个好习惯。

模块的清除函数需要撤销初始化函数所注册的所有设施，习惯上，以相反于注册的顺序撤销设施。

```c
void __exit my_cleanup_function(void)
{
	unregister_those(ptr3,"skull");
	unregister_that(ptr2,"skull");
	unregister_this(ptr1,"skull");
	return
}
```

**每当发生错误时从初始化函数中调用清除函数**，将减少代码的重复并使代码更清晰、更有条理。当然，清除函数**必须在撤销每项设施的注册之前检查它的状态**。

```c
struct something *item1;
struct somethingelse *item2;
int stuff_ok;

void my_cleanup(void)
{
	if(item1)
		release_thing(item1);
	if(item2)
		release_thing2(item2);
	if(stuff_ok)
		unregister_stuff();
	return;
}

int __init my_init(void)
{
	int err = -ENOMEM;
	item1 = allocate_thing(arguments);
	item2 = allocate_thing2(arguments2);
	if(!item1 || !item2)
		goto fail;
	err = register_stuff(item1,item2);
	if(!err)
		stuff_ok = 1;
	else
		stuff_ok = 0;
	return 0;

fail:
	my_cleanup();
	return err;
}
```

以上代码中的初始化方式可以扩展到对大量设施的支持。需要注意的是，因为清除函数被非退出代码调用，因此**不能将清除函数标记为__exit**。

#### 模块装载竞争

首先要始终铭记的是，**在注册完成后，内核的某些部分可能会立即使用我们刚刚注册的任何设施**。也就是说，**初始化函数还在运行的时候，内核就完全可能调用我们的模块**。

所以，**在首次注册完成后，代码就应该准备好被内核的其他部分调用，在用来支持某个设施的所有内部初始化完成之前，不要注册任何设施。**

### 模块参数

由于系统的不同，**驱动程序需要的参数也会发生变化**，例如设备编号和一些用来控制驱动程序操作方式的参数。为满足这种需求，**内核允许对驱动程序指定参数**，而这些参数**可在装载驱动程序模块时改变。**

这些**参数的值可在运行insmod或modprobe命令时赋值**，而**modprobe还可以从其配置文件（/etc/modprobe.conf）中读取参数值**。这两个命令可以接受几种参数类型的赋值

- bool
- invbool：bool和invbool关联的变量应该是int类型
- charp：字符指针值。
- int
- long
- short
- uint
- ulong
- ushort

在insmod改变模块参数之前，**模块必须让这些参数对insmod命令可见**。参数必须**通过module_param宏来声明**，它有三个参数

- 变量的名称
- 类型
- 用于sysfs入口项的访问许可掩码

这个宏**必须放在任何函数之外**，通常是在源文件的头部。

模块的装载器**也支持数组参数**，在**提供数组值时用逗号划分各数组成员**。要声明数组参数，使用**宏module_param_array(name,type,num,perm)**。模块装载器会**拒绝接受超过数组大小的值**。

如果我们**需要的类型不在上面列出的清单中**，可以使用**模块代码中的钩子**来定义这些类型。

**所有模块参数都应给定一个默认值**，insmod只会在用户明确设置了参数的值的情况下参会改变参数的值。

**module_param的最后一个参数**是访问许可值，它用来**控制谁能访问sysfs中对模块参数的表述**

- 如果perm被设置为0，就不会有对应的sysfs入口项
- 否则模块参数会在sys/module中出现，并设置为给定的访问许可

如果一个参数通过sysfs被修改，内核不会以任何方式通知模块。大多数情况下，不应该让模块参数可写。

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>

static char *whom = "Mom";
static int howmany = 1;

module_param(howmany, int,   S_IRUGO);
module_param(whom,    charp, S_IRUGO);

static
int __init m_init(void)
{
	printk(KERN_WARNING "parameters test module is loaded\n");

	for (int i = 0; i < howmany; ++i) {
		printk(KERN_WARNING "#%d Hello, %s\n", i, whom);
	}
	return 0;
}

static
void __exit m_exit(void)
{
	printk(KERN_WARNING "parameters test module is unloaded\n");
}

module_init(m_init);
module_exit(m_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Bryan");
MODULE_DESCRIPTION("Module parameters test program");

```

直接使用insmod，装载模块

```
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ sudo insmod module_parameters.ko 
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ dmesg | tail -10
[  282.742416] raid6: .... xor() 21210 MB/s, rmw enabled
[  282.742417] raid6: using avx2x2 recovery algorithm
[  282.783190] xor: automatically using best checksumming function   avx       
[  282.877516] Btrfs loaded, crc32c=crc32c-intel
[  367.429218] parameters test module is unloaded
[  378.711958] parameters test module is loaded
[  378.711959] #0 Hello, Mom
[  470.278217] parameters test module is unloaded
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ 
```

使用insmod装载模块时，携带参数

```c
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ sudo insmod module_parameters.ko  whom=dady howmany=3
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ sudo rmmod module_parameters 
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ dmesg | tail -10
[  378.711959] #0 Hello, Mom
[  470.278217] parameters test module is unloaded
[  473.729918] parameters test module is loaded
[  473.729919] #0 Hello, Mom
[  556.020495] parameters test module is unloaded
[  578.373167] parameters test module is loaded
[  578.373168] #0 Hello, dady
[  578.373168] #1 Hello, dady
[  578.373169] #2 Hello, dady
[  582.573809] parameters test module is unloaded
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ 
```

在insmod装载模块之前，在/sys/module目录下无法找到module_parameters目录

```c
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ ls /sys/module/module_parameters
ls: cannot access '/sys/module/module_parameters': No such file or directory
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ 
```

insmod装在模块后，可以在/sys/module目录下找到module_parameters目录，并在/sys/module/module_parameters/parameters目录下看到两个参数

```c
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ sudo insmod module_parameters.ko  whom=dady howmany=3
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ ls /sys/module/module_parameters
coresize  initsize   notes       refcnt    srcversion  uevent
holders   initstate  parameters  sections  taint
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ ls /sys/module/module_parameters/parameters/
howmany  whom
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ ls /sys/module/module_parameters/parameters/howmany 
/sys/module/module_parameters/parameters/howmany
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ cat /sys/module/module_parameters/parameters/howmany 
3
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ cat /sys/module/module_parameters/parameters/whom 
dady
bryan@ubuntu:~/Desktop/Linux-Device-Driver-master/02_module_parameters$ 
```

