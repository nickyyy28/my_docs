---
title: 嵌入式Linux-中断子系统
date: 2023-08-12 23:19:43
categories:
  - 嵌入式开发
tags:
  - Linux内核
  - ARM
  - 中断
---

## 1. 中断系统

**CPU在运行时有一些操作是不定时的, 例如用户是否按下按钮, 是否触摸屏幕, 这些操作cpu无法预知什么时候来临, 所以, 为了及时读取这些信息, 产生了三种方法:**

* **阻塞: CPU不知道什么时候事件会来, 所以一直停止运行, 等待事件到来**
* **轮询: CPU执行自己的程序, 每隔一段时间查看是否有事件到来**
* **中断: CPU一直执行自己的程序, 直到外部事件到来通知CPU, CPU停止执行当前程序去执行中断服务函数**

**上面三种处理方式中, 第一种几乎不会使用, 完全没有效率, 占用CPU资源, 无法执行其他程序, 第二种也很少使用, 当外部事件数量多的时候就要花大量的时间去查询, 浪费CPU时间, 现在主流都是用第三种方式: 中断**

**Linux在不断的演化中, 逐渐形成了一套中断系统, 通过Linux系统的api, 用户可以方便的定义中断**

## 2. GIC(generic interrupt controller)

<div align="center"><i>注: 部分内容引用自<a href="https://zhuanlan.zhihu.com/p/443815199" >https://zhuanlan.zhihu.com/p/443815199</a></i></div>

**GIC 是 ARM 公司给 Cortex-A/R 内核提供的一个中断控制器，类似 Cortex-M 内核（STM32）中的 NVIC。目前 GIC 有 4 个版本:V1~V4，V1 是最老的版本，已经被废弃了。V2~V4 目前正在大量的使用。GIC V2 是给 ARMv7-A 架构使用的，比如 Cortex-A7、Cortex-A9、Cortex-A15 等， V3 和 V4 是给 ARMv8-A/R 架构使用的，也就是 64 位芯片使用的。**

**GIC控制器与CPU和其他外设相连的拓扑图如下:**

![image-20221024150313954](https://pic.imgdb.cn/item/64d79f961ddac507cc889613.png)

**UART, Timer, FLASH, 等外设的数据线通过SMBA总线相连与CPU进行数据通信, 中断信息不通过该总线发送至CPU, 而是发送给与之相连的GIC控制器**

**GIC控制器根据中断的优先级, 是否使能进行仲裁后, 发送给对应的CPU**

![image-20221024152802258](https://pic.imgdb.cn/item/64d79fa81ddac507cc88ce64.png)

**左侧部分就是中断源，中间部分就是 GIC 控制器，最右侧就是中断控制器向 处理器内核发送中断信息。我们重点要看的肯定是中间的 GIC 部分，GIC 将众多的中断源分为 分为三类：**

- **1. SPI(Shared Peripheral Interrupt)，共享外设中断，该中断来自于外设，所有 Core 共享的中断。比如按键中断、串口中断等等，这些中断所有的 Core 都可以处理，不限定特定 Core。**
- **2. PPI(Private Peripheral Interrupt)，私有外设中断，该终端来自于外设，被特定的核处理。 GIC 是支持多核的，每个核有自己独有的中断。**
- **3. SGI(Software-generated Interrupt)，软中断，由软件触发引起的中断，通过向寄存器 GICD_SGIR 写入数据来触发，系统会使用 SGI 中断来完成多核之间的通信。**

**中断源有很多，为了区分这些不同的中断源肯定要给他们分配一个唯一 ID，这些 ID 就是中断 ID。**
**GIC-v2中 每一个 CPU 最多支持 1020 个中断 ID，中断 ID 号为 ID0~ID1019。这 1020 个 ID 包 含了 PPI、SPI 和 SGI。这 1020 个 ID 分 配如下：**

- ID0~ID15：这 16 个 ID 分配给 SGI。每个CPU核都有自己的16个。
- ID16~ID31：这 16 个 ID 分配给 PPI。 每个CPU核都有自己的16个。
- ID32~ID1019：这 988 个 ID 分配给 SPI，像 GPIO 中断、串口中断等这些外部中断 ，至于具体到某个 ID 对应哪个中断那就由半导体厂商根据实际情况去定义。

**GIC-v2 架构分为了两个逻辑块：Distributor 和 CPU Interface，也就是分发器端和 CPU 接口端。**

- **Distributor(分发器端)：**中间那个，此逻辑块负责处理各个中断事件的分发问题，也就是中断事件应该发送到哪个 CPU Interface 上去。分发器收集所有的中断源，可以控制每个中断的优先级，它总是将优先级最高的中断事件发送到 CPU 接口端。分发器端要做的主要 工作如下：
  - 全局中断使能控制。
  - 控制每一个中断的使能或者关闭。
  - 设置每个中断的优先级。
  - 设置每个中断的目标处理器列表。
  - 设置每个外部中断的触发模式：电平触发或边沿触发。
  - 设置每个中断属于组 0 还是组 1。
- **CPU Interface(CPU 接口端)**：CPU 接口端听名字就知道是和 CPU Core 相连接的，因此在图中每个 CPU Core 都可以在 GIC 中找到一个与之对应的 CPU Interface。CPU 接口端 就是分发器和 CPU Core 之间的桥梁，CPU 接口端主要工作如下：
  - 使能或者关闭发送到 CPU Core 的中断请求信号。
  - 应答中断。
  - 通知中断处理完成。
  - 设置优先级掩码，通过掩码来设置哪些中断不需要上报给 CPU Core。
  - 定义抢占策略。
  - 当多个中断到来的时候，选择优先级最高的中断通知给 CPU Core。

## 3. GIC控制器中断流程

**在GIC控制器中, 一个中断源的状态有四个:**

- **1. Inactive**
- **2. Pending**
- **3. Active**
- **4. Active and Pending**

### 中断处理流程

- **1. 所有的中断源状态一开始都为Inactive, GIC检测到使能的中断发生，将中断状态设为 pending**
- **2. GIC的仲裁器将最高优先级的pending中断发送到指 定的CPU interface**
- **3.  CPU interface根据配置，将中断信号发送到CPU**
- **4. CPU应答该中断，读取寄存器获取interrupt ID，GIC 更新中断状态为active**
- **5. CPU处理完中断后，发送EOI信号给GIC**

**中断状态转移图如下:**

![image-20221025111953399](https://pic.imgdb.cn/item/64d7a2d51ddac507cc91d664.png)

## 4. 裸机中断

**Cortex-A/R系列的cpu和Cortex-M系列cpu中断的定义类似, 但是也有一定的区别**

**在M系列的产品中, 类似于stm32等32位单片机的中断定义在startup.s的启动文件中, 这个文件中定义了一个中断向量表, cpu的中断控制器(NVIC)接收到中断后会根据中断号去中断向量表中查询对应的函数并跳转执行**

```assembly
__Vectors       DCD     __initial_sp               ; Top of Stack
                DCD     Reset_Handler              ; Reset Handler
                DCD     NMI_Handler                ; NMI Handler
                DCD     HardFault_Handler          ; Hard Fault Handler
                DCD     MemManage_Handler          ; MPU Fault Handler
                DCD     BusFault_Handler           ; Bus Fault Handler
                DCD     UsageFault_Handler         ; Usage Fault Handler
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                ...
                DCD     SPI6_IRQHandler            ; SPI6
                DCD     SAI1_IRQHandler            ; SAI1
                DCD      0                         ; Reserved
                DCD      0                         ; Reserved
                DCD     DMA2D_IRQHandler           ; DMA2D
                                         
__Vectors_End
```

**而在A/R系列的cpu中, 编写逻辑中断也类似需要提供一个启动文件, 在该文件定义中断向量表:**

```assembly
.section .isr_vector
_Reset:
	B reset_handler
	B .
	B c_svc
	B .
	B .
	B .
	B c_irq
	B c_fiq
.text
```

**不过和stm32不同的是, 不需要把类似于串口中断, IIC中断, DMA中断之类的具体外设的中断函数写入, 只需要提供几个不同CPU模式的中断函数, 如IRQ模式中断函数, FIQ模式,SVC模式等等, 具体的外设中断由程序员自己软件定义的跳转过程实现**

**在下面的rtc时钟例子中, 有一个函数指针数组, 该数组保存了根据中断号为下标的中断服务函数, 启动文件中的irq中断函数会去gic控制器的寄存器中查询中断发生的中断号, 然后根据中断号作为下标去中断服务函数数组中获取对应的函数并执行**

```c
//通过__attribute__((interrupt("IRQ")))告诉编译器这个是一个中断函数, 编译器会添加相关的上下文保护代码
void __attribute__ ((interrupt("IRQ"))) c_irq(void){
	asm volatile("cpsid i" : : : "memory", "cc");
	int irq_num = GIC_AcknowledgePending();	//从GIC控制器中获取中断号
	GIC_ClearPendingIRQ(irq_num);			//清除中断状态
	if(isr_table[irq_num] != NULL){			//如果在中断服务函数数组中查找到对应的函数则执行
		isr_table[irq_num]();
	}else{
		printf("no handler found for %d\n",irq_num);
	}
	
    //向GIC控制器发送中断结束信号
	GIC_EndInterrupt(irq_num);
	asm volatile("cpsie i" : : : "memory", "cc");
}
```

**中断号的定义具体到某个 ID 对应哪个中断由半导体厂商根据实际情况去定义了。在实际使用的过程中需要查询相应的芯片手册**

## 5. Linux中断介绍

**Linux中断系统和裸机中断本质上差不多, 不过为了实现更加强大的功能, 比裸机实现起来复杂得多**

### 注册中断:

**Linux内核提供了注册中断服务函数的api**

```c
//include/linux/interrupt.h

/**
 * request_irq - Add a handler for an interrupt line
 * @irq:	The interrupt line to allocate
 * @handler:	Function to be called when the IRQ occurs.
 *		Primary handler for threaded interrupts
 *		If NULL, the default primary handler is installed
 * @flags:	Handling flags
 * @name:	Name of the device generating this interrupt
 * @dev:	A cookie passed to the handler function
 *
 * This call allocates an interrupt and establishes a handler; see
 * the documentation for request_threaded_irq() for details.
 */
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
{
	return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```

### 删除中断服务函数

```c
//kernel/irq/manage.c
/**
 *	free_irq - free an interrupt allocated with request_irq
 *	@irq: Interrupt line to free
 *	@dev_id: Device identity to free
 *
 *	Remove an interrupt handler. The handler is removed and if the
 *	interrupt line is no longer in use by any driver it is disabled.
 *	On a shared IRQ the caller must ensure the interrupt is disabled
 *	on the card it drives before calling this function. The function
 *	does not return until any executing interrupts for this IRQ
 *	have completed.
 *
 *	This function must not be called from interrupt context.
 *
 *	Returns the devname argument passed to request_irq.
 */
const void *free_irq(unsigned int irq, void *dev_id)
{
	struct irq_desc *desc = irq_to_desc(irq);
	struct irqaction *action;
	const char *devname;

	if (!desc || WARN_ON(irq_settings_is_per_cpu_devid(desc)))
		return NULL;

#ifdef CONFIG_SMP
	if (WARN_ON(desc->affinity_notify))
		desc->affinity_notify = NULL;
#endif

	action = __free_irq(desc, dev_id);

	if (!action)
		return NULL;

	devname = action->name;
	kfree(action);
	return devname;
}
```

**Linux中的中断服务函数是以下格式:**

```c
//参数为一个int型的中断号, 以及void*的参数指针
typedef irqreturn_t (*irq_handler_t)(int, void *);

//返回值为irqreturn_t, 是一个枚举类型
enum irqreturn {
	IRQ_NONE		= (0 << 0),
	IRQ_HANDLED		= (1 << 0),
	IRQ_WAKE_THREAD		= (1 << 1),
};

typedef enum irqreturn irqreturn_t;
```

## 6. Linux中断流程

**当一个中断到来时, Linux内核的中断流程大致可以分为四部分**
### 驱动注册中断

```c
//一个简单的rtc时钟pl031的驱动代码
static int __init rtc_init(void)
{

    irqreturn_t ret = 0;

    regs = (rtc_reg_t *)ioremap(RTC_BASE, sizeof(rtc_reg_t)); 
    printk("rtc_init\n");

    set_rtc_alarm(regs);
    ret = request_irq(39, rtc_alarm_handler, 0, "rtc0-test", NULL);
    if (ret == -1){
        printk("request_irq failed!\n");
        return -1;
    }
   
    return 0;
}
```
<div align="center">↓</div>

```c
//include/linux/interrupt.h
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
{
	return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```

<div align="center">↓</div>

```c
//kernel/irq/manage.c
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
			 irq_handler_t thread_fn, unsigned long irqflags,
			 const char *devname, void *dev_id)
{
	struct irqaction *action;		//该结构体指针指向了发生中断时要发生的动作
	struct irq_desc *desc;			//该中断描述符指针, 这两个变量非常重要
	int retval;

	if (irq == IRQ_NOTCONNECTED)
		return -ENOTCONN;

	/*
	 * Sanity-check: shared interrupts must pass in a real dev-ID,
	 * otherwise we'll have trouble later trying to figure out
	 * which interrupt is which (messes up the interrupt freeing
	 * logic etc).
	 *
	 * Also IRQF_COND_SUSPEND only makes sense for shared interrupts and
	 * it cannot be set along with IRQF_NO_SUSPEND.
	 */
	if (((irqflags & IRQF_SHARED) && !dev_id) ||
	    (!(irqflags & IRQF_SHARED) && (irqflags & IRQF_COND_SUSPEND)) ||
	    ((irqflags & IRQF_NO_SUSPEND) && (irqflags & IRQF_COND_SUSPEND)))
		return -EINVAL;

    //根据软中断号获取中断描述符指针
	desc = irq_to_desc(irq);
	if (!desc)
		return -EINVAL;

	if (!irq_settings_can_request(desc) ||
	    WARN_ON(irq_settings_is_per_cpu_devid(desc)))
		return -EINVAL;

	if (!handler) {
		if (!thread_fn)
			return -EINVAL;
		handler = irq_default_primary_handler;
	}

    //内核空间中申请一片空间保存中断相关的处理函数
	action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
	if (!action)
		return -ENOMEM;

	action->handler = handler;		//中断处理函数
	action->thread_fn = thread_fn;	//中断线程化处理函数
	action->flags = irqflags;
	action->name = devname;
	action->dev_id = dev_id;

	retval = irq_chip_pm_get(&desc->irq_data);
	if (retval < 0) {
		kfree(action);
		return retval;
	}
	
    //真正进行注册
	retval = __setup_irq(irq, desc, action);

	if (retval) {
		irq_chip_pm_put(&desc->irq_data);
		kfree(action->secondary);
		kfree(action);
	}

#ifdef CONFIG_DEBUG_SHIRQ_FIXME
	if (!retval && (irqflags & IRQF_SHARED)) {
		/*
		 * It's a shared IRQ -- the driver ought to be prepared for it
		 * to happen immediately, so let's make sure....
		 * We disable the irq to make sure that a 'real' IRQ doesn't
		 * run in parallel with our fake.
		 */
		unsigned long flags;

		disable_irq(irq);
		local_irq_save(flags);

		handler(irq, dev_id);

		local_irq_restore(flags);
		enable_irq(irq);
	}
#endif
	return retval;
}
```

**上面的函数使用了两个比较重要的结构体**

```c
struct irq_desc {
	struct irq_common_data	irq_common_data;
	struct irq_data		irq_data;
	unsigned int __percpu	*kstat_irqs;
	irq_flow_handler_t	handle_irq;
	struct irqaction	*action;	/* IRQ action list */
	unsigned int		status_use_accessors;
	unsigned int		core_internal_state__do_not_mess_with_it;
	unsigned int		depth;		/* nested irq disables */
	unsigned int		wake_depth;	/* nested wake enables */
	unsigned int		tot_count;
	unsigned int		irq_count;	/* For detecting broken IRQs */
	unsigned long		last_unhandled;	/* Aging timer for unhandled count */
	unsigned int		irqs_unhandled;
	atomic_t		threads_handled;
	int			threads_handled_last;
	raw_spinlock_t		lock;
	struct cpumask		*percpu_enabled;
	const struct cpumask	*percpu_affinity;
#ifdef CONFIG_SMP
	const struct cpumask	*affinity_hint;
	struct irq_affinity_notify *affinity_notify;
#ifdef CONFIG_GENERIC_PENDING_IRQ
	cpumask_var_t		pending_mask;
#endif
#endif
	unsigned long		threads_oneshot;
	atomic_t		threads_active;
	wait_queue_head_t       wait_for_threads;
#ifdef CONFIG_PM_SLEEP
	unsigned int		nr_actions;
	unsigned int		no_suspend_depth;
	unsigned int		cond_suspend_depth;
	unsigned int		force_resume_depth;
#endif
#ifdef CONFIG_PROC_FS
	struct proc_dir_entry	*dir;
#endif
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS
	struct dentry		*debugfs_file;
	const char		*dev_name;
#endif
#ifdef CONFIG_SPARSE_IRQ
	struct rcu_head		rcu;
	struct kobject		kobj;
#endif
	struct mutex		request_mutex;
	int			parent_irq;
	struct module		*owner;
	const char		*name;
} ____cacheline_internodealigned_in_smp;

struct irqaction {
	irq_handler_t		handler;
	void			*dev_id;
	void __percpu		*percpu_dev_id;
	struct irqaction	*next;
	irq_handler_t		thread_fn;
	struct task_struct	*thread;
	struct irqaction	*secondary;
	unsigned int		irq;
	unsigned int		flags;
	unsigned long		thread_flags;
	unsigned long		thread_mask;
	const char		*name;
	struct proc_dir_entry	*dir;
} ____cacheline_internodealigned_in_smp;
```

**内核中也和裸机类似, 定义了一个irq_desc的数组, 用来保存中断处理函数等等**

```c
struct irq_desc irq_desc[NR_IRQS];
```

**每个中断都有一个中断描述符, 在上面的函数中构造了一个irqaction, 这个action保存了中断的中断号, 处理函数, 中断线程化处理函数, 设备名称, 设备号等等, 通过软中断号在该数组中找到对应的描述符, 然后将该desc描述符指向该irqaction**
### CPU硬件自动执行的部分

```assembly
指令1
指令2
指令3 <----PC指针
指令4
指令5
```

**当中断来时** 

- **CPU会将CPSR寄存器保存至SPSR_irq寄存器**
- **设置CPSR控制器, CPU进入ARM状态, IRQ模式**
- **CPSR中的IRQ位 置1, 硬件自动关闭IRQ**
- **将当前中断地址(返回地址)即指令4地址保存至LR_irq寄存器**
- **设置PC=0x00000018, 跳转至中断向量表**

**ARM中断向量表的位置:**

```assembly
//arch/arm/kernel/entry-armv.S
.section .vectors, "ax", %progbits
.L__vectors_start:
	W(b)	vector_rst
	W(b)	vector_und
	W(ldr)	pc, .L__vectors_start + 0x1000
	W(b)	vector_pabt
	W(b)	vector_dabt
	W(b)	vector_addrexcptn
	W(b)	vector_irq
	W(b)	vector_fiq
```

**通过vector_irq跳转到:**

```assembly
//arch/arm/kernel/entry-armv.S
/*
 * Interrupt dispatcher
 */
	vector_stub	irq, IRQ_MODE, 4

	.long	__irq_usr			@  0  (USR_26 / USR_32)
	.long	__irq_invalid			@  1  (FIQ_26 / FIQ_32)
	.long	__irq_invalid			@  2  (IRQ_26 / IRQ_32)
	.long	__irq_svc			@  3  (SVC_26 / SVC_32)
	.long	__irq_invalid			@  4
	.long	__irq_invalid			@  5
	.long	__irq_invalid			@  6
	.long	__irq_invalid			@  7
	.long	__irq_invalid			@  8
	.long	__irq_invalid			@  9
	.long	__irq_invalid			@  a
	.long	__irq_invalid			@  b
	.long	__irq_invalid			@  c
	.long	__irq_invalid			@  d
	.long	__irq_invalid			@  e
	.long	__irq_invalid			@  f
```

**该函数又跳转至__irq_usr**

```assembly
//arch/arm/kernel/entry-armv.S
__irq_usr:
	usr_entry
	kuser_cmpxchg_check
	irq_handler
	get_thread_info tsk
	mov	why, #0
	b	ret_to_user_from_irq
 UNWIND(.fnend		)
ENDPROC(__irq_usr)
```

**该函数又会跳转到irq_handler**

```assembly
//arch/arm/kernel/entry-armv.S
.macro	irq_handler
#ifdef CONFIG_GENERIC_IRQ_MULTI_HANDLER
	ldr	r1, =handle_arch_irq
	mov	r0, sp
	badr	lr, 9997f
	ldr	pc, [r1]
#else
	arch_irq_handler_default
#endif
```

**跳转至arch_irq_handler_default**

```assembly
//arch/arm/include/asm/entry-macro-multi.S
.macro	arch_irq_handler_default
	get_irqnr_preamble r6, lr
1:	get_irqnr_and_base r0, r2, r6, lr	/*获取IRQ number*/
	movne	r1, sp
	@
	@ routine called with r0 = irq number, r1 = struct pt_regs *
	@
	badrne	lr, 1b
	bne	asm_do_IRQ
```

**该函数根据HW interrupt ID获取IRQ number, 执行asm_do_IRQ**

```c
//arch/arm/kernel/irq.c
asmlinkage void __exception_irq_entry
asm_do_IRQ(unsigned int irq, struct pt_regs *regs)
{
	handle_IRQ(irq, regs);
}
```

<div align="center">↓</div>

```c
//arch/arm/kernel/irq.c
void handle_IRQ(unsigned int irq, struct pt_regs *regs)
{
	__handle_domain_irq(NULL, irq, false, regs);
}
```

<div align="center">↓</div>

```c
//kernel/irq/irqdesc.c
int __handle_domain_irq(struct irq_domain *domain, unsigned int hwirq,
			bool lookup, struct pt_regs *regs)
{
	struct pt_regs *old_regs = set_irq_regs(regs);
	unsigned int irq = hwirq;
	int ret = 0;

	irq_enter();

#ifdef CONFIG_IRQ_DOMAIN
	if (lookup)
		irq = irq_find_mapping(domain, hwirq);
#endif

	/*
	 * Some hardware gives randomly wrong interrupts.  Rather
	 * than crashing, do something sensible.
	 */
	if (unlikely(!irq || irq >= nr_irqs)) {
		ack_bad_irq(irq);
		ret = -EINVAL;
	} else {
		generic_handle_irq(irq);
	}

	irq_exit();
	set_irq_regs(old_regs);
	return ret;
}
```

<div align="center">↓</div>

```c
//kernel/irq/irqdesc.c
int generic_handle_irq(unsigned int irq)
{
    //根据中断号获取中断描述符
	struct irq_desc *desc = irq_to_desc(irq);
	struct irq_data *data;

	if (!desc)
		return -EINVAL;

	data = irq_desc_get_irq_data(desc);
	if (WARN_ON_ONCE(!in_irq() && handle_enforce_irqctx(data)))
		return -EPERM;

	generic_handle_irq_desc(desc);
	return 0;
}
```

<div align="center">↓</div>

```c
//include/linux/irqdesc.h
static inline void generic_handle_irq_desc(struct irq_desc *desc)
{
	desc->handle_irq(desc);
}
```
**这个文件描述符的handle_irq函数在系统初始化的时候进行的赋值**

### 系统初始化

**系统启动过程中, 由于有多个GIC控制器, 为了将不同GIC控制器的中断区分, Linux在系统初始化的过程中建立了硬件中断号(HW  Interrupt ID) 到软中断号的映射(IRQ number)**

```c
//drivers/irqchip/irq-gic.c
static int gic_irq_domain_map(struct irq_domain *d, unsigned int irq,
				irq_hw_number_t hw)
{
	struct gic_chip_data *gic = d->host_data;
	struct irq_data *irqd = irq_desc_get_irq_data(irq_to_desc(irq));

	switch (hw) {
	case 0 ... 15:
		irq_set_percpu_devid(irq);
		irq_domain_set_info(d, irq, hw, &gic->chip, d->host_data,
				    handle_percpu_devid_fasteoi_ipi,
				    NULL, NULL);
		break;
	case 16 ... 31:
		irq_set_percpu_devid(irq);
		irq_domain_set_info(d, irq, hw, &gic->chip, d->host_data,
				    handle_percpu_devid_irq, NULL, NULL);
		break;
	default:
		irq_domain_set_info(d, irq, hw, &gic->chip, d->host_data,
				    handle_fasteoi_irq, NULL, NULL);	//重点
		irq_set_probe(irq);
		irqd_set_single_target(irqd);
		break;
	}

	/* Prevents SW retriggers which mess up the ACK/EOI ordering */
	irqd_set_handle_enforce_irqctx(irqd);
	return 0;
}
```

**在前面提到, Linux的软中断号0-15是SGI中断, 16-31属于PPI中断, 32-1020才是编写驱动常用的共享中断**

**所以主要分析上面的irq_domain_set_info和它传递的handle_fasteoi_irq这两个函数**

#### irq_domain_set_info

```c
//kernel/irq/irqdomain.c
void irq_domain_set_info(struct irq_domain *domain, unsigned int virq,
			 irq_hw_number_t hwirq, struct irq_chip *chip,
			 void *chip_data, irq_flow_handler_t handler,
			 void *handler_data, const char *handler_name)
{
	irq_domain_set_hwirq_and_chip(domain, virq, hwirq, chip, chip_data);
	__irq_set_handler(virq, handler, 0, handler_name);	//重点
	irq_set_handler_data(virq, handler_data);
}
```

<div align="center">↓</div>

```c
//kernel/irq/chip.c
void
__irq_set_handler(unsigned int irq, irq_flow_handler_t handle, int is_chained,
		  const char *name)
{
	unsigned long flags;
    
    //通过irq number获取文件描述符
	struct irq_desc *desc = irq_get_desc_buslock(irq, &flags, 0);

	if (!desc)
		return;

	__irq_do_set_handler(desc, handle, is_chained, name);	//重点
	irq_put_desc_busunlock(desc, flags);
}
```

<div align="center">↓</div>

```c
//kernel/irq/chip.c
static void
__irq_do_set_handler(struct irq_desc *desc, irq_flow_handler_t handle,
		     int is_chained, const char *name)
{
	if (!handle) {
		handle = handle_bad_irq;
	} else {
		struct irq_data *irq_data = &desc->irq_data;
#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY
		/*
		 * With hierarchical domains we might run into a
		 * situation where the outermost chip is not yet set
		 * up, but the inner chips are there.  Instead of
		 * bailing we install the handler, but obviously we
		 * cannot enable/startup the interrupt at this point.
		 */
		while (irq_data) {
			if (irq_data->chip != &no_irq_chip)
				break;
			/*
			 * Bail out if the outer chip is not set up
			 * and the interrupt supposed to be started
			 * right away.
			 */
			if (WARN_ON(is_chained))
				return;
			/* Try the parent */
			irq_data = irq_data->parent_data;
		}
#endif
		if (WARN_ON(!irq_data || irq_data->chip == &no_irq_chip))
			return;
	}

	/* Uninstall? */
	if (handle == handle_bad_irq) {
		if (desc->irq_data.chip != &no_irq_chip)
			mask_ack_irq(desc);
		irq_state_set_disabled(desc);
		if (is_chained)
			desc->action = NULL;
		desc->depth = 1;
	}
	desc->handle_irq = handle;
	desc->name = name;

	if (handle != handle_bad_irq && is_chained) {
		unsigned int type = irqd_get_trigger_type(&desc->irq_data);

		/*
		 * We're about to start this interrupt immediately,
		 * hence the need to set the trigger configuration.
		 * But the .set_type callback may have overridden the
		 * flow handler, ignoring that we're dealing with a
		 * chained interrupt. Reset it immediately because we
		 * do know better.
		 */
		if (type != IRQ_TYPE_NONE) {
			__irq_set_trigger(desc, type);
			desc->handle_irq = handle;	//重点, 这里真正将handle赋值给了handle_irq
		}

		irq_settings_set_noprobe(desc);
		irq_settings_set_norequest(desc);
		irq_settings_set_nothread(desc);
		desc->action = &chained_action;
		irq_activate_and_startup(desc, IRQ_RESEND);
	}
}
```

**到这里, 中断描述符desc->irq_handle = handle_fasteoi_irq;**

#### handle_fasteoi_irq

![image-20221026155734031](assets/image-20221026155734031.png)

**当CPU的中断到来后最终实际执行到了下面这个函数**

```c
//kernel/irq/chip.c
void handle_fasteoi_irq(struct irq_desc *desc)
{
	struct irq_chip *chip = desc->irq_data.chip;

	raw_spin_lock(&desc->lock);

	if (!irq_may_run(desc))
		goto out;

	desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

	/*
	 * If its disabled or no action available
	 * then mask it and get out of here:
	 */
	if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
		desc->istate |= IRQS_PENDING;
		mask_irq(desc);
		goto out;
	}

	kstat_incr_irqs_this_cpu(desc);
	if (desc->istate & IRQS_ONESHOT)
		mask_irq(desc);

	handle_irq_event(desc);	//重点

	cond_unmask_eoi_irq(desc, chip);

	raw_spin_unlock(&desc->lock);
	return;
out:
	if (!(chip->flags & IRQCHIP_EOI_IF_HANDLED))
		chip->irq_eoi(&desc->irq_data);
	raw_spin_unlock(&desc->lock);
}
```

<div align="center">↓</div>

```c
//kernel/irq/handle.c
irqreturn_t handle_irq_event(struct irq_desc *desc)
{
	irqreturn_t ret;

	desc->istate &= ~IRQS_PENDING;
	irqd_set(&desc->irq_data, IRQD_IRQ_INPROGRESS);
	raw_spin_unlock(&desc->lock);

	ret = handle_irq_event_percpu(desc);	//重点

	raw_spin_lock(&desc->lock);
	irqd_clear(&desc->irq_data, IRQD_IRQ_INPROGRESS);
	return ret;
}
```

<div align="center">↓</div>

```c
//kernel/irq/handle.c
irqreturn_t handle_irq_event_percpu(struct irq_desc *desc)
{
	irqreturn_t retval;
	unsigned int flags = 0;

	retval = __handle_irq_event_percpu(desc, &flags);	//重点

	add_interrupt_randomness(desc->irq_data.irq, flags);

	if (!noirqdebug)
		note_interrupt(desc, retval);
	return retval;
}
```

<div align="center">↓</div>

```c
irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc, unsigned int *flags)
{
	irqreturn_t retval = IRQ_NONE;
	unsigned int irq = desc->irq_data.irq;
	struct irqaction *action;

	record_irq_time(desc);

	for_each_action_of_desc(desc, action) {
		irqreturn_t res;

		/*
		 * If this IRQ would be threaded under force_irqthreads, mark it so.
		 */
		if (irq_settings_can_thread(desc) &&
		    !(action->flags & (IRQF_NO_THREAD | IRQF_PERCPU | IRQF_ONESHOT)))
			lockdep_hardirq_threaded();

		trace_irq_handler_entry(irq, action);
		res = action->handler(irq, action->dev_id);		//这里真正调用了在rtc驱动中注册的中断服务函数
		trace_irq_handler_exit(irq, action, res);

		if (WARN_ONCE(!irqs_disabled(),"irq %u handler %pS enabled interrupts\n",
			      irq, action->handler))
			local_irq_disable();

		switch (res) {
		case IRQ_WAKE_THREAD:
			/*
			 * Catch drivers which return WAKE_THREAD but
			 * did not set up a thread function
			 */
			if (unlikely(!action->thread_fn)) {
				warn_no_thread(irq, action);
				break;
			}

			__irq_wake_thread(desc, action);

			fallthrough;	/* to add to randomness */
		case IRQ_HANDLED:
			*flags |= action->flags;
			break;

		default:
			break;
		}

		retval |= res;
	}

	return retval;
}
```



## 7. Linux中断的上半部和下半部

**Linux的中断处理可以分为上半部和下半部, 因为当中断来的时候, 既要进行寄存器的配置, 同时也要处理数据, 由于Linux是一个多任务系统, 如果在中断中进行大量的数据处理工作, 必然会导致其他任务无法被抢占, 使系统卡顿, 于是Linux便演化出了中断的上下半部**

### 上半部

**响应中断，硬件配置，发送EOI给GIC**

### 下半部

**数据复制、数据包封装转发、编解码…**

**Linux实现中断的下半部主要有四种方式:**

- **软中断 SoftIRQ**
- **TaskLet**
- **工作队列Work queue**
- **中断线程化**

## 8. 软中断SoftIRQ

**内核中提供了软中断相关的接口**

```c
void open_softirq(int nr, void (*action)(struct softirq_action *));
void raise_softirq_irqoff(unsigned int nr);
void raise_softirq(unsigned int nr);
```

**软中断号的定义**

```c
enum
{
	HI_SOFTIRQ=0,
	TIMER_SOFTIRQ,
	NET_TX_SOFTIRQ,
	NET_RX_SOFTIRQ,
	BLOCK_SOFTIRQ,
	IRQ_POLL_SOFTIRQ,
	TASKLET_SOFTIRQ,
	SCHED_SOFTIRQ,
	HRTIMER_SOFTIRQ,
	RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

	NR_SOFTIRQS
};
//内核中提供了一些软中断, 这些软中断号越小代表着优先级越高
```

### 添加一个软中断

**添加自己的软中断只需要在上面的枚举中添加一行就行, 例如**

```c
enum
{
    HI_SOFTIRQ=0,
    ...
    XXX_SOFTIRQ,
    NR_SOFTIRQ
}
```

### 使用软中断

**由于Linux内核提供的接口仅供Linux内核的开发者使用, 所以驱动开发者是无法使用上述接口的, 但是可以通过EXPORT_SYMBOL宏将该接口导出**

**在Linux源码中添加以下代码**

```c
//kernel/softirq.c
EXPORT_SYMBOL(raise_softirq_irqoff);
EXPORT_SYMBOL(raise_softirq);
EXPORT_SYMBOL(open_softirq);
```

#### 修改RTC驱动

```c
static void rtc_softirq_handler(struct softirq_action* sa)
{
    unsigned long tick = regs->RTCDR;
    current_time.hour = (tick % (24 * 3600)) / 3600;
    current_time.min = (tick % 3600) / 60;
    current_time.sec = tick % 60;
    printk("%ld:%ld:%ld\n", current_time.hour, current_time.min, current_time.sec);
}

static irqreturn_t  rtc_alarm_handler(int irq, void *dev_id)
{
    set_rtc_alarm(regs);
    raise_softirq(TASKLET_SOFTIRQ);
    return IRQ_HANDLED;
}


static int __init rtc_init(void)
{

    irqreturn_t ret = 0;

    regs = (rtc_reg_t *)ioremap(RTC_BASE, sizeof(rtc_reg_t)); 
    printk("rtc_init\n");

    set_rtc_alarm(regs);
    ret = request_irq(39, rtc_alarm_handler, 0, "rtc0-test", NULL);
    if (ret == -1){
        printk("request_irq failed!\n");
        return -1;
    }

    open_softirq(TASKLET_SOFTIRQ, rtc_softirq_handler);
   
    return 0;
}

static void __exit rtc_exit(void)
{
    free_irq(39, NULL);
    printk("Goodbye rtc module!\n");
}
```

**在上述代码中, 当中断到来时执行了rtc_alarm_handler, 该函数在执行过程中中断是关闭的, 其他中断无法打断, 在raise_softirq后注册的rtc_softirq_handler会执行, 此时的中断是开启的, 该函数可以被打断, 以此就达到了将上半部和下半部分开的目的**

### 软中断实现

**在Linux内核中, 软中断的服务函数保存在一个结构体中**

```c
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};
```

**和硬中断类似, Linux内核定义了一个softirq_action数组, 该数组保存了每个软中断号对应的软中断服务函数**

```c
static struct softirq_action softirq_vec[NR_SOFTIRQS]; //NR_SOFTIRQS为软中断的数量
```

**软中断的执行是从退出硬中断开始的, 在执行硬中断服务函数的函数前, 会有一个进入中断的操作, 在执行完毕后有退出中断的操作**

```c
int __handle_domain_irq(struct irq_domain *domain, unsigned int hwirq,
			bool lookup, struct pt_regs *regs)
{
	...
       
	irq_enter();	//进入中断

	//执行中断服务函数

	irq_exit();		//退出中断
	
    ...
	return ret;
}
```

**软中断就是从这个irq_exit()开始的**

```c
//kernel/softirq.c
void irq_exit(void)
{
	__irq_exit_rcu();	//重点
	rcu_irq_exit();
	 /* must be last! */
	lockdep_hardirq_exit();
}
				↓
static inline void __irq_exit_rcu(void)
{
#ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
	local_irq_disable();
#else
	lockdep_assert_irqs_disabled();
#endif
	account_irq_exit_time(current);
	preempt_count_sub(HARDIRQ_OFFSET);
	if (!in_interrupt() && local_softirq_pending())
		invoke_softirq();	//重点, 判断当前是否以及退出硬中断以及是否触发了软中断

	tick_irq_exit();
}
			↓
static inline void invoke_softirq(void)
{
	if (ksoftirqd_running(local_softirq_pending()))
		return;

	if (!force_irqthreads) {
		do_softirq_own_stack();
	} else {
		wakeup_softirqd();
	}
}
```
<div align="center">↓</div>
```c
///home/zhaixue/linux-5.10.4/include/linux/interrupt.h
static inline void do_softirq_own_stack(void)
{
	__do_softirq();
}
```

<div align="center">↓</div>

```c
//kernel/softirq.c
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
	unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
	unsigned long old_flags = current->flags;
	int max_restart = MAX_SOFTIRQ_RESTART;
	struct softirq_action *h;
	bool in_hardirq;
	__u32 pending;
	int softirq_bit;

	/*
	 * Mask out PF_MEMALLOC as the current task context is borrowed for the
	 * softirq. A softirq handled, such as network RX, might set PF_MEMALLOC
	 * again if the socket is related to swapping.
	 */
	current->flags &= ~PF_MEMALLOC;

	pending = local_softirq_pending();
	account_irq_enter_time(current);

	__local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
	in_hardirq = lockdep_softirq_start();

restart:
	/* Reset the pending bitmask before enabling irqs */
	set_softirq_pending(0);

	local_irq_enable();

	h = softirq_vec;

	while ((softirq_bit = ffs(pending))) {
		unsigned int vec_nr;
		int prev_count;

		h += softirq_bit - 1;

		vec_nr = h - softirq_vec;
		prev_count = preempt_count();

		kstat_incr_softirqs_this_cpu(vec_nr);

		trace_softirq_entry(vec_nr);
		h->action(h);	//重点, 执行软中断
		trace_softirq_exit(vec_nr);
		if (unlikely(prev_count != preempt_count())) {
			pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
			       vec_nr, softirq_to_name[vec_nr], h->action,
			       prev_count, preempt_count());
			preempt_count_set(prev_count);
		}
		h++;
		pending >>= softirq_bit;
	}

	if (__this_cpu_read(ksoftirqd) == current)
		rcu_softirq_qs();
	local_irq_disable();

	pending = local_softirq_pending();
	if (pending) {
		if (time_before(jiffies, end) && !need_resched() &&
		    --max_restart)
			goto restart;

		wakeup_softirqd();
	}

	lockdep_softirq_end(in_hardirq);
	account_irq_exit_time(current);
	__local_bh_enable(SOFTIRQ_OFFSET);
	WARN_ON_ONCE(in_interrupt());
	current_restore_flags(old_flags, PF_MEMALLOC);
}
```

### 软中断运行机制

- **运行于中断退出后的某个时机**
- **开启中断, 可以被其他中断打断, 不允许嵌套**
- **当软中断执行次数过多或者执行时间大于2ms就会延后到线程中执行, 由系统进程ksoftirqd托管**

## 9. tasklet

**tasklet是基于软中断实现的, 在软中断的枚举定义中就有一个TASKLET_SOFTIRQ, Linux内核提供了使用tasklet的api, 由于软中断仅供内核开发者使用, 所以驱动开发中一般使用tasklet进行中断的下半部操作**

### tasklet 接口

```c
//include/linux/interrupt.h

/**
 * @brief 		初始化一个task
 * 
 * @param t 	task结构体, 由用户定义
 * @param func 	回调函数的函数指针
 * @param data 	传递给回调函数的参数
 */
void tasklet_init(struct tasklet_struct *t,
			 void (*func)(unsigned long), unsigned long data);

/**
 * @brief 		在TASKLET_SOFTIRQ的action回调中的task链表中将该task加入调度
 * 
 * @param t		task结构体
 */
static inline void tasklet_schedule(struct tasklet_struct *t);

/**
 * @brief 		在HI_SOFTIRQ的action回调中的task链表中将该task加入调度, 比普通的tasklet的优先级要高
 * 
 * @param t 	task结构体
 */
static inline void tasklet_hi_schedule(struct tasklet_struct *t);

/**
 * @brief 		在task链表中删除该task
 * 
 * @param t 	task结构体
 */
void tasklet_kill(struct tasklet_struct *t);
```

**将RTC驱动修改为使用tasklet的代码如下**

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/io.h>

typedef volatile struct{
        unsigned long  RTCDR;    /* +0x00: data register */
        unsigned long  RTCMR;    /* +0x04: match register */
        unsigned long  RTCLR;    /* +0x08: load register */
        unsigned long  RTCCR;    /* +0x0C: control register */
        unsigned long  RTCIMSC;  /* +0x10: interrupt mask set and clear register*/
        unsigned long  RTCRIS;   /* +0x14: raw interrupt status register*/
        unsigned long  RTCMIS;   /* +0x18: masked interrupt status register */
        unsigned long  RTCICR;   /* +0x1C: interrupt clear register */
}rtc_reg_t;

struct rtc_time{
    unsigned long year;
    unsigned long month;
    unsigned long day;
    unsigned long hour;
    unsigned long min;
    unsigned long sec;
};

#define RTC_BASE 0x10017000

volatile rtc_reg_t *regs = NULL;
static struct rtc_time current_time;
int counter = 0;

void set_rtc_alarm(rtc_reg_t *regs)
{
    unsigned long tmp = 0;
    tmp = regs->RTCCR;    /* write enable */
    tmp = tmp & 0xFFFFFFFE;
    regs->RTCCR = tmp;

    tmp = regs->RTCDR;    /* get current time */
    regs->RTCMR = tmp + 1;/* set alarm time */

    regs->RTCICR = 1;     /* clear RTCINTR interrupt */ 
    regs->RTCIMSC = 1;    /* set the mask */

    tmp = regs->RTCCR;    /* write disable */
    tmp = tmp | 0x1;
    regs->RTCCR = tmp;
}

static void rtc_tasklet_handler(unsigned long data)
{
    unsigned long tick = regs->RTCDR;
    current_time.hour = (tick % (24 * 3600)) / 3600;
    current_time.min = (tick % 3600) / 60;
    current_time.sec = tick % 60;
    printk("%ld:%ld:%ld\n", current_time.hour, current_time.min, current_time.sec);
}

struct tasklet_struct rtc_task;

static irqreturn_t  rtc_alarm_handler(int irq, void *dev_id)
{
    set_rtc_alarm(regs);
    tasklet_schedule(&rtc_task);
    return IRQ_HANDLED;
}

static int __init rtc_init(void)
{

    irqreturn_t ret = 0;

    regs = (rtc_reg_t *)ioremap(RTC_BASE, sizeof(rtc_reg_t)); 
    printk("rtc_init\n");

    set_rtc_alarm(regs);
    ret = request_irq(39, rtc_alarm_handler, 0, "rtc0-test", NULL);
    if (ret == -1){
        printk("request_irq failed!\n");
        return -1;
    }

    tasklet_init(&rtc_task, rtc_tasklet_handler, 0);
   
    return 0;
}

static void __exit rtc_exit(void)
{
    free_irq(39, NULL);
    tasklet_kill(&rtc_task);
    printk("Goodbye rtc module!\n");
}

module_init(rtc_init);
module_exit(rtc_exit);

MODULE_LICENSE("GPL");
```

### tasklet运行机制

**在Linux内核中定义了两个链表, 用来存放普通的TASKLET_SOFTIRQ任务和HI_SOFTIRQ的任务**

```c
//kernel/softirq.c
static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);
```

**Linux内核中定义了TASKLET_SOFTIRQ和HI_SOFTIRQ这两个软中断的action, 在触发该软中断时会去遍历该task链表, 并取出其中的任务并执行**

**tasklet的运行机制:**

- **基于软中断**
- **开启中断, 运行于中断上下文**
- **当tasklet负载过重时, 会被转移到进程上下文中运行**

## 10.工作队列workqueue

### workqueue API介绍

```c
//初始化一个work
#define INIT_WORK(_work, _func);

/**
 * @brief 			将一个work添加进work队列中参与调度
 * 
 * @param work 		用户的work
 * @return true 
 * @return false 
 */
static inline bool schedule_work(struct work_struct *work);

/**
 * @brief 			将一个work从work队列中取消
 * 
 * @param work 		用户的work
 * @return true 
 * @return false 
 */
bool cancel_work_sync(struct work_struct *work);
```

**中断下半部使用workqueue实现的rtc驱动**

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/workqueue.h>
#include <linux/io.h>

typedef volatile struct{
        unsigned long  RTCDR;    /* +0x00: data register */
        unsigned long  RTCMR;    /* +0x04: match register */
        unsigned long  RTCLR;    /* +0x08: load register */
        unsigned long  RTCCR;    /* +0x0C: control register */
        unsigned long  RTCIMSC;  /* +0x10: interrupt mask set and clear register*/
        unsigned long  RTCRIS;   /* +0x14: raw interrupt status register*/
        unsigned long  RTCMIS;   /* +0x18: masked interrupt status register */
        unsigned long  RTCICR;   /* +0x1C: interrupt clear register */
}rtc_reg_t;

struct rtc_time{
    unsigned long year;
    unsigned long month;
    unsigned long day;
    unsigned long hour;
    unsigned long min;
    unsigned long sec;
};

#define RTC_BASE 0x10017000

volatile rtc_reg_t *regs = NULL;
static struct rtc_time current_time;
int counter = 0;

void set_rtc_alarm(rtc_reg_t *regs)
{
    unsigned long tmp = 0;
    tmp = regs->RTCCR;    /* write enable */
    tmp = tmp & 0xFFFFFFFE;
    regs->RTCCR = tmp;

    tmp = regs->RTCDR;    /* get current time */
    regs->RTCMR = tmp + 1;/* set alarm time */

    regs->RTCICR = 1;     /* clear RTCINTR interrupt */ 
    regs->RTCIMSC = 1;    /* set the mask */

    tmp = regs->RTCCR;    /* write disable */
    tmp = tmp | 0x1;
    regs->RTCCR = tmp;
}

static void rtc_legacy_workqueue(struct work_struct* work)
{
    unsigned long tick = regs->RTCDR;
    current_time.hour = (tick % (24 * 3600)) / 3600;
    current_time.min = (tick % 3600) / 60;
    current_time.sec = tick % 60;
    printk("rtc workqueue work!");
    printk("%ld:%ld:%ld\n", current_time.hour, current_time.min, current_time.sec);
}

static struct work_struct rtc_work;

static irqreturn_t  rtc_alarm_handler(int irq, void *dev_id)
{
    set_rtc_alarm(regs);
    schedule_work(&rtc_work);
    return IRQ_HANDLED;
}

static int __init rtc_init(void)
{
    irqreturn_t ret = 0;
    regs = (rtc_reg_t *)ioremap(RTC_BASE, sizeof(rtc_reg_t)); 
    printk("rtc_init\n");
    set_rtc_alarm(regs);
    ret = request_irq(39, rtc_alarm_handler, 0, "rtc0-test", NULL);
    if (ret == -1){
        printk("request_irq failed!\n");
        return -1;
    }
    INIT_WORK(&rtc_work, rtc_legacy_workqueue);
    return 0;
}

static void __exit rtc_exit(void)
{
    free_irq(39, NULL);
    cancel_work_sync(&rtc_work);
    printk("Goodbye rtc module!\n");
}

module_init(rtc_init);
module_exit(rtc_exit);

MODULE_LICENSE("GPL");
```

### 延迟workqueue API介绍

**延迟工作队列是基于workqueue实现的, 可以用来实现一些需要延时执行的工作, 例如按键消抖**

```c
//初始化一个延迟work
 INIT_DELAYED_WORK(_work, _func);

/**
 * @brief 将一个work加入延迟工作队列中, 延迟一定时间后执行
 * 
 * @param dwork 	用户的work
 * @param delay 	延迟时间
 * @return true 
 * @return false 
 */
static inline bool schedule_delayed_work(struct delayed_work *dwork,
					 unsigned long delay);

/**
 * @brief 等待该work执行完毕
 * 
 * @param dwork 	用户的work
 * @return true 
 * @return false 
 */
bool flush_delayed_work(struct delayed_work *dwork);

/**
 * @brief 取消延时工作任务并等待它完成
 * 
 * @param dwork 	用户的work
 * @return true 
 * @return false 
 */
bool cancel_delayed_work_sync(struct delayed_work *dwork);
```

**使用延时工作队列实现中断下半部的rtc驱动**

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/workqueue.h>
#include <linux/io.h>

typedef volatile struct{
        unsigned long  RTCDR;    /* +0x00: data register */
        unsigned long  RTCMR;    /* +0x04: match register */
        unsigned long  RTCLR;    /* +0x08: load register */
        unsigned long  RTCCR;    /* +0x0C: control register */
        unsigned long  RTCIMSC;  /* +0x10: interrupt mask set and clear register*/
        unsigned long  RTCRIS;   /* +0x14: raw interrupt status register*/
        unsigned long  RTCMIS;   /* +0x18: masked interrupt status register */
        unsigned long  RTCICR;   /* +0x1C: interrupt clear register */
}rtc_reg_t;

struct rtc_time{
    unsigned long year;
    unsigned long month;
    unsigned long day;
    unsigned long hour;
    unsigned long min;
    unsigned long sec;
};

#define RTC_BASE 0x10017000

volatile rtc_reg_t *regs = NULL;
static struct rtc_time current_time;
int counter = 0;

void set_rtc_alarm(rtc_reg_t *regs)
{
    unsigned long tmp = 0;
    tmp = regs->RTCCR;    /* write enable */
    tmp = tmp & 0xFFFFFFFE;
    regs->RTCCR = tmp;

    tmp = regs->RTCDR;    /* get current time */
    regs->RTCMR = tmp + 1;/* set alarm time */

    regs->RTCICR = 1;     /* clear RTCINTR interrupt */ 
    regs->RTCIMSC = 1;    /* set the mask */

    tmp = regs->RTCCR;    /* write disable */
    tmp = tmp | 0x1;
    regs->RTCCR = tmp;
}

static void rtc_delayed_workqueue(struct work_struct* work)
{
    unsigned long tick = regs->RTCDR;
    current_time.hour = (tick % (24 * 3600)) / 3600;
    current_time.min = (tick % 3600) / 60;
    current_time.sec = tick % 60;
    printk("rtc workqueue work!");
    printk("%ld:%ld:%ld\n", current_time.hour, current_time.min, current_time.sec);
}

static struct delayed_work rtc_delay_work;

static irqreturn_t  rtc_alarm_handler(int irq, void *dev_id)
{
    set_rtc_alarm(regs);
    schedule_delayed_work(&rtc_delay_work, 3*HZ);
    return IRQ_HANDLED;
}

static int __init rtc_init(void)
{
    irqreturn_t ret = 0;
    regs = (rtc_reg_t *)ioremap(RTC_BASE, sizeof(rtc_reg_t)); 
    printk("rtc_init\n");

    set_rtc_alarm(regs);
    ret = request_irq(39, rtc_alarm_handler, 0, "rtc0-test", NULL);
    if (ret == -1){
        printk("request_irq failed!\n");
        return -1;
    }

    INIT_DELAYED_WORK(&rtc_delay_work, rtc_delayed_workqueue);
    return 0;
}

static void __exit rtc_exit(void)
{
    free_irq(39, NULL);
    flush_delayed_work(&rtc_delay_work);
    cancel_delayed_work_sync(&rtc_delay_work);
    printk("Goodbye rtc module!\n");
}

module_init(rtc_init);
module_exit(rtc_exit);

MODULE_LICENSE("GPL");
```

### workqueue的运行机制

- **运行于进程上下文**
- **开启中断, 可以被抢占**

### workqueue的弊端

**在Linux中, 每个CPU都有一个工作队列线程, 这个线程维护了一个共享的workqueue, 叫做system_wq, 不同的驱动通过api向该workqueue中添加work, 多核CPU上, Linux则会使用均衡负载策略将work分配到不同的cpu上**

**由于工作队列运行在进程上下文, 所以驱动中的work是可以进行sleep的, 但是由于workqueue的执行是串行的, 所以, 当一个work在休眠时, 该workqueue上的其他work是无法执行的, 加入cpu的work过多, 势必会导致由于该work的休眠/唤醒带来的时间浪费, 而拖慢整个系统的运行, 而且一个workqueue上的work是无法被转移到另一个cpu上的workqueue上运行的, 这就是普通workqueue的弊端**

## 11. CMWQ(并发队列)

<div align="center"><i>注: 部分内容引用自<a href="https://www.likecs.com/show-190862.html" >https://www.likecs.com/show-190862.html</a></i></div>

### WQ概述

**Workqueue(WQ)机制是Linux2.6以前内核中最常用的异步处理机制。Workqueue机制的主要概念包括：work用于描述放到队列里即将被执行的函数；worker表示一个独立的线程，用于执行异步上下文处理；workqueue用于存放work的队列。
当workqueue上有work条目时，worker线程被触发来执行work对应的函数。如果有多个work条目在队列，worker会按顺序处理所有work。**

**在最初的WQ实现中，多线程WQ（MTWQ）在每个CPU上都有一个worker线程，单线程WQ（STWQ）则总共只有一个worker线程。一个MTWQ的worker个数和CPU核数相同，多年来，MTWQ大量使用使得线程数量大量增加，甚至超过了某些系统对PID空间默认32K的限制。
尽管MTWQ浪费大量资源，但其提供的并发水平还是不能让人满意。并发的限制在STWQ和MTWQ上都存在，虽然MT相对来说不那么严重。MTWQ在每个CPU上提供了一个上下文执行环境，STWQ则在整个系统提供一个上下文执行环境。work任务需要竞争这些有限的执行环境资源，从而导致死锁等问题。**

### CMWQ概述

**并发和资源之间的紧张关系使得一些使用者不得不做出一些不必要的折中，比如libata的polling PIOs选择STWQ，这样就无法有两个polling PIOs同时进行处理。因为MTWQ并不能提供高并发能力，因此async和fscache不得不实现自己的线程池来提供高并发能力。
Concurrency Managed Workqueue (CMWQ)重新设计了WQ机制，并实现如下目标：**

- **1. 保持原workqueue API的兼容；**
- **2. 使用per-CPU统一的worker池，为所有WQ共享使用并提供灵活的并发级别，同时不浪费不必要的资源；**
- **3. 自动调整worker池和并发级别，让使用者不用关心这些细节。**

### CMWQ设计思想

**一个work是一个简单的结构体，保存一个函数指针用于异步执行。任何驱动或者子系统想要一个函数被异步执行，都需要设置一个work指向该函数并将其放入workqueue队列。然后worker线程从队列上获取work并执行对应的函数，如果队列里没有work，则worker线程处于空闲状态。这些worker线程用线程池机制来管理。**

**CMWQ设计时将面向用户的workqueue机制和后台worker线程池管理机制进行了区分。后台的workqueue被称为GCWQ（推测可能是Global Concurrency WorkQueue），在每个CPU上存在一个GCWQ，用于处理该CPU上所有workqueue的work。每个GCWQ有两个线程池：一个用于普通work处理，另一个用于高优先级work处理。**

**内核子系统和驱动程序通过workqueue API创建和调度work，并可以通过设置flags来指定CPU核心、可重复性、并发限制，优先级等。当work放入workqueue时，通过队列参数和属性决定目标GCWQ和线程池，work最终放入对应线程池的共享worklist上。通过如果没有特别设定，work会被默认放入当前运行的CPU核上的GCWQ线程池的worklist上。**

**GCWQ的线程池在实现时同时考虑了并发能力和资源占用，仅可能占用最小的资源并提供足够的并发能力。每个CPU上绑定的线程池通过hook到CPU调度机制来实现并发管理。当worker被唤醒或者进入睡眠都会通知到线程池，线程池保持对当前可以运行的worker个数的跟踪。通常我们不期望一个work独占CPU和运行很多个CPU周期，因此维护刚好足够的并发以防止work处理的速度降低是最优的。当CPU上有一个或多个runnalbe的worker，线程池不会启动新的work任务。当上一个running的work转入睡眠，则立即调度一个新的worker。这样当有work在pending的时候，CPU一直保持干活的状态。这样来保证用最小的worker个数同时足够的执行带宽。**

**维持idle状态的worker只是消耗部分kthreads的内存，因此CMWQ在杀掉idle的worker之前一段时间让其活着。**

**unbound的WQ并不使用上述机制，而是用pseudo unbound CPU的线程池去尽快处理所有work。CMWQ的使用者来控制并发级别，并可以设置一个flag来忽略并发管理机制。**

**CMWQ通过创建更多的worker以及rescue-worker来保证任务按时处理。所有可能在内存回收的代码路径上执行的work必须放到特定的workqueue，该workqueue上有一个rescue-worker可以在内存压力下执行，这样避免在内存回收时出现死锁。**

### CMWQ API介绍

```c
/**
 * @brief 用于分配一个WQ。原来的create_workqueue()系列接口已经弃用并计划删除
 * 
 * @param fmt 			该wq的名字
 * @param flags 		该wq的属性
 * @param max_active 	该wq可使用最多的线程数
 * @param ...
 * @return 				返回的wq指针
 */
struct workqueue_struct *alloc_workqueue(const char *fmt,
					 unsigned int flags,
					 int max_active, ...)
    
/**
 * @brief 		将一个work加入到一个workqueue中开始调度
 * 
 * @param wq 	target wq
 * @param work 	target work
 * @return true 
 * @return false 
 */
static inline bool queue_work(struct workqueue_struct *wq,
			      struct work_struct *work);
```

**基于CMWQ的中断下半部实现的rtc驱动:**

```c
#include "linux/wait.h"
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/workqueue.h>
#include <linux/io.h>

typedef volatile struct{
        unsigned long  RTCDR;    /* +0x00: data register */
        unsigned long  RTCMR;    /* +0x04: match register */
        unsigned long  RTCLR;    /* +0x08: load register */
        unsigned long  RTCCR;    /* +0x0C: control register */
        unsigned long  RTCIMSC;  /* +0x10: interrupt mask set and clear register*/
        unsigned long  RTCRIS;   /* +0x14: raw interrupt status register*/
        unsigned long  RTCMIS;   /* +0x18: masked interrupt status register */
        unsigned long  RTCICR;   /* +0x1C: interrupt clear register */
}rtc_reg_t;

struct rtc_time{
    unsigned long year;
    unsigned long month;
    unsigned long day;
    unsigned long hour;
    unsigned long min;
    unsigned long sec;
};

#define RTC_BASE 0x10017000

volatile rtc_reg_t *regs = NULL;
static struct rtc_time current_time;
int counter = 0;

static struct work_struct rtc_work;
static struct workqueue_struct* m_workqueue;

void set_rtc_alarm(rtc_reg_t *regs)
{
    unsigned long tmp = 0;
    tmp = regs->RTCCR;    /* write enable */
    tmp = tmp & 0xFFFFFFFE;
    regs->RTCCR = tmp;

    tmp = regs->RTCDR;    /* get current time */
    regs->RTCMR = tmp + 5;/* set alarm time */

    regs->RTCICR = 1;     /* clear RTCINTR interrupt */ 
    regs->RTCIMSC = 1;    /* set the mask */

    tmp = regs->RTCCR;    /* write disable */
    tmp = tmp | 0x1;
    regs->RTCCR = tmp;
}

static void rtc_legacy_workqueue(struct work_struct* work)
{
    unsigned long tick = regs->RTCDR;
    current_time.hour = (tick % (24 * 3600)) / 3600;
    current_time.min = (tick % 3600) / 60;
    current_time.sec = tick % 60;
    printk("rtc cmwq work!");
    printk("%ld:%ld:%ld\n", current_time.hour, current_time.min, current_time.sec);
}

static irqreturn_t  rtc_alarm_handler(int irq, void *dev_id)
{
    set_rtc_alarm(regs);
    queue_work(m_workqueue, &rtc_work);
    return IRQ_HANDLED;
}

static int __init rtc_init(void)
{
    irqreturn_t ret = 0;
    regs = (rtc_reg_t *)ioremap(RTC_BASE, sizeof(rtc_reg_t)); 
    printk("rtc_init\n");
    set_rtc_alarm(regs);
    ret = request_irq(39, rtc_alarm_handler, 0, "rtc0-test", NULL);
    if (ret == -1){
        printk("request_irq failed!\n");
        return -1;
    }
    m_workqueue = alloc_workqueue("rtc_wq", WQ_MEM_RECLAIM, 3);
    INIT_WORK(&rtc_work, rtc_legacy_workqueue);
    return 0;
}

static void __exit rtc_exit(void)
{
    free_irq(39, NULL);
    flush_workqueue(m_workqueue);
    destroy_workqueue(m_workqueue);
    printk("Goodbye rtc module!\n");
}

module_init(rtc_init);
module_exit(rtc_exit);

MODULE_LICENSE("GPL");
```

*参考: Documents/core-api/workqueue.rst*

## 12.中断线程化

**中断线程化与WQ不同的是, 中断线程化的下半部会创建一个内核线程直接参与调度, 当有中断发生时，响应中断后，会把中断处理函数当做一个线程与其他线程一样的处理，中断将作为内核线程运行，而且也可以被赋予不同的优先级，任务的优先级可能比中断线程的优先级高，这样做的目的就是保证高优先级的任务能被优先处理。**

**中断线程化后进一步压缩了上半部的工作量，上半部的工作仅仅需要完成 “快速检查”，譬如确保中断的确来自期望的设备；如果检查通过，它将对硬件中断完成确认并通知内核唤醒中断处理线程完成中断处理的下半部。**

**降低内核中的延迟只是中断线程化带来的好处之一，除此之外它还具备其他方面的优点。其中最突出的一点是：由于中断线程化后简化乃至避免了整个中断处理流程中 “硬中断” 和 “软中断” 两个阶段之间可能涉及的锁同步机制，从而降低了整体实现上的复杂性。中断处理线程化还有助于内核的调试。**

### 中断线程化API

```c
/**
 * @brief 
 * 
 * @param irq 			软中断号
 * @param handler 		中断处理函数
 * @param thread_fn 	中断线程化处理函数
 * @param irqflags 		中断属性
 * @param devname 		设备名
 * @param dev_id 		设备id
 * @return int 
 */
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
			 irq_handler_t thread_fn, unsigned long irqflags,
			 const char *devname, void *dev_id);
```

**使用中断线程化处理的rtc驱动:**

```c
#include "linux/irqreturn.h"
#include "linux/stddef.h"
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/io.h>

typedef volatile struct{
        unsigned long  RTCDR;    /* +0x00: data register */
        unsigned long  RTCMR;    /* +0x04: match register */
        unsigned long  RTCLR;    /* +0x08: load register */
        unsigned long  RTCCR;    /* +0x0C: control register */
        unsigned long  RTCIMSC;  /* +0x10: interrupt mask set and clear register*/
        unsigned long  RTCRIS;   /* +0x14: raw interrupt status register*/
        unsigned long  RTCMIS;   /* +0x18: masked interrupt status register */
        unsigned long  RTCICR;   /* +0x1C: interrupt clear register */
}rtc_reg_t;

struct rtc_time{
    unsigned long year;
    unsigned long month;
    unsigned long day;
    unsigned long hour;
    unsigned long min;
    unsigned long sec;
};

#define RTC_BASE 0x10017000

volatile rtc_reg_t *regs = NULL;
static struct rtc_time current_time;
int counter = 0;

void set_rtc_alarm(rtc_reg_t *regs)
{
    unsigned long tmp = 0;
    tmp = regs->RTCCR;    /* write enable */
    tmp = tmp & 0xFFFFFFFE;
    regs->RTCCR = tmp;

    tmp = regs->RTCDR;    /* get current time */
    regs->RTCMR = tmp + 1;/* set alarm time */

    regs->RTCICR = 1;     /* clear RTCINTR interrupt */ 
    regs->RTCIMSC = 1;    /* set the mask */

    tmp = regs->RTCCR;    /* write disable */
    tmp = tmp | 0x1;
    regs->RTCCR = tmp;
}


static irqreturn_t rtc_thread_handler(int irq, void *dev_id)
{
    unsigned long tick = regs->RTCDR;
    current_time.hour = (tick % (24 * 3600)) / 3600;
    current_time.min = (tick % 3600) / 60;
    current_time.sec = tick % 60;
    printk("%ld:%ld:%ld\n", current_time.hour, current_time.min, current_time.sec);
    return IRQ_HANDLED;
}

static irqreturn_t  rtc_alarm_handler(int irq, void *dev_id)
{
    set_rtc_alarm(regs);
    return IRQ_WAKE_THREAD;
}

static int __init rtc_init(void)
{
    irqreturn_t ret = 0;
    regs = (rtc_reg_t *)ioremap(RTC_BASE, sizeof(rtc_reg_t)); 
    printk("rtc_init\n");

    set_rtc_alarm(regs);
    ret = request_threaded_irq(39, rtc_alarm_handler, rtc_thread_handler, 0, "rtc", NULL);
    if (ret == -1){
        printk("request_irq failed!\n");
        return -1;
    }
   
    return 0;
}

static void __exit rtc_exit(void)
{
    free_irq(39, NULL);
    printk("Goodbye rtc module!\n");
}

module_init(rtc_init);
module_exit(rtc_exit);
MODULE_LICENSE("GPL");
```

