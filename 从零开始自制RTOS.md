---
title: 从零开始自制RTOS
date: 2023-06-11 08:57:16
categories: 
  - 嵌入式开发
tags:
  - RTOS
  - STM32
  - 操作系统
---

# 从零自制RTOS

## 1. 什么是RTOS

## 2. 需要的前置知识

### ARM汇编

**仅需要知道六个基础指令和基本的寄存器**

#### 基本寄存器

##### R0-R3

**父函数在调用子函数前先将子函数入口参数存入 R0~R3 寄存器中，若只有一个入口参数则使用 R0 寄存器传递，若有2个入口参数则使用 R0 和 R1 寄存器传递，以此类推。。当超过4个参数时，其余的入口参数则以此压入当前栈通过栈传递。子函数运行时，其将根据自身参数个数从 R0~R3 或者栈中读取入口参数**

```c
int var_1 = 0;
void func1(int a, int b)
{
	var_1 = a + b;
}
```

**该函数调用时,会将a和b放入R0-R3中，将该函数反汇编可以看到：**

```assembly
i.func1
    func1
    0x08000598:    1842        B.      ADDS     r2,r0,r1
    0x0800059a:    4b01        .K      LDR      r3,[pc,#4] ; [0x80005a0] = 0x20000000
    0x0800059c:    601a        .`      STR      r2,[r3,#0]
    0x0800059e:    4770        pG      BX       lr
```

**两个参数分别放入了R0和R1寄存器中，但是R0-R3一共只有四个寄存器，如果函数的参数大于四个，就需要用到栈了，通过栈将多出的参数保存：**

```c
int func(int v1, int v2,int v3,int v4,int v5,int v6,int v7,int v8,int v9,int v10,int v11,int v12,int v13,int v14,int v15,int v16)
{
	volatile int tmp;
	tmp += v1;
	tmp += v2;
	tmp += v3;
	tmp += v4;
	tmp += v5;
	tmp += v6;
	tmp += v7;
	tmp += v8;
	tmp += v9;
	tmp += v10;
	tmp += v11;
	tmp += v12;
	tmp += v13;
	tmp += v14;
	tmp += v15;
	tmp += v16;
	return tmp;
}
```

**反汇编后查看函数如下：**

```assembly
i.func
    func
    0x0800050c:    e92d4ff8    -..O    PUSH     {r3-r11,lr}
    0x08000510:    4604        .F      MOV      r4,r0
    0x08000512:    ad0a        ..      ADD      r5,sp,#0x28
    0x08000514:    e89510e0    ....    LDM      r5,{r5-r7,r12}
    0x08000518:    e9ddab10    ....    LDRD     r10,r11,[sp,#0x40]
    0x0800051c:    e9dd890e    ....    LDRD     r8,r9,[sp,#0x38]
    0x08000520:    9800        ..      LDR      r0,[sp,#0]
    0x08000522:    4420         D      ADD      r0,r0,r4
    ...
    0x0800058e:    9000        ..      STR      r0,[sp,#0]
    0x08000590:    9800        ..      LDR      r0,[sp,#0]
    0x08000592:    e8bd8ff8    ....    POP      {r3-r11,pc}
    0x08000596:    0000        ..      MOVS     r0,r0
```

**可以看到，程序使用了SP寄存器（栈指针）来进行函数参数的读取**

##### R4-R11 存放函数局部变量的寄存器

**普通的通用寄存器。AAPCS规定，发生函数调用时，父函数无需对这些寄存器进行备份处理，若子函数需要使用这些寄存器，则由子函数负责备份（需要使用哪个就备份哪个），以防止破坏父函数保存在 R4~R11 寄存器中的数据。子函数返回父函数前需先恢复 R4~R11 寄存器中的数值（使用了哪个就恢复哪个），恢复到父函数调用子函数这一时刻的数值，然后再返回到父函数**

```c
void func()
{
    int a,b,c,d; //放在R4-R11寄存器或者栈区
}
```

##### R12(IP) 

**内部调用暂时寄存器 IP。它在过程链接胶合代码（例如，交互操作胶合代码）中用于此角色。在过程调用之间，可以将它用于任何用途。被调用函数在返回之前不必恢复 R12**

##### R13(SP)栈指针

**保护栈的当前指针，函数存储在栈中的数据就是通过这个寄存器来寻址的。函数返回时需要保证 SP 指向 调用 该函数时的栈地址。**

##### R14(LR)链接寄存器

**用来保存函数的返回地址。父函数调用子函数时，父函数会将调用子函数指令的下一条指令地址存入到 LR 寄存器中，当子函数返回时只需要跳转到 LR 寄存器里的地址就会返回父函数继续执行。父函数调用子函数将子函数返回地址存入 LR 寄存器前， LR 寄存器中保存的可能是父函数返回其上一级函数的地址或其他有用的数据，因此需要先备份 LR 寄存器然后才能调用子函数**

```c
//以一段代码为例
//main()中:
volatile int x = 10, y =20;
volatile int reslut = 0;
reslut = add(&x, &y);

//add(int*, int*)中:
int add(int *a, int *b)
{
	volatile int tmp = *a;
	tmp += *b;
	return tmp;
}

//main()反汇编:
i.main
main
    ...
	0x080006c4:    200a        .       MOVS     r0,#0xa				;把10保存到R0中, 等同于int x = 10
    0x080006c6:    900e        ..      STR      r0,[sp,#0x38]		;把R0保存到栈中
    0x080006c8:    2014        .       MOVS     r0,#0x14			;把20保存到R0中, 等同于int y = 20
    0x080006ca:    900d        ..      STR      r0,[sp,#0x34]		;把R0保存到栈中
    0x080006cc:    2000        .       MOVS     r0,#0				;把0保存到R0中, 等同于int reslut = 0
    0x080006ce:    900c        ..      STR      r0,[sp,#0x30]		;把R0保存到栈中	
    0x080006d0:    a90d        ..      ADD      r1,sp,#0x34			;把sp指针+34的值放到R1寄存器,相当于&y
    0x080006d2:    a80e        ..      ADD      r0,sp,#0x38			;把sp指针+38的值放到R0寄存器,相当于&x
    0x080006d4:    f7ffff0c    ....    BL       add ; 0x80004f0		;将下一条指令的地址0x080006d8存到LR寄存																		 器,然后跳转到add函数
    0x080006d8:    900c        ..      STR      r0,[sp,#0x30]		;PC指向该条指令
    0x080006da:    980e        ..      LDR      r0,[sp,#0x38]
    ...

//add(int*, int*)反汇编:
i.add
add
    0x080004f0:    b518        ..      PUSH     {r3,r4,lr}			;该函数中使用了R3,R4,并需要返回,所以将																		 R3,R3,LR入栈
    0x080004f2:    4602        .F      MOV      r2,r0				;函数内容自行理解
    0x080004f4:    6810        .h      LDR      r0,[r2,#0]
    0x080004f6:    9000        ..      STR      r0,[sp,#0]
    0x080004f8:    6808        .h      LDR      r0,[r1,#0]
    0x080004fa:    9c00        ..      LDR      r4,[sp,#0]
    0x080004fc:    4420         D      ADD      r0,r0,r4
    0x080004fe:    9000        ..      STR      r0,[sp,#0]
    0x08000500:    6810        .h      LDR      r0,[r2,#0]
    0x08000502:    680c        .h      LDR      r4,[r1,#0]
    0x08000504:    1903        ..      ADDS     r3,r0,r4
    0x08000506:    9800        ..      LDR      r0,[sp,#0]
    0x08000508:    bd18        ..      POP      {r3,r4,pc}			;当函数执行完成后,将R3,R4恢复至函数调用前,																	  并将PC寄存器指向LR存放的地址,即为main中调																		用add完成后下一条指令的地址0x080006d8
```



##### R15(PC) 程序

**正在执行的指令地址就存储在 PC 寄存器中，更改 PC 寄存器的数值就会执行这个数值所对应的地址中的指令**

##### PSR(CPSR, SPSRs)程序状态寄存器

**ARM的PSR寄存器分为CPSR(当前程序状态寄存器)和SPSR(保存程序状态寄存器)，CPSR用来保存程序的各种状态，包括条件标志位，中断标志位，当前处理器模式控制以及其他状态和控制位，在所有模式下均可访问。每个异常模式都有一个SPSR寄存器，当异常来临时，将CPSR的值保存至SPSR，(USER模式和SYS模式均没有SPSR，因为它们不是异常模式)**

![PSR格式](https://pic1.imgdb.cn/item/646dd1d20d2dde577702d605.png)

**PSR位的类型：**

- 预留位：为将来的扩展预留
- 用户可写位：在任意模式下都可写，N，Z，C，V，Q，和GE[3:0]以及E位都是用户可写的。
- 特权模式位：在特权模式下可写，用户模式下写特权位没有效果，A，I，F，T和M[4:0]都是特权位。
- 执行状态位：在特权模式下可写，用户模式下写执行状态位没有效果，J和T位都是执行状态位，在ARM状态下一直为0

- 条件标志位
  N，Z，C，V这四位被称为条件标志位，几乎所有指令都会测试CPSR中的条件标志位来决定指令是否执行。
  条件标志位有两种方式修改：
  1，执行比较指令：CMN，CMP，TEQ以及TST
  2，执行一些其他的算数，逻辑或者移动指令，并且他们的目的寄存器不是R15。

**各个标志位的含义：**

- N：设为指令结果的第31位，如果结果当作二进制补码表示的有符号整型，N=1表示结果是负的；N=0表示结果是非负的。

- Z：指令结果是0的情况下设置为1，反之设为0。

- C：C位的设置存在以下四种情况：
  - 对于加法，包括比较指令CMN，如果加法运算产生无符号溢出，C设为1，反之设为0。
  - 对于减法，包括比较指令CMP，如果减法运算产生一个借位，C就设置为1，反之设为0。
  - 对于包含移位操作的非加减运算，C设为移位器移出的最后一位。
  - 对于其他的非加减运算，C通常保持不变。
  - 这一位的设置存在以下两种情况：
    - 在两个二进制补码的有符号整形的加法和减法指令运算时，如果发生符号溢出，设置为1。
    - 对于非加法和减法指令，V通常是不改变的。

**以上这些标志位还可以通过以下的这些方式进行修改：**

- 执行MSR指令，向CPSR或SPSR寄存器中写入新值。
- 执行目的寄存器是R15的MRC指令，该指令的目的是将些处理器产生的条件标志位传递到ARM处理器中。
- 执行LDM指令的某些变种指令，这些变种指令会复制SPSR的值到CPSR中。这种主要是用于异常返回的情况下。
- 在特权模式执行RFE指令可以从内存中向CPSR中加载新值。
- 执行目的寄存器是R15的算术逻辑指令的设置标志位的变种指令，它们也可以复制SPSR的值到CPSR中，用于异常返回。

#### 基本指令

##### ADD 加指令

```assembly
ADD R0, R1, R2 ;R0 = R1 + R2
ADD R0, R1, #1 ;R0 = R1 + 1
```

##### SUB 减指令

```assembly
SUB R0, R1, R2 ;R0 = R1 - R2
SUB R0, R1, #1 ;R0 = R1 - 1
```

##### LDR 读内存指令

```assembly
LDR R0, [R1, #4] ;将R1的值加4作为地址，将该地址的数据取出来存放到R0
```

##### STR 写内存指令

```assembly
STR R0, [R1, #8] ;将R0的值写入到R1+8这个地址的内存上去
```

##### CMP 比较指令

```assembly
CMP R0, R1 ;比较R0和R1的值保存到PSR寄存器上
```

##### B 跳转指令

**跳转指令分为好几种，包括B，BX，BEQ，BL， BLX等**

```assembly
B func ;最简单的跳转，无条件跳转到func函数地址去，从那里继续执行

CMP R0, #1 
BEQ func ;当CPSR的Z条件码置位后，跳转至func函数执行

BL{条件} func ;将当前PC值保存至LR中，然后跳转至func函数执行，该指令用在需要返回的函数中

BX{条件} func ;带状态切换的跳转，当最低为为1时，切换到Thumb指令执行，最低位为0时切换到ARM指令执行

BLX func ;结合了BL和BX的特点
```

###  中断上下文保护

**通过对函数的反汇编分析,可以看到如果函数中用到了R4-R11寄存器,那么就必须在函数开始将修改的这些寄存器和LR寄存器提前PUSH入栈,对寄存器进行保护**

```c
int func(int v1, int v2,int v3,int v4,int v5,int v6,int v7,int v8,int v9,int v10,int v11,int v12,int v13,int v14,int v15,int v16);

//该函数部分反汇编如下
i.func
func
    0x08000516:    e92d4ff8    -..O    PUSH     {r3-r11,lr}
    0x0800051a:    4604        .F      MOV      r4,r0
    0x0800051c:    ad0a        ..      ADD      r5,sp,#0x28
    0x0800051e:    e89510e0    ....    LDM      r5,{r5-r7,r12}
    0x08000522:    e9ddab10    ....    LDRD     r10,r11,[sp,#0x40]
```

```c
void send_addr(void *addr)
{
	char temp[12] = {0};
	sprintf(temp, "0x%p",addr);
	temp[10] = '\r';
	temp[11] = '\n';
	usart1_send_string(temp, 12);
}

//部分反汇编如下
i.send_addr
send_addr
    0x0800071c:    b53e        >.      PUSH     {r1-r5,lr}
    0x0800071e:    4604        .F      MOV      r4,r0
    0x08000720:    2000        .       MOVS     r0,#0
    0x08000722:    9000        ..      STR      r0,[sp,#0]
    0x08000724:    9001        ..      STR      r0,[sp,#4]
```

**以上几个例子中,当函数的局部变量过多时,不可避免会使用R4-R11这些寄存器用来保存临时变量,所以函数在调用前需要保存使用的寄存器到栈中,函数退出前将栈中的数据弹出,恢复R4-R11到调用前的状态**

**以上是普通函数调用时的寄存器保护流程,但是在中断里,程序并不知道什么时候中断会来临,并打断当前执行的程序,跳转到中断对应的服务函数中去**

```c
//以下面这段代码为例
//main()中:
volatile int x = 10, y =20;
volatile int reslut = 0;
reslut = add(&x, &y);

//main()反汇编:
i.main
main
    ...
	0x080006c4:    200a        .       MOVS     r0,#0xa				;把10保存到R0中, 等同于int x = 10
    0x080006c6:    900e        ..      STR      r0,[sp,#0x38]		;把R0保存到栈中
    0x080006c8:    2014        .       MOVS     r0,#0x14			;把20保存到R0中, 等同于int y = 20
    0x080006ca:    900d        ..      STR      r0,[sp,#0x34]		;把R0保存到栈中
    0x080006cc:    2000        .       MOVS     r0,#0				;把0保存到R0中, 等同于int reslut = 0
    0x080006ce:    900c        ..      STR      r0,[sp,#0x30]		;把R0保存到栈中	
    0x080006d0:    a90d        ..      ADD      r1,sp,#0x34			;把sp指针+34的值放到R1寄存器,相当于&y
    0x080006d2:    a80e        ..      ADD      r0,sp,#0x38			;把sp指针+38的值放到R0寄存器,相当于&x
    0x080006d4:    f7ffff0c    ....    BL       add ; 0x80004f0		;将下一条指令的地址0x080006d8存到LR寄存																		 器,然后跳转到add函数
    0x080006d8:    900c        ..      STR      r0,[sp,#0x30]		;PC指向该条指令
    0x080006da:    980e        ..      LDR      r0,[sp,#0x38]
    ...

//当中断来临时,例如SVC_Handler(任何一个中断都可以)
void SVC_Handler(void)
{
	usart1_send_string("svc fault comming\r\n", 19);
}
//SVC_Handler反汇编如下
i.SVC_Handler
SVC_Handler
    0x08000488:    b510        ..      PUSH     {r4,lr}
    0x0800048a:    2113        .!      MOVS     r1,#0x13
    0x0800048c:    a001        ..      ADR      r0,{pc}+8 ; 0x8000494
    0x0800048e:    f000f9ad    ....    BL       usart1_send_string ; 0x80007ec
    0x08000492:    bd10        ..      POP      {r4,pc}
//可以看到中断的反汇编和其他函数并没有什么不同,同样只保存了R4-R11,LR寄存器
	
//当中断到来时, main中的函数可能还在执行下面的指令
	0x080006c4:    200a        .       MOVS     r0,#0xa				;把10保存到R0中, 等同于int x = 10
	0x080006c6:    900e        ..      STR      r0,[sp,#0x38]		;把R0保存到栈中
--->0x080006c8:    2014        .       MOVS     r0,#0x14			;把20保存到R0中, 等同于int y = 20
	0x080006ca:    900d        ..      STR      r0,[sp,#0x34]		;把R0保存到栈中
	0x080006cc:    2000        .       MOVS     r0,#0				;把0保存到R0中, 等同于int reslut = 0
	0x080006ce:    900c        ..      STR      r0,[sp,#0x30]		;把R0保存到栈中
        
//此时的程序还在使用R0寄存器, 但是程序跳去执行了SVC_Handler函数,然而在中断函数中也用到了R0寄存器,同时在反汇编中没看到保护R0的操作,那么CPU是如何保证执行完中断后继续执行中断前的代码的?
//实际上, 在中断来临时,CPU会自动保存R0-R3,R12,LR,PC,xPSR寄存器, 中断执行完成后,CPU自动恢复这些寄存器
```

```assembly
;可以使用以下指令进行寄存器的保护
STMFD SP!, {R0-R3,LR,PC, PSR} 
; 从PSR,PC, LR, R3-R0依次将寄存器存入栈中
```



**也就是说, 中断的上下文保护是由两个部分完成的, 一部分由CPU自动完成保存&恢复, 另一部分R4-R11由软件自动保存&恢复, 示意图如下**

![image-20230605004837698](https://pic.imgdb.cn/item/647cc00f1ddac507cca9f5f3.png)

## 3. 第一个自定义任务

### 0. 任务的定义

**什么是任务? 可以这样理解,一个任务就像一个单独的程序,任务里面有一个while循环, 在循环里面不停地做某些工作,这个循环永远不会退出, 一个RTOS中同时运行着很多的任务, 根据不同的方案进行任务的切换**

**任务需要哪些东西?**

#### 任务栈

**每一个任务都需要有一个任务栈, 由于任务在运行期间, 对于全局变量,代码段, 这些东西可以通过地址进行访问, 但是在函数中定义的局部变量, 却需要保存在栈中, 如果多个任务使用同一个栈, 就会导致上一个任务保存的栈帧被破坏, 从而无法恢复到上一个任务, 所以, 每一个任务都需要一个单独的任务栈**

#### 任务入口

**任务需要从某个地址进入运行, 这个地址通常是用户自定义的C函数, 通过该地址, OS就能够跳转到该位置运行**

#### 任务优先级

**不同的任务优先级也不一样, 有些任务需要高实时性,有些任务只需要空闲时运行, 所以给任务制定优先级能够更好地完成工作**

**通过以上定义, 可以将一个任务以结构体定义出来, 代码如下:**

```c
typedef void (*TASK_FUNCTION)(void*);	//任务入口函数

typedef enum{
	None = 0x0,				//无优先级
	PriorityIDLE,			//空闲优先级
	PriorityNormal,			//普通优先级
	PriorityHigh,			//高优先级
	PriorityRealTime		//实时优先级
}TASK_PRIORITY;				//任务优先级, 数字越大任务的优先级也就越高


typedef struct{
	TASK_FUNCTION f;		//任务入口函数
	void* param;			//任务参数
	TASK_PRIORITY priority; //任务优先级
	uint8_t *stack_base; 	//任务栈基地址
	const char *task_name;	//任务名
	uint32_t stack_size;	//任务栈大小
	uint32_t *stack_top;	//任务栈顶
	uint32_t *cur_stack;	//当前栈指针
	uint8_t isFirstRun;		//是否第一次运行
}TASK_TCB;					//Task Control Block任务控制块, 包含了一个任务的基本信息
```

**除了单个的任务之外, 还需要一个优先队列来保存创建的任务, 根据优先级来决定调度器的调度策略, 优先队列的节点包装了TCB以及这个任务的状态信息等等**

```c
typedef enum{
	TASK_DISABLED = 0x0,	//禁止调度
	TASK_READY = 0x1,		//准备好调度
	TASK_EXEC,				//正在运行
	TASK_TERM,				//挂起
	TASK_DELAY,				//延时
}TASK_STATUS;

typedef struct TASK_NODE{
	TASK_TCB task;			//任务控制块
	TASK_STATUS status;		//任务状态
	uint32_t delay_tick;	//延时的tick
	struct TASK_NODE *pre;	//上一个任务节点 低优先级
	struct TASK_NODE *next;	//下一个任务节点 高优先级
}TASK_NODE;

typedef struct{
	TASK_NODE tasks[MAX_TASK_COUNT];	//所有的任务节点数组
	uint32_t size;						//当前任务数量
	TASK_NODE *highest_node;			//最高优先级任务
	TASK_NODE *lowest_node;				//最低优先级任务
}TASK_LIST;
```

**通过TASK_NODE结构体包装了一个TASK_TCB变量, 还有任务状态,任务延时等信息, 整个任务通过一个数组模拟实现了一个优先队列, 通过pre/next指针指向低/高优先级的节点, 以此获得一条从最低优先级到最高优先级的静态链表, 这样调度器可以直接获取优先级最高的任务**

### 1. 创建任务

**创建任务即在优先队列中添加一个TASK_NODE, 并根据优先级关系更新最高和最低优先级指针, 具体代码如下**

```c
static TASK_TCB* list_add_task(const char *task_name, TASK_FUNCTION func, void* param, TASK_PRIORITY priority, uint8_t *stack, uint32_t stack_size)
{
	TASK_TCB *tcb = NULL;
	TASK_NODE *node = NULL;
	if (os_task_list.size >= MAX_TASK_COUNT) {
		return tcb;
	}
	if (os_task_list.size == 0) {
		task_tcb_init(&os_task_list.tasks[os_task_list.size].task, task_name, func, param, priority, stack, stack_size);
		tcb = &os_task_list.tasks[os_task_list.size].task;
		os_task_list.highest_node = &os_task_list.tasks[os_task_list.size];
		os_task_list.lowest_node = &os_task_list.tasks[os_task_list.size];
		os_task_list.tasks[os_task_list.size].next = NULL;
		os_task_list.tasks[os_task_list.size].pre = NULL;
		node = &os_task_list.tasks[os_task_list.size];
		os_task_list.size ++;
	} else {
		task_tcb_init(&os_task_list.tasks[os_task_list.size].task, task_name, func, param, priority, stack, stack_size);
		tcb = &os_task_list.tasks[os_task_list.size].task;
		node = &os_task_list.tasks[os_task_list.size];
		os_task_list.size ++;
		if (os_task_list.highest_node == os_task_list.lowest_node) {
			if (priority > os_task_list.highest_node->task.priority) {
				os_task_list.lowest_node->next = node;
				os_task_list.highest_node = node;
				node->pre = os_task_list.lowest_node;
			} else {
				os_task_list.highest_node->pre = node;
				os_task_list.lowest_node = node;
				node->next = os_task_list.highest_node;
			}
		} else {
			if (node->task.priority <= os_task_list.lowest_node->task.priority) {
				os_task_list.lowest_node->pre = node;
				node->next = os_task_list.lowest_node;
				node->pre = NULL;
				os_task_list.lowest_node = node;
			} else if (node->task.priority >= os_task_list.highest_node->task.priority) {
				os_task_list.highest_node->next = node;
				node->pre = os_task_list.highest_node;
				node->next = NULL;
				os_task_list.highest_node = node;
			} else {
				TASK_NODE *tmp = os_task_list.lowest_node;
				while (tmp->next->task.priority <= node->task.priority) {
					tmp = tmp->next;
				}
				node->next = tmp->next;
				tmp->next->pre = node;
				tmp->next = node;
				node->pre = tmp;
			}
		}
	}
	node->status = TASK_READY;

	/* 第一次进入函数没有编译器和cpu帮助保存恢复寄存器,所以需要伪造现场 */
	int *stack_top = (int *)(stack + stack_size);
	tcb->stack_top = stack_top;
	stack_top -= 16; //需要保存16个寄存器 PSR, PC, LR, R12, R3, R2, R1, R0, R11, R10, R9, R8, R7, R6, R5, R4

	/*R4-R11*/
	stack_top[0] = 0; //R4
	stack_top[1] = 0; //R5
	stack_top[2] = 0; //R6
	stack_top[3] = 0; //R7
	stack_top[4] = 0; //R8
	stack_top[5] = 0; //R9
	stack_top[6] = 0; //R10
	stack_top[7] = 0; //R11

	/*R0-R3*/
	stack_top[8] = (int)param; //R0 函数参数
	stack_top[9] = 0; //R1
	stack_top[10] = 0; //R2
	stack_top[11] = 0; //R3

	/*R12, LR, PC, PSR*/
	stack_top[12] = 0; //R12
	stack_top[13] = 0; //LR 没有返回地址,所以设置为0
	stack_top[14] = (int)func; //PC 设为函数入口地址
	// stack_top[15] = 0x01000000; //PSR
	stack_top[15] = (1 << 24); //PSR 将Thumb位设置为1,使用Thumb指令集

	tcb->cur_stack = (uint32_t*)stack_top;	//保存当前栈指针
	tcb->isFirstRun = 1;
	return tcb;
}

static void task_tcb_init(TASK_TCB* tcb, const char *task_name, TASK_FUNCTION func, void* param, TASK_PRIORITY priority, uint8_t *stack, uint32_t stack_size)
{
	tcb->task_name = task_name;
	tcb->f = func;
	tcb->param = param;
	tcb->priority = priority;
	tcb->stack_base = stack;
	tcb->stack_size = stack_size;
}

TASK_TCB* create_task(const char *task_name, TASK_FUNCTION func, void* param, TASK_PRIORITY priority, uint8_t *stack, uint32_t stack_size)
{
	return list_add_task(task_name, func, param, priority, stack, stack_size);
}
```

**该函数主要分为两部分, 一部分是优先队列的插入过程, 一部分是任务栈的伪造现场**

**队列的插入不过多赘述, 读者自行学习数据结构, 主要的部分是伪造现场**

**在前面提到, 一个中断来临时现场的保护分为两个部分, CPU自动保存的8个寄存器以及软件保存的R4-R11寄存器, 由于任务的启动和切换是由中断函数中的调度器完成的, 所以任务的初始化需要将该任务的栈伪造成中断来临后进行现场保护的假象**

```c
/* 第一次进入函数没有编译器和cpu帮助保存恢复寄存器,所以需要伪造现场 */
	int *stack_top = (int *)(stack + stack_size);	//获得栈顶指针
	tcb->stack_top = stack_top;	
	stack_top -= 16; //需要保存16个寄存器,栈顶指针减16x4=64字节位置 PSR, PC, LR, R12, R3, R2, R1, R0, R11, R10, R9, R8, R7, R6, R5, R4

	/*R4-R11*/
	stack_top[0] = 0; //R4
	stack_top[1] = 0; //R5
	stack_top[2] = 0; //R6
	stack_top[3] = 0; //R7
	stack_top[4] = 0; //R8
	stack_top[5] = 0; //R9
	stack_top[6] = 0; //R10
	stack_top[7] = 0; //R11

	/*R0-R3*/
	stack_top[8] = (int)param; //R0 函数参数
	stack_top[9] = 0; //R1
	stack_top[10] = 0; //R2
	stack_top[11] = 0; //R3

	/*R12, LR, PC, PSR*/
	stack_top[12] = 0; //R12
	stack_top[13] = 0; //LR 没有返回地址,所以设置为0
	stack_top[14] = (int)func; //PC 设为函数入口地址
	// stack_top[15] = 0x01000000; //PSR
	stack_top[15] = (1 << 24); //PSR 将Thumb位设置为1,使用Thumb指令集

	tcb->cur_stack = (uint32_t*)stack_top;	//保存当前栈指针
```

**在main函数中, 调用create_task函数, 就可以添加一个任务**

```c
volatile static __attribute__((__aligned(4))) uint8_t task1_stack[512]; //任务栈, 栈需要4字节对齐, 使用__aligned(4)修饰

//任务1函数入口
void task1(void *arg)
{
	volatile int a = 0;
	char buffer[30] = "this is task1 num:   \r\n";
	uint8_t flag = 0;
	int tmp_var = 0;
	while(1) {
		//do something
		buffer[19] = (char)(tmp_var / 100 + 48);
		buffer[20] = (char)((tmp_var % 100) / 10 + 48);
		buffer[21] = (char)(tmp_var % 10 + 48);
		usart1_send_string(buffer, 23);
		flag = ! flag;
		tmp_var++;
		if (tmp_var > 100) {
			tmp_var = 0;
		}
		task_delay(1);
		task1_usage = stack_usage();
		delay(1000000);
	}
}

int mymain()
{
	create_task("task1", task1, "task1", PriorityNormal, (uint8_t*)task1_stack, 512);

	os_start_schedule();	//开启调度
	return 0;
}
```

### 2. 启动任务

**在本项目中, 任务的调度依赖系统定时器中断, 通过实现Systick_Handler中断函数, 每隔1s进行一次调度, 由于任务的切换涉及到栈的切换, 为了能够在中断来后对当前栈进行保护, 所以使用汇编函数将Systick_Handler进行包装, 具体代码如下**

```assembly
				EXPORT	__Vectors
				IMPORT	HardFault_Handler
				IMPORT	UsageFault_Handler
				IMPORT	SVC_Handler
				IMPORT	SysTick_Handler
				EXPORT 	SysTick_Handler_asm
				EXPORT	UsageFault_Handler_asm
; 中断向量表
__Vectors		DCD		0
				DCD		Reset_Handler				; Reset Handler
				DCD     0                ; NMI Handler
                DCD     HardFault_Handler        	; Hard Fault Handler
                DCD     0          ; MPU Fault Handler
                DCD     0           ; Bus Fault Handler
                DCD     UsageFault_Handler_asm         	; Usage Fault Handler
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     SVC_Handler                ; SVCall Handler
                DCD     0           ; Debug Monitor Handler
                DCD     0                          ; Reserved
                DCD     0             ; PendSV Handler
                DCD     SysTick_Handler_asm            ; SysTick Handler

; 系统定时器的中断服务函数
SysTick_Handler_asm PROC
				BL 		SysTick_Handler		;跳转到SysTick_Handler
				ENDP
```

**在.c中定义了SysTick_Handler函数, 参数如下:**

```c
void SysTick_Handler(void)
{
	SCB_Type * SCB = (SCB_Type *)SCB_BASE_ADDR;
	/* clear exception status */
	SCB->ICSR |= SCB_ICSR_PENDSTCLR_Msk;		//清除系统中断标志位, 防止一直进入中断
	
	//os调度
	os_schedule();
	
}
```

**在SysTick_Handler中调用了os的调度函数, 该函数用来进行任务的切换**

```c
THREAD_LOCAL TASK_NODE* cur_task = NULL; 	//当前任务的控制块
TASK_NODE *idle_task = NULL;				//空闲任务的控制块, 该任务由os创建, 
TASK_NODE *delay_node = NULL, *init_node = NULL, *next_node = NULL;
void os_schedule(void)
{
    //判断os是否启动
	if (is_started()) {
		//遍历任务链表, 将有延时的任务的延时时间减1
		delay_node = os_task_list.highest_node;
		while (delay_node != NULL) {
			if (delay_node->status == TASK_DELAY) {
				delay_node->delay_tick --;
				if (delay_node->delay_tick == 0) {
					delay_node->status = TASK_READY;
				}
			}
			delay_node = delay_node->pre;
		}

		//有两个可能, 系统没有开始调度,cur_task为NULL, 系统已经开始调度,cur_task不为NULL
		if (cur_task == NULL) {
			//系统第一次开始调度, 从最高优先级的任务开始调度
			init_node = os_task_list.highest_node;
			//找到最高优先级且状态为TASK_READY的任务
			while(init_node != NULL && init_node->status != TASK_READY) {
				init_node = init_node->pre;
			}
			if (init_node != NULL) {
				init_node->status = TASK_EXEC;
				cur_task = init_node;
				//由于系统第一次调度,所以可以直接从函数入口地址开始运行
				usart1_send_string("os_start_task\r\n", 15);
				cur_task->task.isFirstRun = 0;
				start_task((int)cur_task->task.cur_stack, 0xFFFFFFF9);	//启动任务
			}
		} else {
			//系统不是第一次开始调度, 需要找到下一个调度任务
			next_node = os_task_list.highest_node;
			//找到最高优先级且状态为TASK_READY的任务
			while(next_node != NULL && next_node->status != TASK_READY) {
				next_node = next_node->pre;
			}
			if (next_node != NULL) {
				cur_task = next_node;
				if (cur_task->task.isFirstRun) {
					//切换的任务为第一次进入调度
					cur_task->status = TASK_EXEC;
					cur_task->task.isFirstRun = 0;
					//由于任务第一次运行, 直接从函数入口地址开始运行
					usart1_send_string("start_task\r\n", 12);
					start_task((int)cur_task->task.cur_stack, 0xFFFFFFF9);
				} else {
					//切换的任务不是第一次进入调度, 需要从上一次中断的地方开始运行
					cur_task->status = TASK_EXEC;
					usart1_send_string("resume_task\r\n", 13);
					resume_task((int)cur_task->task.cur_stack, 0xFFFFFFF9);
				}
			} else {
				//找不到任何可以调度的任务, 恢复到IDLEtask
				cur_task = idle_task;
				cur_task->status = TASK_EXEC;
				resume_task((int)cur_task->task.cur_stack, 0xFFFFFFF9);
			}
		}
	} else {
		return;
	}
}
```

**其中启动任务的函数就是start_task这个函数, 这个函数是内联汇编函数, 传入两个参数, 一个是任务栈指针一个是LR寄存器, 代码如下**

```c
static __asm void start_task(int stack, int lr_)
{
	//R0是栈指针, R1是返回地址
	//从任务栈中将R4-R11读出来写入寄存器
	LDMIA R0!, {R4-R11}

	//更新SP
	MSR MSP, R0

	//触发硬件中断返回
	BX R1
}
```

**在前面提到, ARM的中断函数中,CPU会自动保存一部分寄存器, 然后软件再保存一部分, 那么当中断退出的时候, 寄存器又是怎么恢复的呢, 实际上, 当CPU进入中断的时候, LR寄存器并不会指向中断来临前的地址,而是会被赋一个特殊的值, 0xFFFFFFF9, 中断退出时, CPU跳转到该地址, 将栈中的数据恢复到PSR, PC, LR, R12, R0-R3这些寄存器上去, 所以启动任务实际上就是模拟退出中断的过程, 首先从目标任务栈中恢复R4-R11寄存器, 然后把当前栈指针更新为目标任务的栈指针, 最后跳转到0xFFFFFFF9, 硬件恢复寄存器**

### 3. 切换任务

**当系统从主栈进入第一个任务栈后, 后续的所有任务的切换全是从当前任务栈切换到下一个任务的任务栈, 和从主栈切换到任务栈不同的是, 从主栈切换到任务栈不需要保存主栈的栈帧, 直接切换到任务栈就行, 因为开始调度任务后永远不会回到主栈, 但是从一个任务栈切换到另一个任务栈之后还是会调度回当前的任务, 所以需要切换前保存当前的栈帧**

**本RTOS里, 在SysTick_Handler_asm函数中, 对在调用SysTick_Handler之前进行了栈帧的保护, 代码如下:**

```assembly
SysTick_Handler_asm PROC
				; 保存现场
				IMPORT |cur_task|		;cur_task是外部定义的指向当前任务的指针
				LDR R0, =|cur_task|		;R0 = &cur_task 获得指针的地址
				LDR R0, [R0, #0]		;R0 = *R0		解引用指针
				CMP R0, #0				;将该指针指向的值和0相比较
				BEQ DO_NOT_SAVE_STACK	;如果等于0的话说明指针为NULL, 当前为主栈直接跳转到DO_NOT_SAVE_STACK,不进行栈帧的保存
				
SAVE_OLD_STACK
				STMDB SP!, {R4-R11}		;软件将R4-R11保存至栈中
				LDR R0, =|cur_task|		;获取指针的地址
				LDR R0, [R0, #0]		;获取指针指向的TCB的地址
				STR SP, [R0, #0x1c]		;将当前SP栈寄存器保存到TCB*->cur_stack中
				
DO_NOT_SAVE_STACK
				BL 		SysTick_Handler
				ENDP
```

**STR SP, [R0, #0x1c]这一步为什么是0x1c, 这要从TCB结构体的定义说起**

```c
typedef struct{
	TASK_FUNCTION f;		//任务入口函数
	void* param;			//任务参数
	TASK_PRIORITY priority; //任务优先级
	uint8_t *stack_base; 	//任务栈基地址
	const char *task_name;	//任务名
	uint32_t stack_size;	//任务栈大小
	uint32_t *stack_top;	//任务栈顶
	uint32_t *cur_stack;	//当前栈指针
	uint8_t isFirstRun;		//是否第一次运行
}TASK_TCB;

typedef enum{
	TASK_DISABLED = 0x0,	//禁止调度
	TASK_READY = 0x1,		//准备好调度
	TASK_EXEC,				//正在运行
	TASK_TERM,				//挂起
	TASK_DELAY,				//延时
}TASK_STATUS;

typedef struct TASK_NODE{
	TASK_TCB task;			//任务控制块
	TASK_STATUS status;		//任务状态
	uint32_t delay_tick;	//延时的tick
	struct TASK_NODE *pre;	//上一个任务节点 低优先级
	struct TASK_NODE *next;	//下一个任务节点 高优先级
}TASK_NODE;
```

**TASK_NODE中TASK_TCB的偏移为0, 而在TASK_TCB中cur_stack的偏移为28字节, 前面有6个指针变量和一个4字节的枚举, 总共28字节, 十六进制表示为0x1c, 所以将SP寄存器需要保存在cur_task+0x1c位置的地方**

![GIF 2023-6-9 14-29-37](https://pic.imgdb.cn/item/6482c6ff1ddac507cc84350d.gif)

## 4. 如何优化调度器

## 5. 添加定时器模块

## 6. 添加互斥锁

## 7. 添加信号量

## 8. TODO
