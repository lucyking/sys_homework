###一 多任务的假象
现在用操作系统好像都支持多任务。然而系统实际在执行的进程数目，是由主板上的实际cpu数目决定的。多任务只是cpu和内核创建的一个假象。调度器为为各个进程分配相应的时间片和实现进程间切换。
进程切换和进程优先级有关，进程可以分为实时进程（硬和软实时）和非实时进程。各个进程运行时按照时间片调度。分配给进程的时间片大小与其相对重要性相当。
进程会有几种不同的状态：运行，等待，睡眠。根据运行时环境进程状态会在这几种状态之间切换。
进程又可以分为用户态和核心态两类。普通进程总是可以被抢占。核心态进程不能被抢占。但是具有最高优先级的中断可以暂停用户和核心态进程。

在内核中用task_struct结构体来描述进程的具体属性。该结构定义在include/sched.h文件中。
struct task_struct{ 
	volatile long state; 
	void *stack;
	atomic_t usage;
	... ...
        ... ...

};

task_struct中的成员除简单类型变量外，还包含了很多指向其他结构体的指针。
比如说其中的volatile long state保存了进程的当前状态，state的可能取值有：
* TASK_RUNNING:进程等待被选中执行，无需等待外部事件
* TASK_INTERRUPTIBLE:处于等待或睡眠状态，等事件完成时，内核发送信号给该进程，其state转变成TASK_RUNNING状态。
* TASK_UNINTERRUPTIBLE:因内核指令而暂停的睡眠进程，不能被外部信号唤醒，只能由内核唤醒。
* TASK_STOPPED:进程停止运行，例如被调试器暂停。
* TASK_TRACED:被调试进程，用以区分常规的睡眠进程。


###二 复制进程
Linux用于复制进程的系统调用有三个：
* fork:重量级复制。建立一个父进程的完整副本，然后作为子进程执行。采用copy-on-write(COW)技术。
* vfork：与fork类似但不创建父进程副本。父子进程之间共享数据，可以节省大量cpu时间。不过由于fork采用COW技术，vfork在速度方面不再有优势。
* clone：用来产生线程，对父子进程之间的共享复制进行精确控制。

fork,vfork,clone调用的入口点是sys_fork,sys_vfork和sys_clone。这三个函数从寄存器中提取用户空间信息，然后再调用与硬件架构无关的do_fork函数，由其完成进程复制。
do_fork()的原型如下：
kernel/fork.c
long do_fork(unsigned long clone_flags,
	     unsigned long stack_start,
             struct pt_regs *regs,
             unsigned long stack_size,
             int __user *parent_tidptr,
             int __user *child_tidptr)
其中
* clone_flags：一个标志集合，用来制定控制复制过程的一些属性。
* stack_start:用户状态下栈的起始地址。
* regs:指向寄存器集合的一个指针，其中以原始形式保存了调用参数。
* stack_size:用户状态下栈的大小。通常为0。
* parent_tidptr,child_tidptr:指向用户空间地址的两个指针，分别指向父子进程的PID。
不同的fork变体，主要是通过clone_flags来区分。

###三 调度器
进程管理和调度由调度器完成。调度器的任务是实现各个任务共享CPU时间，创造程序并行执行的错觉。其任务主要分为两部分：一个是调度策略，另一个是上下文切换。
schedule函数是调度操作的基础。该函数定义在kernel/sched.c中。
调度器使用一系列数据结构来排序和管理系统中的进程。可以通过直接的或者周期性的轮询来激活调度。

###四 完全公平调度操作
CFS算法依赖于虚拟时钟，用以度量等待进程在完全公平系统中所能得到的CPU时间。所有与虚拟时钟相关的计算都在update_curr中执行。
