## XV6源代码中断/异常部分阅读报告

​	在GitHub上搜索XV6，找到一个xv6项目代码的镜像 `https://github.com/guilleiguaran/xv6` ，用 `git` 将其克隆到本地。

​	打开项目文件夹，看到一个 `main.c` ，打开可以看到一个 `main` 函数，这个函数就是整个项目的入口。 `main` 函数中调用了很多函数，且很多都是以 `init` 结尾的，可以猜出它们都是初始化的函数。最后 `main` 函数调用了 `mpmain` 函数进入系统的循环。根据这些函数找到相应的代码。

​	`main` 函数中调用了 `tvinit` 函数，其注释为 `trap vectors` ， `mpmain` 函数调用了 `idtinit` 函数，其注释为 `load idt register` ，在 `trap.c` 找到了关于这两个函数的相关代码。

```C
void
tvinit(void)
{
  int i;

  for(i = 0; i < 256; i++)
    SETGATE(idt[i], 0, SEG_KCODE<<3, vectors[i], 0);
  SETGATE(idt[T_SYSCALL], 1, SEG_KCODE<<3, vectors[T_SYSCALL], DPL_USER);
  
  initlock(&tickslock, "time");
}

void
idtinit(void)
{
  lidt(idt, sizeof(idt));
}
```

​	可见 `tvinit` 函数主要调用了 `SETGATE` 函数，对照 `mmu.h` 中的描述，可知这些操作将中断处理程序的地址写入了 `idt` 数组中。而 `idtinit` 函数调用 `lidt` 函数，则是将 `idt` 数组的地址载入到中断向量表寄存器中。而中断处理程序的以及中断向量表都是由 `vectors.pl` 生成的。查看其中代码可知，其生成的中断处理程序的结构如下：

```asm
vector0:
	pushl $0
	pushl $0
	jmp alltraps
vector1:
	pushl $0
	pushl $1
	jmp alltraps
...
```

​	可见中断处理程序会把中断号压入栈中，然后跳转到 `alltraps` ，由它来负责具体处理。在 `trapasm.S` 中找到 `alltraps` 的相关代码。

```asm
.globl alltraps
alltraps:
  # Build trap frame.
  pushl %ds
  pushl %es
  pushl %fs
  pushl %gs
  pushal
```

​	`alltraps` 会继续把寄存器压入栈中保存现场，构造出 `trap frame` 。

```asm
  # Set up data and per-cpu segments.
  movw $(SEG_KDATA<<3), %ax
  movw %ax, %ds
  movw %ax, %es
  movw $(SEG_KCPU<<3), %ax
  movw %ax, %fs
  movw %ax, %gs

  # Call trap(tf), where tf=%esp
  pushl %esp
  call trap
  addl $4, %esp

  # Return falls through to trapret...
.globl trapret
trapret:
  popal
  popl %gs
  popl %fs
  popl %es
  popl %ds
  addl $0x8, %esp  # trapno and errcode
  iret
```

​	然后会重新设置段寄存器，进入内核态，然后调用C函数`trap`来处理中断，处理完返回后弹出栈中寄存器恢复现场。`trap`函数的代码在`trap.c`中。

```C
trap(struct trapframe *tf)
{
	if(tf->trapno == T_SYSCALL){
		if(proc->killed)
    		exit();
    	proc->tf = tf;
    	syscall();
    	if(proc->killed)
    		exit();
    	return;
	}
  	...
}
```

​	`trap` 函数接收一个`trap frame` 指针参数。首先根据`trapno`检查是否是系统调用，如果是系统调用，则判断执行系统调用的进程是否被杀死。如果已杀死，则直接`exit`，否则将进程的`trap frame` 改成传进来的参数，然后用`syscall` 执行系统调用。最后再通过`proc->killed`检查是否已经被杀死。这是`kill`系统调用的需要，该系统调用通过将`killed`置为1来杀死一个进程，因此如果检查到该为为1即可直接将该进程杀死。

```C
switch(tf->trapno){
	case T_IRQ0 + IRQ_TIMER:		...
	case T_IRQ0 + IRQ_IDE:			...
	case T_IRQ0 + IRQ_IDE+1:		...
	case T_IRQ0 + IRQ_KBD:			...
	case T_IRQ0 + IRQ_COM1:			...
	case T_IRQ0 + 7:
	case T_IRQ0 + IRQ_SPURIOUS:		...
	default:						...
}
```

​	接下来用一个`switch` 语句，判断中断产生的原因是硬件中断还是异常，并且分情况处理。如是`trap` 则调用相应的函数处理。

```C
if(proc && proc->killed && (tf->cs&3) == DPL_USER)
	exit();

if(proc && proc->state == RUNNING && tf->trapno == T_IRQ0+IRQ_TIMER)
    yield();

if(proc && proc->killed && (tf->cs&3) == DPL_USER)
    exit();
```

​	在最后再判断是否需要把该进程杀死。如果`killed` 已经被设置为1，并且是在用户空间中运行，则直接`exit`。如果仍然在内核态执行，则让它继续运行直到返回，此时再将它杀死并回收相应的资源。