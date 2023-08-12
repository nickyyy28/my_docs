---
title: 嵌入式Linux-系统调用
date: 2023-08-12 22:58:23
categories:
  - 嵌入式开发
tags:
  - Linux内核
  - ARM
---

<div align="center">注: 本文档中的所有代码均运行于arm32平台上</div>

## 1. 什么是系统调用(syscall)?

**无论是嵌入式系统还是桌面PC中, 使用的CPU往往都会有多个运行模式,cpu在这些不同的模式下的权限是不同的, 低权限无法使用一些`特权指令`, 而且Linux系统也存在两个状态, 用户态和内核态, 用户态在运行时有时候会需要进行一些特权操作, 例如读写文件, 建立socket连接等等操作需要进入内核态, 而操作系统提供给用户的接口就叫做`系统调用`**

### 不同CPU的各种模式

* **X86** 

  **ring0-ring3**

* **ARM 32bit**
  * **USR - 用户模式**
  * **IRQ - 普通处理中断模式**
  * **FIQ - 快速处理中断模式**
  * **SVC - 操作系统保护模式 **
  * **ABT - 中止模式,处理存储器故障、实现虚拟存储器和存储器保护**
  * **UND - 未定义, 处理未定义的指令陷阱，支持硬件协处理器的软件仿真**
  * **SYS  - 系统模式, 运行特权操作系统任务**

* **ARM 64bit**

  **EL1-EL4**

## 2. Linux的系统调用

**Linux的系统调用主要分为系统调用表的定义, 系统调用函数的声明, 系统调用函数的实现**

<div align="center">注:KERNEL_DIR 为你的Linux内核源码目录</div>

### 系统调用表

**在Linux内核中, 系统调用想要被操作系统找到, 首先需要在一个系统调用表中添加定义**

**该表的位置在: `KERNEL_DIR/arch/arm/include/generated/calls-eabi.S`**

*注: 若不是嵌入式系统该文件该文件则位于: KERNEL_DIR/arch/arm/include/generated/calls-oabi.S*

```assembly
NATIVE(0, sys_restart_syscall)
NATIVE(1, sys_exit)
NATIVE(2, sys_fork)
NATIVE(3, sys_read)
NATIVE(4, sys_write)
NATIVE(5, sys_open)
NATIVE(6, sys_close)
NATIVE(8, sys_creat)
NATIVE(9, sys_link)
...
NATIVE(437, sys_openat2)
NATIVE(438, sys_pidfd_getfd)
NATIVE(439, sys_faccessat2)
NATIVE(440, sys_process_madvise)
```

### 系统调用函数的声明

**Linux系统中系统调用函数的声明在一个统一的头文件中, 位于: `KERNEL_DIR/include/linux/syscalls.h`**

```c
/* fs/dcache.c */
asmlinkage long sys_getcwd(char __user *buf, unsigned long size);

/* fs/cookies.c */
asmlinkage long sys_lookup_dcookie(u64 cookie64, char __user *buf, size_t len);

/* fs/eventfd.c */
asmlinkage long sys_eventfd2(unsigned int count, int flags);

/* fs/eventpoll.c */
asmlinkage long sys_epoll_create1(int flags);
asmlinkage long sys_epoll_ctl(int epfd, int op, int fd,
				struct epoll_event __user *event);
...
```

### 系统调用函数的实现

**Linux系统调用函数的实现并不统一, 例如sys_kill函数就位于: `KERNEL/arch/arm/kernel/signal.c`中**

**其他的系统调用分布在Linux源码中各个模块的文件中**

## 3. 用代码使用系统调用

**Linux系统提供了一个syscall函数, 通过该函数可以进行系统调用**

```
long syscall(long number, ...);
```

**使用该函数需要先包含以下头文件**

```c
#include <unistd.h>
#include <sys/syscall.h>
```

**下面使用syscall实现一个简单的printf函数, 将字符串输出至终端**

```c
#include <unistd.h>
#include <sys/syscall.h>

int main(int argc, char *argv[])
{
    syscall(4, 1, "Hello World\n", 12);
    return 0;
}
```

**传入系统调用参数的含义如下:**

```c
4: 指定的系统调用号, 通过在系统调用表中的定义, sys_write函数的序号为4 
```

![image-20221020150810029](https://pic.imgdb.cn/item/64d79d711ddac507cc822841.png)

```c
1: 你想向哪个文件写入, 在Linux系统中, 一切皆文件, 包括当前终端, 终端的文件描述符为1
"Hello World\n": 想要写入的字符串地址
12: 字符串长度
```

**交叉编译该文件, 复制可执行文件至开发板上可以看到执行结果:**

![image-20221020151725520](https://pic.imgdb.cn/item/64d79d7a1ddac507cc8241df.png)

## 4. 如何添加一个系统调用

**如果想在Linux内核中添加自己的系统调用, 可以像第二部分中的一样, 在三个文件中分别添加自己的定义和实现**

### 1. 添加系统调用表

**在calls-eabi.S中追加一行代码**

```assembly
NATIVE(441, sys_hello)	;注意, 不要使用重复的系统调用号
```

### 2. 添加系统调用函数声明

```c
/*自定义的系统调用*/
asmlinkage void sys_helo(const char __user *buff, size_t count);
```

### 3. 添加系统调用函数实现

```c
asmlinkage void sys_hello(char __user *buff, size_t count)
{
	char kernel_buf[100] = {0};
	if (buff) {
		copy_from_user(kernel_buf, buff, count > 100 ? 100 : count);
		kernel_buf[99] = '\0';
		printk("sys_hello: %s\n", kernel_buf);
	}
}
```

### 编译测试

**重新编译出zImage, uImage后更新内核文件, 编写系统调用测试代码如下: **

```c
#include <unistd.h>
#include <sys/syscall.h>

int main(int argc, char *argv[])
{
    syscall(441, "this is sys_hello\n", 18);
    return 0;
}
```

**交叉编译后执行如下:**

![image-20221020153904465](https://pic.imgdb.cn/item/64d79d841ddac507cc825ea1.png)

**成功执行了自己的系统调用**

## 5. 系统调用流程分析

<div align="center">注: arm32中svc指令等同于swi</div>

**通过查看man文档, 可以看到, 在进行系统调用时, 有一个指令让cpu进入特权模式, 同时将系统调用号, 系统调用参数传递进入cpu的寄存器中, 具体如下图:**

![image-20221020155208575](https://pic.imgdb.cn/item/64d79d8a1ddac507cc8270e1.png)

![image-20221020155448778](https://pic.imgdb.cn/item/64d79d921ddac507cc8287b6.png)

**通过该图可以看到, arm32-eabi平台上, 使cpu进入特权模式的指令是swi 0, 系统调用号保存在r7寄存器中, 而函数的参数则保存在r0-r6寄存器中**

**使用编译器编译代码时, 会默认链接上glibc的库, glibc是gnu组织发起的c标准库实现, 该库实现了对系统调用的封装, 通过反汇编该arm-libc.so动态库, 可以查到关于write操作的具体指令如下:**

![image-20221020155730745](https://pic.imgdb.cn/item/64d79d991ddac507cc829d4d.png)

**查看write函数的实现, 发现其将4传入了r7寄存器,这个4就是sys_write的系统调用号 然后执行了svc 0, cpu进入特权模式, 和上面提到的流程一样**

**在cpu执行了swi/svc 0指令后, 便跳转进入了相关的swi/svc handler中断服务函数, 该函数定义在: `KERNEL_DIR/arch/arm/kernel/entry-common.S`**

```assembly
/*=============================================================================
 * SWI handler
 *-----------------------------------------------------------------------------
 */

	.align	5
ENTRY(vector_swi)
#ifdef CONFIG_CPU_V7M
	v7m_exception_entry
#else
	sub	sp, sp, #PT_REGS_SIZE
	stmia	sp, {r0 - r12}			@ Calling r0 - r12
 ARM(	add	r8, sp, #S_PC		)
 ARM(	stmdb	r8, {sp, lr}^		)	@ Calling sp, lr
 THUMB(	mov	r8, sp			)
 THUMB(	store_user_sp_lr r8, r10, S_SP	)	@ calling sp, lr
	mrs	saved_psr, spsr			@ called from non-FIQ mode, so ok.
 TRACE(	mov	saved_pc, lr		)
	str	saved_pc, [sp, #S_PC]		@ Save calling PC
	str	saved_psr, [sp, #S_PSR]		@ Save CPSR
	str	r0, [sp, #S_OLD_R0]		@ Save OLD_R0
#endif
	zero_fp
	alignment_trap r10, ip, __cr_alignment
	asm_trace_hardirqs_on save=0
	enable_irq_notrace
	ct_user_exit save=0

	/*
	 * Get the system call number.
	 */

#if defined(CONFIG_OABI_COMPAT)

	/*
	 * If we have CONFIG_OABI_COMPAT then we need to look at the swi
	 * value to determine if it is an EABI or an old ABI call.
	 */
#ifdef CONFIG_ARM_THUMB
	tst	saved_psr, #PSR_T_BIT
	movne	r10, #0				@ no thumb OABI emulation
 USER(	ldreq	r10, [saved_pc, #-4]	)	@ get SWI instruction
#else
 USER(	ldr	r10, [saved_pc, #-4]	)	@ get SWI instruction
#endif
 ARM_BE8(rev	r10, r10)			@ little endian instruction

#elif defined(CONFIG_AEABI)

	/*
	 * Pure EABI user space always put syscall number into scno (r7).
	 */
#elif defined(CONFIG_ARM_THUMB)
	/* Legacy ABI only, possibly thumb mode. */
	tst	saved_psr, #PSR_T_BIT		@ this is SPSR from save_user_regs
	addne	scno, r7, #__NR_SYSCALL_BASE	@ put OS number in
 USER(	ldreq	scno, [saved_pc, #-4]	)

#else
	/* Legacy ABI only. */
 USER(	ldr	scno, [saved_pc, #-4]	)	@ get SWI instruction
#endif

	/* saved_psr and saved_pc are now dead */

	uaccess_disable tbl

	adr	tbl, sys_call_table		@ load syscall table pointer

#if defined(CONFIG_OABI_COMPAT)
	/*
	 * If the swi argument is zero, this is an EABI call and we do nothing.
	 *
	 * If this is an old ABI call, get the syscall number into scno and
	 * get the old ABI syscall table address.
	 */
	bics	r10, r10, #0xff000000
	eorne	scno, r10, #__NR_OABI_SYSCALL_BASE
	ldrne	tbl, =sys_oabi_call_table
#elif !defined(CONFIG_AEABI)
	bic	scno, scno, #0xff000000		@ mask off SWI op-code
	eor	scno, scno, #__NR_SYSCALL_BASE	@ check OS number
#endif
	get_thread_info tsk
	/*
	 * Reload the registers that may have been corrupted on entry to
	 * the syscall assembly (by tracing or context tracking.)
	 */
 TRACE(	ldmia	sp, {r0 - r3}		)

local_restart:
	ldr	r10, [tsk, #TI_FLAGS]		@ check for syscall tracing
	stmdb	sp!, {r4, r5}			@ push fifth and sixth args

	tst	r10, #_TIF_SYSCALL_WORK		@ are we tracing syscalls?
	bne	__sys_trace

	invoke_syscall tbl, scno, r10, __ret_fast_syscall

	add	r1, sp, #S_OFF
2:	cmp	scno, #(__ARM_NR_BASE - __NR_SYSCALL_BASE)
	eor	r0, scno, #__NR_SYSCALL_BASE	@ put OS number back
	bcs	arm_syscall
	mov	why, #0				@ no longer a real syscall
	b	sys_ni_syscall			@ not private func

#if defined(CONFIG_OABI_COMPAT) || !defined(CONFIG_AEABI)
	/*
	 * We failed to handle a fault trying to access the page
	 * containing the swi instruction, but we're not really in a
	 * position to return -EFAULT. Instead, return back to the
	 * instruction and re-enter the user fault handling path trying
	 * to page it in. This will likely result in sending SEGV to the
	 * current task.
	 */
9001:
	sub	lr, saved_pc, #4
	str	lr, [sp, #S_PC]
	get_thread_info tsk
	b	ret_fast_syscall
#endif
ENDPROC(vector_swi)
```

**上面的函数会导入系统调用表的指针, 并读取r7寄存器的值, 然后根据该值从系统调用表查找相应的系统调用函数, 并跳入该函数执行系统调用**

## 6. 虚拟系统调用(vsyscall)

**普通系统调用的流程需要从用户态->内核态->用户态, 中间需要经过大量的cpu状态切换, 需要大量的寄存器需要保存和恢复, 这个过程会耗费大量的时间, 对于一些只是简单读取一些数据的系统调用, 完全不需要这些操作, 例如读取当前时间, 不需要进行大量的cpu状态切换, 只是读取硬件的某个值罢了, 所以, 为了缓解该情况, 虚拟系统调用应运而生**

**系统调用中含有一个sys_time的函数, 该系统调用会读取硬件的时间返回给用户, 但是这个系统调用会带来一定的开销, 为了减小开销, glibc对该系统调用进行了优化, 通过mmap之类的函数将硬件地址映射在虚拟地址, 这样用户就可以直接读取虚拟地址的值, 而不需要反复切换cpu状态浪费资源**

```shell
cat /proc/self/maps #该命令可以输出当前进程的内存使用情况
```

![image-20221020170609816](https://pic.imgdb.cn/item/64d79da51ddac507cc82bf76.png)

**可以内存中看到有5个分区, 其中有常规的堆区/栈区, 还有一个vsyscall区, 这个区域就是glibc对部分系统调用实现的虚拟系统调用**

## 7. 虚拟动态共享对象VDSO(virtual dynamic shared object)

**VSDO可以认为是vsyscall的升级版本, vsyscall的缺点有很多, 比如它的内存地址是固定写死在程序里的, 很容易被恶意程序攻击, 且仅支持4个系统调用, 而在glibc-2.22版本之后, gnu组织加入了vdso对象, 它支持更多的虚拟系统调用, 而且使用了动态内存地址, 进一步提高了安全性, 速度更快**

**为了兼容以前的程序, gnu并没有删除vsyscall的内容, 依然提供vsyscall的支持**
