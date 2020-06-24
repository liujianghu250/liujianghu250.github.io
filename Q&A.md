## 逻辑地址、虚拟地址、线性地址、物理地址的区别是什么？
答：
**逻辑地址**是编程时使用的地址，比如常用到的指针，指针内存的是某变量的地址，这个地址指的就是逻辑地址。

**物理地址**是用于内存芯片级的单元寻址，与处理器和CPU连接的地址总线相对应。

**虚拟地址**和**线性地址**是一个东西，都是指从逻辑地址转换到虚拟地址的中间的一个地址，只有在保护模式中才使用。因为它不是真实存在的地址，与物理地址相对应，所以叫虚拟地址，也因为它的地址空间是线性的，所以也叫线性地址。

（有时候好像也会用虚拟地址代指逻辑地址，还是尽量避免这种情况吧。）

### 实模式
在实模式中，逻辑地址=段基址+段内偏移，即段寄存器中存放段基址的高16位，段内偏移也只有16位。

物理地址 = 段寄存器内的值 * 10H + 偏移量

实模式下没有线性地址。

### 保护模式
保护模式下的转换相对比较复杂，简单总结如下：

逻辑地址通过分段机制转换成线性地址。

线性地址再通过分页机制转换成物理地址。

注意：Linxu对于分段机制只是简单使用，四个主要的代码段与数据段的基址都是0，限长都是4GB，所以线性地址的值其实和逻辑地址的偏移量的值是一致的。


## 为什么每个进程都需要有自己的页表，每个线程是否有自己的页表，每个内核线程呢？

### 为什么每个进程都需要有自己的页表？

每个进程都需要有自己的页表，因为每个进程的地址空间都应该是独立的，即不同进程间，大多数数据应该是独立的，不能互相访问的。

每个进程都拥有自己的页表，可以保证别的进程在一般情况下，是访问不到它的数据的，因为两个不同进程即便使用相同的线性地址，访问的也是各自的地址空间，指向的物理地址也一般不会相同。

可以思考这样一个问题，假如多个进程共同享有一个页表，如何保证彼此互不影响呢？

在每个页表项上标注该页的拥有者(某一个进程）也许是一个解决办法，就像linux里的文件一样，但是这样一来，开销就太大了，在空间上，需要存储进程的id，在时间上，每次访问页时都需要对页的拥有者进行验证……

由此可以看出，独立的页表，或者说独立的地址空间，对于作为资源分配的单位的进程来说，是相当必要的。

### 每个线程是否有自己的页表？

要回答这个问题，先要明白线程的含义与作用，以及线程与进程的区别。

进程是分配系统资源（CPU时间、内存等）的实体。

而一个进程，可能由多个线程组成。每个线程负责进程的一小部分工作。

举个例子，在电脑上打开qq，这就是一个进程。如果在qq中同时打开多个聊天窗口，那么每个聊天窗口就是一个线程。

每个线程拥有一些独立的资源，但大部分资源，都是和其他所有线程，也就是整个进程共享的。

而对于内存资源来说，最好最简单的共享方法，无疑就是共享同一个地址空间，也就是说，共享页表。

对于线程来说，它的独立性没有进程那么强，反而有大量需要和其他进程共享的内存资源，所以，线程不需要有自己的页表。

### 内核线程是否有自己的页表？

上面提到的线程，其实是**用户线程**。

还有一种线程，叫**内核线程**，负责刷新磁盘高速缓存，交换不用的页框，维护网络连接等任务。

内核线程只运行在内核态，而普通进程既可以运行在内核态，也可以运行在用户态。

也正因为内核线程只运行在内核态，所以用户空间对它而言没有意义，它只使用内核空间，也就是大于PAGE_OFFSET的线性地址空间。

这也就意味着，它完全不需要拥有自己的页表——使用**内核页表**就够了。

#### 内核页表
内核维持着一组自己使用的页表，驻留在所谓的主内核页全局目录中。主内核页全局目录的最高目录项部分作为参考模型，为系统中每个普通进程对于的页全局目录项提供参考模型。

内核会确保对主内核页全局目录的修改能传递到由进程使用的页全局目录中。


## 进程切换（换入、换出）时是如何保留自己的页表的？

从本质上来讲，每个进程切换由两步组成。

1. 切换页全局目录以安装一个新的地址空间；

2. 切换内核态堆栈和硬件上下文，因为硬件上下文提供了内核执行新进程所需要的所有信息，包括CPU寄存器。

保留页表，无疑是发生在第一步。

那么这一步是如何实现的呢？

首先认识一个寄存器，cr3控制寄存器，它存放的是正在使用的页目录的物理地址。

所以切换当前页目录表，也就是改变cr3寄存器的值，具体操作如下：

Linux把cr3控制寄存器的内容保存在**前一个执行进程的描述符**中，然后把**下一个要执行进程的描述符**的值装入cr3寄存器中。

也就是说，进程切换时，进程保留自己的页表的方法，是将页目录表的物理地址，存到进程的描述符中。

在2.6.11中，进程描述符都是task_struct类型结构，其中有一个字段是指向内存区描述符结构的指针，struct mm_struct *mm;

而struct mm_struct中有指向页目录表的指针pgd_t * pgd;

### cr3中是页目录表的物理地址，而指针是逻辑地址，是怎么实现相互转换的呢？

页目录表放在内核空间，物理地址和逻辑地址的转换很简单，加减PAGE_OFFSET即可。


## X86_32的虚拟地址空间是如何分布的？X86_64的虚拟地址空间是如何分布的？
### X86_32的虚拟地址空间分布

## 不同页表的两个页表项可以指向同一个页吗？ 

可以。

内核通过页描述符记录每个物理页的信息，描述符中有一个字段是atomic_t _count，页的引用计数器。

该字段为-1，说明相应物理页空闲，可以被分配给任一进程或内核本身；如果该字段**大于或等于0**，则说明页框被分配给了**一个或多个进程**，或者用于存放一些内核数据结构。

由上面的描述就可以看出，同一个物理页，是有可能被分配给多个进程的。

比如共享内存。还有采用Copy On Write技术的父子进程的页表。

## x86：32位、PAE、64位，分别的页表组织是怎样？
### 32位

RAM为4GB，线性地址32位，物理地址也是32位，cr3寄存器中存放页目录表的物理地址。

#### 页目录项中PS=0，页大小为4KB时。

根据cr3控制寄存器找到页目录表，每个页目录表项4字节，最多1024项。

线性地址最高10位是Directory字段，即页目录表的索引，根据该字段可以找到一个具体的页目录表项（2^10 = 1024）。

页目录表项中存放着页表的基址，可以找到一个具体页表。

线性地址中间10位是Table字段，是页表的索引，根据该字段可以找到一个具体的页表项。

页表项结构与页目录表项相同，里面存放一个物理页的基址。

线性地址最低12位是Offset字段，是页内偏移，由此可以找到具体某一个字节的物理地址。

线性地址分级为10+10+12

#### PS = 1,页大小为4MB
cr3指向页目录表。

线性地址最高10位指向页目录表1024个项中的一个，每个项都指向一个物理页。

线性地址低22位作为页内偏移，找到一个字节的物理地址。

线性地址分级为10+22

### PAE
RAM为64GB，线性地址32位，但物理地址36位，cr3寄存器存放页目录指针表（PDPT）的基地址。

一个页目录指针表中，有4个64位表项。

#### PS=0，即页大小为4KB时。
cr3指向一个PDPT

线性地址最高两位（31-30）指向PDPT4个项中的一个。每个项都指向一个页目录表。

64GB的RAM被分成2^24个物理页，页表项的物理地址字段从20位扩展到了24位，再加上12个标志位，32位的页表项已经满足不了需求。为了对齐方便访问，所以每个页表项大小从32位变成了64位。

页目录表中页目录表项的数目也从1024变成了512。

线性地址的位（29-21），9位，指向页目录表中的512个项中的一个，而每一个项都指向一个页表。

线性地址位20-12，9位，指向页表中512个项中的一个，而每一个项都指向一个物理页。

线性地址位11-0,12位，作为页内偏移，最终找到一个字节的物理地址。

线性地址分级为2+9+9+12

#### PS=1,页大小为2MB

cr3指向一个PDPT

位31-30指向PDPT中4个项中的一个

位29-21指向页目录中512个项中的一个

位20-0，2MB页中的偏移量

线性地址分级为2+9+21

### 64位
4级分页。

采用页全局目录，页上级目录，页中间目录，页表，具体实现与32位类似。

页大小为4KB，寻址使用48位，线性地址分级为 9+9+9+9+12。

高16位被用作符号扩展，这高16位要么全是0，要么全是1。



## x86架构下的内存为什么分为ZONE_DMA、ZONE_DMA32、ZONE_NORMAL、ZONE_HIGH？ X86_64下包含哪些Zone类型？

### x86架构下的内存为什么分为ZONE_DMA、ZONE_DMA32、ZONE_NORMAL、ZONE_HIGH？

因为80x86架构有两种硬件约束：

1. ISA总线的直接内存存取（DMA）处理器有一个严格的限制：它们只能对RAM的前16MB寻址；

2. 在具有大容量RAM的现代32位计算机中，CPU不能直接访问所有物理内存。以Linux为例，在初始化阶段，内核只把896MB的RAM窗口映射到内核线性地址空间，如果一个程序需要对现有RAM的其他部分寻址，那就必须把某些其他线性地址间隔映射到所需的RAM。。

因为第一种限制，低于16MB的内存被称为ZONE_DMA,单独作为一个管理区。该区包含的页框可以由老式基于ISA的设备通过DMA使用。

在x86_64架构下，还需要ZONE_DMA32，因为需要32位地址寻址的DMA.这部分是高于16MB，低于4GB。

因为第二种限制，高于896MB的内存页框被称为ZONE_HIGHMEM,不能由内核直接访问。64位体系结构上ZONE_HIGHMEM总是为空。

剩余的部分，即高于16MB，低于896MB的部分，被称为ZONE_NORMAL,内核可以直接进行访问，但不能通过DMA使用。

### X86_64下包含哪些Zone类型？

和x86相比，多了ZONE_DMA32，少了ZONE_HIGHMEM。

0-16MB，是ZONE_DMA

16MB-4GB，是ZONE_DMA32

4GB以上，是ZONE_NORMAL

## 伙伴系统算法的原理？

 将所有空闲页框分组成11个块链表，每个块链表的节点分别是1,2,4,8,16,32,64,128,256,512,1024个连续的页框。
当需要分配内存时，去寻找最小的，可以满足条件的块。

比如要请求分配一个250个页框的内存。

1.去256个页框的链表中寻找，如果有空闲的块，就将它分配给请求内存的进程。

2. 如果没有空闲的块，就去寻找下一个更大的页块，即512个页框的块。如果有空闲块，就将该块分成两等份，一半分配给进程，另一半插入到256个页框的链表中。

3.如果512个页框的链表中也没有空闲块，也去寻找下一个更大的页块，即1024个页框的块。如果有，就先将该页框两等分，其中一半插入512个页框的链表中，另一半再次二等分，一半插入256个页框的链表中，另一半分配给进程。

4.如果1024个页框的链表中也没有空闲块，就放弃分配内存并发出出错信号。

而释放过程就是以上过程的逆过程，在释放后悔试图把一对空闲伙伴块合并。

伙伴算法可以尽可能避免外部碎片的产生，因为剩下的所有空闲块，最小也是1个页框，完全可以分配给需要的内存。

而合并操作也保证了连续空间尽可能大。

但它会导致内部碎片。

## 释放一个page到伙伴系统后，这个page需要跟旁边的空闲page合并成更大的块，那么是跟左边的合并 还是跟右边的合并呢？ 

答案取决于块的地址。

在伙伴系统中，对于伙伴的定义如下：

 1. 两个块具有相同的大小，记作b
 2. 它们的物理地址是连续的
 3. 第一个块的第一个页框的物理地址是 2 x b x 2^12的倍数。
 
第三个条件是为了保证内存对齐，也是必须满足的条件。

所以，如果释放的page，也可以扩大范围到包含b个页框的块（对于单独的一页，b就为1），它的第一个页框物理地址是2 x b x 2^12的倍数，那它就和右边（地址比它大）的块合并，否则，就和左边的块合并。

## pthread_create()和fork()有什么区别？ 
pthread_creat()和fork()都是用户态的API，但作用不同。一个是创建线程，一个是创建进程。

它们调用的系统调用也不同，一个调用了clone（），一个调用了fork()。

这两个系统调用的服务例程自然也不相同，一个是sys_clone(),一个是sys_fork(),但两个服务例程都调用了do_fork()函数。

`
asmlinkage long sys_fork(struct pt_regs *regs)
{
	return do_fork(SIGCHLD, regs->rsp, regs, 0, NULL, NULL);
}

asmlinkage long sys_clone(unsigned long clone_flags, unsigned long newsp, void __user *parent_tid, void __user *child_tid, struct pt_regs *regs)
{
	if (!newsp)
		newsp = regs->rsp;
	return do_fork(clone_flags, newsp, regs, 0, parent_tid, child_tid);
}
`

## 什么是内核抢占？系统中有哪些内核抢占的时机？ 
### 什么是内核抢占？

首先介绍非抢占式内核:当进程在内核态执行时，它不能被任意挂起，也不能被另一个进程代替。

而对于内核抢占，可以这么说：

如果进程正在执行内核函数，即它在内核态运行时，允许发生内核切换（被替代的进程是正执行内核函数的进程），这个内核就是抢占的。

注意，非抢占式内核页可以自动放弃CPU，从而进行进程切换，比如进程由于等待资源而不得不转入睡眠状态。

这种切换称为计划性切换。抢占式内核和非抢占式内核不同的切换方式是另一种，在响应引起进程切换的异步事件（例如唤醒高优先权进程的中断处理程序），这种叫强制性进程切换。

总结起来就是，抢占内核的特点为，一个在**内核态**运行的进程，可能在执行内核函数期间被**另一个**进程取代。

### 内核抢占的时机有哪些？
1. 一个进程在执行异常处理程序时（肯定是内核态），另一个具有较高优先级的进程B变为可执行状态。比如发出了中断请求而且相应的处理程序唤醒了进程B。

2. 一个执行异常处理程序的进程已经用完了它的时间配额。（只有当内核正在执行异常处理程序（尤其是系统调用时），而且内核抢占没有被显式禁止时，才可能抢占内核，此外，本地CPU还必须打开本地中断，否则无法完成内核抢占）

3. 结束内核控制路径（通常是一个中断处理程序）时发生。

4.也可能是在异常处理程序调用preemt_enable()重新允许内核抢占时发生。

5.还可能发生在启用可延迟函数的时候。

## Linux内核有哪些同步机制？他们的区别是什么？哪些同步机制可以用于中断处理函数？

技术 | 说明 | 适用范围
----|----|----
每CPU变量| 在CPU之间复制数据结构 |所有CPU
原子操作 
内存屏障
自旋锁
信号量
顺序锁
本地中断的禁止
本地软中断的禁止
读-拷贝-更新（RCU）

## 获得自旋锁后可以发生内核抢占么？为什么？
不能。从效率看，因为如果有其他进程在等待锁，那么当前占用自旋锁的进程被抢占会导致等待锁的进程忙等，会浪费大量资源。

事实上，适合使用自旋锁的内核资源都很锁很短的时间，与其马上抢占，不如等待释放自旋锁后再抢占。

从代码层面看，spin_lock的第一句，就是禁止内核抢占……

## 内核是如何维持墙钟和单调时钟的？系统睡眠唤醒对单调时钟是否有影响？设置系统时间对单调时钟是 否有影响？
## Linux中的高精度定时器和低精度定时器分别是如何组织的？
## malloc()分配的物理内存是连续的吗，kmalloc()呢，为什么？
malloc()函数是对进程中的堆（一个特殊的线性区）进行内存分配的函数。
malloc(size)请求size个字节的动态内存。如果分配成功，就返回所分配内存单元第一个字节的线性地址。
malloc()分配的内存的线性地址是连续的，在同一页框内的物理地址也是连续的，但是不同页之间的物理地址不一定连续。

kmalloc()分配的物理内存是连续的。因为调用kmalloc()函数实际得到的是普通高速缓存中的对象，它们具有几何分布的大小。
## 缺页异常处理的过程是怎样的？ 
## echo m > /proc/sysrq-trigger可以将内存信息比较详细的输出到dmesg中，请解读 
/proc/sysrq-trigger文件通过/drivers/tty/sysrq.c文件生成。
首先是模块注册。
`
static int __init sysrq_init(void)
{
	sysrq_init_procfs();

	if (sysrq_on())
		sysrq_register_handler();

	return 0;
}
module_init(sysrq_init);
`
然后看sysrq_init_procfs()如何实现。如果带参数CONFIG_PROC_FS，那么就初始化，否则不做任何事情。

`
#ifdef CONFIG_PROC_FS

static ssize_t write_sysrq_trigger(struct file *file, const char __user *buf,
				   size_t count, loff_t *ppos)
{
	if (count) {
		char c;

		if (get_user(c, buf))
			return -EFAULT;
		__handle_sysrq(c, false);
	}

	return count;
}

static const struct file_operations proc_sysrq_trigger_operations = {
	.write		= write_sysrq_trigger,
	.llseek		= noop_llseek,
};

static void sysrq_init_procfs(void)
{
	//生成/proc/sysrq-trigger文件，并定义相关操作。主要是write。
	if (!proc_create("sysrq-trigger", S_IWUSR, NULL,
			 &proc_sysrq_trigger_operations))
		pr_err("Failed to register proc interface\n");
}

#else

static inline void sysrq_init_procfs(void)
{
}

#endif /* CONFIG_PROC_FS */
`
在write_sysrq_trigger()中，调用了__handle_sysrq，该函数即为核心函数。其中最重要的是
op_p = __sysrq_get_key_op(key);
与 op_p->handler(key);

`

void __handle_sysrq(int key, bool check_mask)
{
	struct sysrq_key_op *op_p;
	int orig_log_level;
	int i;

	rcu_sysrq_start();
	rcu_read_lock();
	/*
	 * Raise the apparent loglevel to maximum so that the sysrq header
	 * is shown to provide the user with positive feedback.  We do not
	 * simply emit this at KERN_EMERG as that would change message
	 * routing in the consumers of /proc/kmsg.
	 */
	orig_log_level = console_loglevel;
	console_loglevel = 7;
	printk(KERN_INFO "SysRq : ");

        op_p = __sysrq_get_key_op(key);
        if (op_p) {
		/*
		 * Should we check for enabled operations (/proc/sysrq-trigger
		 * should not) and is the invoked operation enabled?
		 */
		if (!check_mask || sysrq_on_mask(op_p->enable_mask)) {
			printk("%s\n", op_p->action_msg);
			console_loglevel = orig_log_level;
			op_p->handler(key);
		} else {
			printk("This sysrq operation is disabled.\n");
		}
	} else {
		printk("HELP : ");
		/* Only print the help msg once per handler */
		for (i = 0; i < ARRAY_SIZE(sysrq_key_table); i++) {
			if (sysrq_key_table[i]) {
				int j;

				for (j = 0; sysrq_key_table[i] !=
						sysrq_key_table[j]; j++)
					;
				if (j != i)
					continue;
				printk("%s ", sysrq_key_table[i]->help_msg);
			}
		}
		printk("\n");
		console_loglevel = orig_log_level;
	}
	rcu_read_unlock();
	rcu_sysrq_end();
}
`

__sysrq_get_key_op通过key得到相应的sysrq_key_op，其中的关键是sysrq_key_table这个数组。


`
struct sysrq_key_op *__sysrq_get_key_op(int key)
{
        struct sysrq_key_op *op_p = NULL;
        int i;

	i = sysrq_key_table_key2index(key);
	if (i != -1)
	        op_p = sysrq_key_table[i];

        return op_p;
}
`

sysrq_key_table的定义如下：

`
static struct sysrq_key_op *sysrq_key_table[36] = {
	&sysrq_loglevel_op,		/* 0 */
	&sysrq_loglevel_op,		/* 1 */
	&sysrq_loglevel_op,		/* 2 */
	&sysrq_loglevel_op,		/* 3 */
	&sysrq_loglevel_op,		/* 4 */
	&sysrq_loglevel_op,		/* 5 */
	&sysrq_loglevel_op,		/* 6 */
	&sysrq_loglevel_op,		/* 7 */
	&sysrq_loglevel_op,		/* 8 */
	&sysrq_loglevel_op,		/* 9 */

	/*
	 * a: Don't use for system provided sysrqs, it is handled specially on
	 * sparc and will never arrive.
	 */
	NULL,				/* a */
	&sysrq_reboot_op,		/* b */
	&sysrq_crash_op,		/* c & ibm_emac driver debug */
	&sysrq_showlocks_op,		/* d */
	&sysrq_term_op,			/* e */
	&sysrq_moom_op,			/* f */
	/* g: May be registered for the kernel debugger */
	NULL,				/* g */
	NULL,				/* h - reserved for help */
	&sysrq_kill_op,			/* i */
#ifdef CONFIG_BLOCK
	&sysrq_thaw_op,			/* j */
#else
	NULL,				/* j */
#endif
	&sysrq_SAK_op,			/* k */
#ifdef CONFIG_SMP
	&sysrq_showallcpus_op,		/* l */
#else
	NULL,				/* l */
#endif
	&sysrq_showmem_op,		/* m */
	&sysrq_unrt_op,			/* n */
	/* o: This will often be registered as 'Off' at init time */
	NULL,				/* o */
	&sysrq_showregs_op,		/* p */
	&sysrq_show_timers_op,		/* q */
	&sysrq_unraw_op,		/* r */
	&sysrq_sync_op,			/* s */
	&sysrq_showstate_op,		/* t */
	&sysrq_mountro_op,		/* u */
	/* v: May be registered for frame buffer console restore */
	NULL,				/* v */
	&sysrq_showstate_blocked_op,	/* w */
	/* x: May be registered on ppc/powerpc for xmon */
	/* x: May be registered on sparc64 for global PMU dump */
	NULL,				/* x */
	/* y: May be registered on sparc64 for global register dump */
	NULL,				/* y */
	&sysrq_ftrace_dump_op,		/* z */
};
`
可以看出，如果当输入为‘m’时，op_p = &sysrq_showmem_op。

`
、static void sysrq_handle_showmem(int key)
{
	show_mem(0);
}
static struct sysrq_key_op sysrq_showmem_op = {
	.handler	= sysrq_handle_showmem,
	.help_msg	= "show-memory-usage(m)",
	.action_msg	= "Show Memory",
	.enable_mask	= SYSRQ_ENABLE_DUMP,
};
`

show_mem()函数定义在lib/show_mem.c文件中，具体实现如下：
`

void show_mem(unsigned int filter)
{
	pg_data_t *pgdat;
	unsigned long total = 0, reserved = 0, highmem = 0;

	printk("Mem-Info:\n");
	show_free_areas(filter);

	if (filter & SHOW_MEM_FILTER_PAGE_COUNT)
		return;

	for_each_online_pgdat(pgdat) {
		unsigned long flags;
		int zoneid;

		pgdat_resize_lock(pgdat, &flags);
		for (zoneid = 0; zoneid < MAX_NR_ZONES; zoneid++) {
			struct zone *zone = &pgdat->node_zones[zoneid];
			if (!populated_zone(zone))
				continue;

			total += zone->present_pages;
			reserved += zone->present_pages - zone->managed_pages;

			if (is_highmem_idx(zoneid))
				highmem += zone->present_pages;
		}
		pgdat_resize_unlock(pgdat, &flags);
	}

	printk("%lu pages RAM\n", total);
	printk("%lu pages HighMem/MovableOnly\n", highmem);
	printk("%lu pages reserved\n", reserved);
#ifdef CONFIG_QUICKLIST
	printk("%lu pages in pagetable cache\n",
		quicklist_total_size());
#endif
}
`

## ps可以看到内核线程的内存使用都是0，为什么？
通过man ps可以找到以下关于%mem的信息：
ratio of the process's resident set size  to the physical memory on the machine,  expressed as a percentage.  (alias pmem)
进程的驻留集大小(resident set size，RSS）与计算机上物理内存的比率，以百分比表示。 （别名pmem）
关于RSS：
RSS is the Resident Set Size and is used to show **how much memory is allocated to that process** and is **in RAM**. It does not include memory that is swapped out. It does include memory from shared libraries as long as the pages from those libraries are actually in memory. It does include all stack and heap memory.
内核线程不使用用户空间，而它使用的内核空间的内存，其实是所有进程共享的，不算进RSS里。

## top命令中看到的CPU核在各种状态的百分比是如何计算出来的？
man top

`
Line 2 shows CPU state percentages based on the interval since the last refresh.

       As  a  default,  percentages for these individual categories are displayed.  Where two labels are shown below,
       those for more recent kernel versions are shown first.
           us, user    : time running un-niced user processes
           sy, system  : time running kernel processes
           ni, nice    : time running niced user processes
           id, idle    : time spent in the kernel idle handler
           wa, IO-wait : time waiting for I/O completion
           hi : time spent servicing hardware interrupts
           si : time spent servicing software interrupts
           st : time stolen from this vm by the hypervisor
`
## gdb -p process_A_pid 跟踪上进程A后可以修改进程A中变量的值，它是如何做到的？ 
gdb的实现基础是ptrace系统调用。

 ptrace 系统调用是用于进程跟踪的, 当进程调用了 ptrace 跟踪某个进程之后:
1. 调用 ptrace 的进程会变成被跟踪进程的父进程;
2. 被跟踪进程的进程状态被标记为 TASK_TRACED;
3. 发送给被跟踪子进程的信号 (SIGKILL 除外) 会被转发给父进程, 而子进程会被阻塞;
4. 父进程收到信号后, 可以对子进程进行检查和修改, 然后让子进程继续执行;

在 man ptrace 中可以找到 ptrace 的定义原型:（man ptrace失败，为什么？）
`
#include <sys/ptrace.h>
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
`
其中 request 参数指定了我们要使用 ptrace 的什么功能, 大致可以分为以下几类:
`
PTRACE_ATTACH 或 PTRACE_TRACEME 建立进程间的跟踪关系;
PTRACE_TRACEME 是被跟踪子进程调用的, 表示让父进程来跟踪自己, 通常是通过 GDB 启动新进程的时候使用;
PTRACE_ATTACH 是父进程调用 attach 到已经运行的子进程中; 这个命令会有权限的检查, non-root 的进程不能 attach 到 root 进程中;
PTRACE_PEEKTEXT, PTRACE_PEEKDATA, PTRACE_PEEKUSR 等读取子进程内存/寄存器中保留的值;
PTRACE_POKETEXT, PTRACE_POKEDATA, PTRACE_POKEUSR 等修改被跟踪进程的内存/寄存器;
PTRACE_CONT，PTRACE_SYSCALL, PTRACE_SINGLESTEP 控制被跟踪进程以何种方式继续运行;
PTRACE_SYSCALL 会让被调用进程在每次 进入/退出 系统调用时都触发一次 SIGTRAP; strace 就是通过调用它来实现的, 在每次进入系统调用的时候读取出系统调用参数, 在退出系统调用的时候读取出返回值;
PTRACE_SINGLESTEP 会在每执行完一条指令后都触发一次 SIGTRAP; GDB 的 nexti, next 命令都是通过它来实现的;
PTRACE_DETACH, PTRACE_KILL 脱离进程间的跟踪关系;
`
当父进程在子进程之前结束时, trace 关系会被自动解除；
参数 pid 表示的是要跟踪进程的 pid, addr 表示要监控的被跟踪子进程的地址.

修改进程内变量是通过传递PTRACE_PEEKDATA参数实现的。
在include/uapi/linux/ptrace.h文件中可以找到该参数的定义：#define PTRACE_POKEDATA   5
然后寻找一下引用它的语句，可以在kernel/ptrace.c中的ptrace_request()函数中找到：

`
int ptrace_request(struct task_struct *child, long request,
		   unsigned long addr, unsigned long data)
{
	switch (request) {
	case PTRACE_POKEDATA:
		return generic_ptrace_pokedata(child, addr, data);	
`

generic_ptrace_pokedata()函数实现如下：

`
int generic_ptrace_pokedata(struct task_struct *tsk, unsigned long addr,
			    unsigned long data)
{
	int copied;

	copied = access_process_vm(tsk, addr, &data, sizeof(data), 1);
	return (copied == sizeof(data)) ? 0 : -EIO;
}
`
access_process_vm(tsk, addr, &data, sizeof(data), 1)是核心。

/*
 * Access another process' address space.
 * Source/target buffer must be kernel space,
 * Do not walk the page table directly, use get_user_pages
 */

`
int access_process_vm(struct task_struct *tsk, unsigned long addr,
		void *buf, int len, int write)
{
	struct mm_struct *mm;
	int ret;

	mm = get_task_mm(tsk);
	if (!mm)
		return 0;

	ret = __access_remote_vm(tsk, mm, addr, buf, len, write);
	mmput(mm);

	return ret;
}
`
在__access_remote_vm()中，get_user_pages()得到相应的在用户空间的页描述符地址，然后copy_to_user_page()函数实现将buf复制到该进程用户空间的目标页的目标地址，从而实现修改变量。

`
static int __access_remote_vm(struct task_struct *tsk, struct mm_struct *mm,
		unsigned long addr, void *buf, int len, int write)
{
	struct vm_area_struct *vma;
	void *old_buf = buf;

	down_read(&mm->mmap_sem);
	/* ignore errors, just check how much was successfully transferred */
	while (len) {
		int bytes, ret, offset;
		void *maddr;
		struct page *page = NULL;

		ret = get_user_pages(tsk, mm, addr, 1,
				write, 1, &page, &vma);
		if (ret <= 0) {
			/*
			 * Check if this is a VM_IO | VM_PFNMAP VMA, which
			 * we can access using slightly different code.
			 */
#ifdef CONFIG_HAVE_IOREMAP_PROT
			vma = find_vma(mm, addr);
			if (!vma || vma->vm_start > addr)
				break;
			if (vma->vm_ops && vma->vm_ops->access)
				ret = vma->vm_ops->access(vma, addr, buf,
							  len, write);
			if (ret <= 0)
#endif
				break;
			bytes = ret;
		} else {
			bytes = len;
			offset = addr & (PAGE_SIZE-1);
			if (bytes > PAGE_SIZE-offset)
				bytes = PAGE_SIZE-offset;

			maddr = kmap(page);
			if (write) {
				copy_to_user_page(vma, page, addr,
						  maddr + offset, buf, bytes);
				set_page_dirty_lock(page);
			} else {
				copy_from_user_page(vma, page, addr,
						    buf, maddr + offset, bytes);
			}
			kunmap(page);
			page_cache_release(page);
		}
		len -= bytes;
		buf += bytes;
		addr += bytes;
	}
	up_read(&mm->mmap_sem);

	return buf - old_buf;
}
`


## 系统调用的流程是怎样的？

## x86上有哪些系统调用的方式？当前x86_64使用的是哪种系统调用方式？ 
## 主流的N路组相连的CPU cache是怎么回事？
10.cache的映射
主存与cache的地址映射方式有全相联方式、直接方式和组相联方式三种。
直接映射
将一个主存块存储到唯一的一个Cache行。

多对一的映射关系，但一个主存块只能拷贝到cache的一个特定行位置上去。
cache的行号i和主存的块号j有如下函数关系：i=j mod m（m为cache中的总行数）

优点：硬件简单，容易实现
缺点：命中率低， Cache的存储空间利用率低

全相联映射
可以将一个主存块存储到任意一个Cache行。
主存的一个块直接拷贝到cache中的任意一行上

优点：命中率较高，Cache的存储空间利用率高
缺点：线路复杂，成本高，速度低

组相联映射
可以将一个主存块存储到唯一的一个Cache组中任意一个行。
将cache分成u组，每组v行，主存块存放到哪个组是固定的，至于存到该组哪一行是灵活的，即有如下函数关系：cache总行数m＝u×v 组号q＝j mod u

组间采用直接映射，组内为全相联
硬件较简单，速度较快，命中率较高

[cache](https://blog.csdn.net/yhb1047818384/article/details/79604976)
## X86的CPU有哪些主要的cache？哪些是socket共享的，哪些是core共享的？

以下回答主要来自[这篇文章](https://zhuanlan.zhihu.com/p/109206967)

 对目前主流的x86平台，CPU的缓存（cache）分为L1(Low level cache)，L2(Middle Level cache)，L3或者LLC（last level cache）总共3级。
 可以用lscpu命令查看相关信息。 
 ![!image](https://pic1.zhimg.com/80/v2-5115b8a97d5ee147c1ad67e8a222fab0_720w.jpg)
 
 L1分为L1i和L1di,i指的是instruction指令缓存，d是数据data缓存。
 
 作为冯·诺依曼体系的计算机，x86价格的指令和数据在内存中是统一管理的。但由于两者内容访问特性的不同（指令刷新率更低且不会被复写），L1的缓存是做了区分的。
 
 当前的Intel平台中L1缓存的时延为3个时钟周期，以2.0GHz的CPU计算约1.5纳秒。这种级别的时延可以极大的加速超线程以及CPU分支预测带来的性能优势。

L2缓存的时延是L1的5倍左右，即8ns。每个CPU的物理核心都有自己独立的L2缓存空间。

L3的时延在50～70个时钟周期，30ns。不同于L2，L3缓存是由同一个socket上的所有物理core共享。L3在使用场景中最大的用途是减少数据回写内存的频率，加速多核心之间的数据同步。

## cache一致性协议
