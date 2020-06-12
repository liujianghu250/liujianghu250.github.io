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
内核维持着一组自己使用的页表，驻留在所谓的主内核页全局目录中。主内核页全局目录的最高目录项部分作为参考模型，位系统中每个普通进程对于的页全局目录项提供参考模型。

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

首先看看创建新进程时，内核是如何为新进程分配页全局目录的。

也就是看看pgd_alloc()函数的具体实现：

`
static inline pgd_t *pgd_alloc(struct mm_struct *mm)
{
	unsigned boundary;
	pgd_t *pgd = (pgd_t *)__get_free_page(GFP_KERNEL|__GFP_REPEAT);
	if (!pgd)
		return NULL;
	/*
	 * Copy kernel pointers in from init.
	 * Could keep a freelist or slab cache of those because the kernel
	 * part never changes.
	 */
	boundary = pgd_index(__PAGE_OFFSET);
	memset(pgd, 0, boundary * sizeof(pgd_t));
	memcpy(pgd + boundary,
	       init_level4_pgt + boundary,
	       (PTRS_PER_PGD - boundary) * sizeof(pgd_t));
	return pgd;
}
`

重点应该是__get_free_page，但是暂时没看懂，后面再来看。

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

## IDT表是如何被初始化的？

### IDT的第一次初始化

当计算机还运行在实模式时，IDT被初始化并由BIOS例程使用。这部分由BIOS负责，暂时不深入了解。-

#### 预初始化

一旦Linux接管，IDT就被转移到RAM的另一个区域，并进行第二次初始化。

如果是x86_64，在x86_64_start_kernel()函数中可以看到以下语句：

`
	for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
		set_intr_gate(i, early_idt_handler_array[i]);
	load_idt((const struct desc_ptr *)&idt_descr);
`

循环内部调用了 set_intr_gate ，它接受两个参数：

中断号，即 向量号；

中断处理程序的地址。

同时，这个函数还会将中断门插入至 IDT 表中，代码中的 &idt_descr 数组即为 IDT。

`
early_idt_handler_array 数组，它定义在 arch/x86/include/asm/segment.h 头文件中，包含了前32个异常处理程序的地址：
#define EARLY_IDT_HANDLER_SIZE   9
#define NUM_EXCEPTION_VECTORS    32

extern const char early_idt_handler_array[NUM_EXCEPTION_VECTORS][EARLY_IDT_HANDLER_SIZE];
`

early_idt_handler_array 是一个大小为 288 字节的数组，每一项为 9 个字节，其中2个字节的备用指令用于向栈中压入默认错误码（如果异常本身没有提供错误码的话），2个字节的指令用于向栈中压入向量号，剩余5个字节用于跳转到异常处理程序。


我们只通过一个循环向 IDT 中填入了前32项内容，这是因为在整个初期设置阶段，中断是禁用的。

early_idt_handler_array 数组中的每一项指向的都是同一个通用中断处理程序early_idt_handler_common ，定义在 arch/x86/kernel/head_64.S 

它是一个初期中段处理程序，只处理缺页中断。

### 用有意义的陷阱和中断处理程序替换early_idt_handler_common 。

比如，初始化阶段，early_trap_init()会被调用。该函数的主要作用是初始化调试功能 （#DB（debug） -当 TF 标志位和rflags被设置时会被使用）和 int3 （#BP（breakpoint））中断门，还有页错误。 

`
void __init early_trap_init(void)
{
	set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
	/* int3 can be called from all */
	set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
#ifdef CONFIG_X86_32
	set_intr_gate(X86_TRAP_PF, page_fault);
#endif
	load_idt(&idt_descr);
}
`

该函数调用层次如下：start_kernel() -> setup_arch（）-> early_trap_init()


一旦所有的处理程序都被替换完成，对控制单元产生的每个不同的异常，IDT都有一个专门的陷阱或系统门，而对于可编程中断控制器确认的每一个IRQ，IDT都将包括一个专门的中断门。

使用set_intr_gate（）插入用户程序无法访问的中断门，set_system_gate（）插入系统门

set_system_intr_gate（）插入系统中断门，set_trap_gate（）插入陷阱门……

## 说一下内核处理中断的完整过程？

确定向量，读idt表，找到中断处理程序，保存与装载寄存器值，跳转到中断处理程序，中断处理完成后，恢复寄存器值，控制权转交给被中断的进程。

执行中断处理程序时，Linux中，紧随中断要执行的操作有三类：

1. 禁止可屏蔽中断后执行的操作，要在一个中断处理程序中立即执行，如对PIC应答中断，对PIC或设备控制器编程，或者修改由设备和处理器同时访问的数据结构。

2. 开中断情况下，需要很快完成的操作。如，修改只有处理器才会访问的数据结构。

3. 可以被延迟较长的时间间隔，由独立的函数执行的操作，如把缓冲区的内容拷贝到某个进程的地址空间。


### I/O中断

在PCI总线的体系结构中，几个设备可以共享同一个IRQ线，而一个中断向量是对应一个IRQ线，而不是设备。

所以仅仅中断向量无法说明所有问题，I/O中断处理程序必须足够灵活以给多个设备同时提供服务。

中断处理程序的灵活性是以两种不同的方式实现的。
#### IRQ共享

中断处理程序执行多个中断服务例程（ISR）。每个ISR是一个与单独设备（共享IRQ线）相关的函数，因为不可能预先知道是哪个特定的设备产生IRQ，所以，每个ISR都被执行，以验证它的设备是否需要关注。如果是，当设备产生中断时，就执行需要执行的所有操作。

#### IRQ动态分配
一条IRQ线在可能的最后时刻才与一个设备驱动程序相关联，例如，软盘设备的IRQ线只有在用户访问软盘设备时才被分配。

### 处理器间中断

处理器间中断允许一个处理器向其他处理器发送中断信号。处理器间中断（IPI）不是通过IRQ传输的，而是作为信号直接放在连接所有CPU本地APIC的总线上。

Linux定义了三种处理器间中断

#### CALL_FUNCTION_VECTOR(向量0xfb）
发往所有的CPU（不包括发送者），强制这些CPU运行发送者传递过来的函数。

例如，地址存放在全局变量call_data中来传递的函数，可能强制其他所有CPU都停止，也可能强制它们设置内存类型范围寄存器的内容。

#### RESCHEDULE_VECTOR(向量0xfc)
当一个CPU接收到这种类型的中断时，相应的处理程序（reschedule_interrupt()）限定自己来应答中断，当从中断返回时，所有的重新调度自动执行。

#### INVALIDATE_TLB_VECTOR(向量0xfd)
发往所有的CPU（不包括发送者），强制它们的转换后援缓冲器（TLB）变为无效。

### 时钟中断

## 网卡的中断服务程序正在执行过程中会被新的网卡中断打断吗？


## Linux支持哪些中断下半部，它们的区别是什么？ 
三种，分别是软中断、tasklet、工作队列
### 软中断和tasklet
软中断和tasklet都是可延迟函数，两者有密切的关系，tasklet是在软中断之上实现的（有时候软中断包括tasklet）。

但软中断的分配是静态的，即在编译时定义。

而tasklet的分配和初始化可以在运行时进行（例如安装一个内核模块时）。

软中断可以并发地运行在多个CPU中，但相同类型的tasklet总是被串行地执行，也就是说，不能在两个CPU上同时运行相同类型的tasklet。

软中断是可重入函数而且必须明确地使用自旋锁保护其数据结构，而tasklet不必是可重入的。

由于软中断必须使用可重入函数，这就导致设计上的复杂度变高，作为设备驱动程序的开发者来说，增加了负担。如果不需要软中断的并行特性，tasklet就是最好的选择。

### 软中断和工作队列
可延迟函数（包括软中断和tasklet）运行在中断上下文中，于是导致了一些问题：软中断不能睡眠、不能阻塞。由于中断上下文出于内核态，没有进程切换，所以如果软中断一旦睡眠或者阻塞，将无法退出这种状态，导致内核会整个僵死。但可阻塞函数不能用在中断上下文中实现，必须要运行在进程上下文中，例如访问磁盘数据块的函数。因此，可阻塞函数不能用软中断来实现。但是它们往往又具有可延迟的特性。

因此在2.6版的内核中出现了在内核态运行的工作队列（替代了2.4内核中的任务队列）。它也具有一些可延迟函数的特点（需要被激活和延后执行），但是能够能够在不同的进程间切换，以完成不同的工作。

## 对/proc/interrupts和/proc/softirqs进行解读
### /proc/interrupts 
/proc/interrupts 是一个 sequence file.，简单来说，就是通过代码生成的文件。

关于sequence file的详细信息，见这里：http://www.tldp.org/LDP/lkmpg/2.6/html/x861.html


对于/proc/interrupts,在读取这个文件时，系统会遍历 0 ～ nr_irq (包含 nr_irq)个中断号，对每个中断号都调用 show_interrupts() 来获取该中断的信息。

对于每个中断，会依次显示irq的序号， 在各自cpu上发生中断的次数，可编程中断控制器，设备名称（request_irq的dev_name字段）

`

int show_interrupts(struct seq_file *p, void *v)
{
	static int prec;

	unsigned long flags, any_count = 0;
	int i = *(loff_t *) v, j;
	struct irqaction *action;
	struct irq_desc *desc;

	if (i > ACTUAL_NR_IRQS)
		return 0;

	if (i == ACTUAL_NR_IRQS)
		return arch_show_interrupts(p, prec);

	/* print header and calculate the width of the first column */
	if (i == 0) {
		for (prec = 3, j = 1000; prec < 10 && j <= nr_irqs; ++prec)
			j *= 10;

		seq_printf(p, "%*s", prec + 8, "");
		for_each_online_cpu(j)
			seq_printf(p, "CPU%-8d", j);
		seq_putc(p, '\n');
	}

	desc = irq_to_desc(i);
	if (!desc)
		return 0;

	raw_spin_lock_irqsave(&desc->lock, flags);
	for_each_online_cpu(j)
		any_count |= kstat_irqs_cpu(i, j);
	action = desc->action;
	if (!action && !any_count)
		goto out;

	seq_printf(p, "%*d: ", prec, i);
	for_each_online_cpu(j)
		seq_printf(p, "%10u ", kstat_irqs_cpu(i, j));

	if (desc->irq_data.chip) {
		if (desc->irq_data.chip->irq_print_chip)
			desc->irq_data.chip->irq_print_chip(&desc->irq_data, p);
		else if (desc->irq_data.chip->name)
			seq_printf(p, " %8s", desc->irq_data.chip->name);
		else
			seq_printf(p, " %8s", "-");
	} else {
		seq_printf(p, " %8s", "None");
	}
#ifdef CONFIG_GENERIC_IRQ_SHOW_LEVEL
	seq_printf(p, " %-8s", irqd_is_level_type(&desc->irq_data) ? "Level" : "Edge");
#endif
	if (desc->name)
		seq_printf(p, "-%-8s", desc->name);

	if (action) {
		seq_printf(p, "  %s", action->name);
		while ((action = action->next) != NULL)
			seq_printf(p, ", %s", action->name);
	}

	seq_putc(p, '\n');
out:
	raw_spin_unlock_irqrestore(&desc->lock, flags);
	return 0;
}
#endif
`


           CPU0       CPU1       CPU2       CPU3
  0:         77          0          0          0   IO-APIC-edge      timer
  1:         10          0          0          0   IO-APIC-edge      i8042
  4:        360          0          0          0   IO-APIC-edge      serial
  6:          3          0          0          0   IO-APIC-edge      floppy
  8:          0          0          0          0   IO-APIC-edge      rtc0
  9:          0          0          0          0   IO-APIC-fasteoi   acpi
 11:          0          0          0          0   IO-APIC-fasteoi   uhci_hcd:usb1, virtio3
 12:         15          0          0          0   IO-APIC-edge      i8042
 14:      53215          0          0          0   IO-APIC-edge      ata_piix
 15:          0          0          0          0   IO-APIC-edge      ata_piix
 24:          0          0          0          0   PCI-MSI-edge      virtio1-config
 25:     211536          0          0          0   PCI-MSI-edge      virtio1-req.0
 26:          0          0          0          0   PCI-MSI-edge      virtio0-config
 27:     150065          0          0          0   PCI-MSI-edge      virtio0-input.0
 28:          1          0          0          0   PCI-MSI-edge      virtio0-output.0
 29:          2     138324          0          0   PCI-MSI-edge      virtio0-input.1
 30:          1          0          0          0   PCI-MSI-edge      virtio0-output.1
 31:          0          0          0          0   PCI-MSI-edge      virtio2-config
 32:        273          0          0          0   PCI-MSI-edge      virtio2-req.0
NMI:          0          0          0          0   Non-maskable interrupts
LOC:    5718362    5821125    5866926    5722651   Local timer interrupts
SPU:          0          0          0          0   Spurious interrupts
PMI:          0          0          0          0   Performance monitoring interrupts
IWI:     101565      73773      65567      74876   IRQ work interrupts
RTR:          0          0          0          0   APIC ICR read retries
RES:     570831     614110     641190     601586   Rescheduling interrupts
CAL:      32238      51419      80493      62383   Function call interrupts
TLB:      61163      73326      73073      74288   TLB shootdowns
TRM:          0          0          0          0   Thermal event interrupts
THR:          0          0          0          0   Threshold APIC interrupts
DFR:          0          0          0          0   Deferred Error APIC interrupts
MCE:          0          0          0          0   Machine check exceptions
MCP:        181        181        181        181   Machine check polls
ERR:          0
MIS:          0
PIN:          0          0          0          0   Posted-interrupt notification event
PIW:          0          0          0          0   Posted-interrupt wakeup event

参考博客：https://feichashao.com/proc-interrupts/

### /proc/softirqs

而/proc/softirqs，也是一个sequence file。

生成该文件的代码是fs/proc/softirq.c,具体如下：

`

#include <linux/init.h>
#include <linux/kernel_stat.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>


static int show_softirqs(struct seq_file *p, void *v)
{
	int i, j;

	seq_puts(p, "                    ");
	for_each_possible_cpu(i)
		seq_printf(p, "CPU%-8d", i);
	seq_putc(p, '\n');

	for (i = 0; i < NR_SOFTIRQS; i++) {
		seq_printf(p, "%12s:", softirq_to_name[i]);
		for_each_possible_cpu(j)
			seq_printf(p, " %10u", kstat_softirqs_cpu(i, j));
		seq_putc(p, '\n');
	}
	return 0;
}

static int softirqs_open(struct inode *inode, struct file *file)
{
	return single_open(file, show_softirqs, NULL);
}

static const struct file_operations proc_softirqs_operations = {
	.open		= softirqs_open,
	.read		= seq_read,
	.llseek		= seq_lseek,
	.release	= single_release,
};

static int __init proc_softirqs_init(void)
{
	proc_create("softirqs", 0, NULL, &proc_softirqs_operations);
	return 0;
}
module_init(proc_softirqs_init);
`

重点是show_softirqs()函数，可以看出，对于每个软中断，它向文件输出了softirq_to_name，也就是软中断的名字，还调用了kstat_softirqs_cpu（），看了很久的代码，大概看出这是在查询一个每CPU变量的值，softirqs[irq]的值。unsigned int softirqs[NR_SOFTIRQS]是struct kernel_stat 中的成员，猜测它是irq（软中断号）对应的软中断在特定CPU上发生的次数。


                    CPU0       CPU1       CPU2       CPU3
          HI:          1          0          0          0
       TIMER:    1326665    1168106    1275189    1152706
      NET_TX:          0          0          0          0
      NET_RX:     434110     428897     104943      81492
       BLOCK:      34647          0          0          0
BLOCK_IOPOLL:          0          0          0          0
     TASKLET:         16          0          0          0
       SCHED:     350968     316695     441771     296038
     HRTIMER:          0          0          0          0
         RCU:     875246     787096     816726     770378
	 
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

## 内核是如何维持墙钟和单调时钟的？系统睡眠唤醒对单调时钟是否有影响？设置系统时间对单调时钟是 否有影响？ Linux中的高精度定时器和低精度定时器分别是如何组织的？ malloc()分配的物理内存是连续的吗，kmalloc()呢，为什么？
## 缺页异常处理的过程是怎样的？ 
## echo m > /proc/sysrq-trigger可以将内存信息比较详细的输出到dmesg中，请解读 ps可以看到内核线程的内存使用都是0，为什么？
## top命令中看到的CPU核在各种状态的百分比是如何计算出来的？
## gdb -p process_A_pid 跟踪上进程A后可以修改进程A中变量的值，它是如何做到的？ 
## 系统调用的流程是怎样的？
## x86上有哪些系统调用的方式？当前x86_64使用的是哪种系统调用方式？ 
## 主流的N路组相连的CPU cache是怎么回事？
## X86的CPU有哪些主要的cache？哪些是socket共享的，哪些是core共享的？
 

