
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
	 
