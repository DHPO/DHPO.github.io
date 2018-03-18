---
title: OS Lab1 实验报告
date: 2018-03-18 20:46:06
tags:
  - mit6.828
  - OS
---

实验内容:https://ipads.se.sjtu.edu.cn/courses/os/labs/lab1.html

## 环境配置

### gcc降级

需要注意的是这个lab只能使用**gcc4.8及以下版本**编译，否则运行会有问题。

gcc降级的最简单的方法是将`/usr/bin`目录下的`gcc`删去，并做一个到`gcc-4.8`的符号链接：

```bash
cd /usr/bin
sudo rm gcc
sudo ln -s gcc-4.8 gcc
```

如果没有安装gcc4.8，就先安装一下：

```bash
sudo apt install gcc-4.8
```

### 32位依赖包的安装

在编译的时候如果出现`__udivdi3 not found`或`__muldi3 not found`的问题，可能是缺少依赖包导致的：

```bash
sudo apt-get install gcc-multilib
```

需要注意的是，如果问题依然存在，可能上述的依赖包是给gcc5以上使用的，安装给gcc4.8使用的依赖包即可：

```bash
sudo apt install gcc-4.8-multilib
```

### qemu安装

这个没什么坑，按照[tools](https://ipads.se.sjtu.edu.cn/courses/os/labs/tools.html)说的做就行了

## 系统的启动流程

刚启动时，内存的布局是这样的（都是物理地址）

```
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

启动的步骤大概是这样的：

1. 从ROM中加载BIOS程序，从`0xffff0`开始执行
2. BIOS将磁盘(或其他设备)的第一个扇区加载至`0x7c00`并执行（即boot loader,`/boot/boot.S`）
3. boot loader 在完成硬件初始化、启动保护模式、加载kernel等任务后，将控制权转移到kernel(`/kern/entry.S`)
4. entry.S加载页表，设置esp、ebp，之后将控制权转移到init.c中的`i386_init`之后就是初始化显示设备、启动console之类的工作

启动过程中的注意点如下：

1. 注意区别BIOS和boot loader，我之前一直把BIOS加载boot loader和boot loader加载kernel当作一回事，然后就十分僵硬

2. BIOS和boot loader 都经历了从实模式到保护模式的变化(i8086 => i386)

3. kernel的加载位置是0x10000，而entry的位置是0x10000c（根据ELF的格式，在加载完成后0x10018处的数据就是entry的地址）

4. 页表在entrypgdir.c中，映射方式为：

   - [0, 4MB) => [0, 4MB) 只读
   - [KERNBASE + 0, KERNBASE + 4MB) => [0, 4MB）可写

   因此在启用虚拟地址后依然可以使用物理地址取指

5. boot loader在磁盘的第一个sector，kernel从第二个sector开始（见/boot/main.c的注释）

6. ljmp和非本地跳转的longjump不是一回事

## 可变参数数量的函数

理论上，只需要caller将参数按照顺序压栈，callee按照首个参数的地址依次+4的顺序读内存，即可得到所有的参数，实现可变参数数量。

但是这样有两个问题：

- 需要知道参数的数据类型
- 需要知道参数的数量

stdarg.h对上述操作进行了一定程度的封装。

### stdarg.h

stdarg.h中存在如下的宏定义：

```c
#define _INTSIZEOF(n) ((sizeof(n)+sizeof(int)-1)&~(sizeof(int) - 1) ) // size对齐
#define va_start(ap,v) ( ap = (va_list)&v + _INTSIZEOF(v) ) //第一个可选参数地址
#define va_arg(ap,t) ( *(t *)((ap += _INTSIZEOF(t)) - _INTSIZEOF(t)) ) //下一个参数地址
#define va_end(ap) ( ap = (va_list)0 ) // 将指针置为无效
```

以lab中的代码为例：

printf.c:25

```c
int
cprintf(const char *fmt, ...)
{
	va_list ap;
	int cnt;

	va_start(ap, fmt);
	cnt = vcprintf(fmt, ap);
	va_end(ap);

	return cnt;
}
```

printfmt.c:158

```c
case '*':
	precision = va_arg(ap, int);
	goto process_precision;
```

其中`va_start`使用第一个参数的地址初始化参数列表`ap`，之后每次使用时传入`ap`和参数类型，从中取出下一个参数。

不过这依然无法解决参数数量的预期和实际不一致的问题，下面的一些练习就与此相关。

## Exercises

### Exercise 3

> At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

gdb记录如下：

```
[   0:7c2d] => 0x7c2d:	ljmp   $0x8,$0x7c32
0x00007c2d in ?? ()
(gdb) 
The target architecture is assumed to be i386
=> 0x7c32:	mov    $0x10,%ax
0x00007c32 in ?? ()
```

说明引起变化的语句是

```assembly
# Jump to next instruction, but in 32-bit code segment.
# Switches processor into 32-bit mode.
  ljmp    $PROT_MODE_CSEG, $protcseg
```

> What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?

boot loader的最后一条是(/boot/main.c:60)

```c
((void (*)(void)) (ELFHDR->e_entry))();
```

kernel的第一条是(/kern/entry.S:44)

```assembly
entry:
	movw	$0x1234,0x472			# warm boot
```

> How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?

ELF的头部提供了信息

### Exercise5

> Reset the machine (exit QEMU/GDB and start them again). Examine the 8 words of memory at 0x00100000 at the point the BIOS enters the boot loader, and then again at the point the boot loader enters the kernel. Why are they different? What is there at the second breakpoint? (You do not really need to use QEMU to answer this question. Just think.)

boot loader 将kernel加载到了这里

### Exercise6

> Trace through the first few instructions of the boot loader again and identify the first instruction that would "break" or otherwise do the wrong thing if you were to get the boot loader's link address wrong. Then change the link address in`boot/Makefrag` to something wrong, run `make clean`, recompile the lab with `make`, and trace into the boot loader again to see what happens. Don't forget to change the link address back and `make clean` afterwards!

即修改/boot/Makefrag:28

```
$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o $@.out $^
```

`0x7c00`修改为其它值后无法启动。BIOS钦定了`0x7c00`来接管，就不要搞一个大新闻了

### Exercise7

> Use QEMU and GDB to trace into the JOS kernel and find where the new virtual-to-physical mapping takes effect. Then examine the Global Descriptor Table (GDT) that the code uses to achieve this effect, and make sure you understand what's going on.

不知道怎么才能用gdb看GDT，据说要运行在ring0才行

> What is the first instruction *after* the new mapping is established that would fail to work properly if the old mapping were still in place? Comment out or otherwise intentionally break the segmentation setup code in `kern/entry.S`, trace into it, and see if you were right.

接下来使用虚拟地址的地方就会出错，也就是这一句(/kern/entry.S:67)：

```assembly
jmp	*%eax
```

这里源代码中也有一个小问题：

> Now paging is enabled, but we're still running at a low EIP(why is this okay?)

关于这一点我在系统的启动流程的注意点4中已经解释过。

### Exercise8

> We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment. Remember the octal number should begin with '0'.

我一个OS的lab怎么第一项工作是做字符串处理？

参考16进制的写法就行了：

```c
case 'o':
	// Replace this with your code.
	// display a number in octal form and the form should begin with '0'
	putch('0', putdat);
	num = getuint(&ap, lflag);
	base = 8;
	goto number;
```

### Exercise9.0

> You need also to add support for the "+" flag, which forces to precede the result with a plus or minus sign (+ or -) even for positive numbers.

思路：如果读到‘+’，那么立一个flag：“需要显示‘+’”；再像平常那样读符号，如果是有符号整数且是正数，那么向输出流中放一个‘+’，再正常显示数字。

```c
case '+':
	signedflag = 1;
	goto reswitch;
/* ... */
case 'd':
	num = getint(&ap, lflag);
	if ((long long) num < 0) {
		putch('-', putdat);
		num = -(long long) num;
	}
	else if (signedflag) {
		putch('+', putdat);
	}
	base = 10;
	goto number;
```



### Exercise9.1

> Explain the interface between `printf.c` and `console.c`. Specifically, what function does `console.c`export? How is this function used by `printf.c`?

这两者的接口为`cputchar`，作用是往输出流中放入一个字符

### Exercise9.2

> Explain the following from `console.c`:
>
> ```c
 if (crt_pos >= CRT_SIZE) {
 	int i;
	memcpy(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
	for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
 		crt_buf[i] = 0x0700 | ' ';
 	crt_pos -= CRT_COLS;
   }
 ```

整体上移一行，并在新的一行中填入空格

### Exercise9.3

> For the following questions you might wish to consult the notes for Lecture 2. These notes cover GCC's calling convention on the x86.
>
> Trace the execution of the following code step-by-step:
>
> ```c
  int x = 1, y = 3, z = 4;
  cprintf("x %d, y %x, z %d\n", x, y, z);
 ```

> -  In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?

`fmt`指向格式化字符串`"x %d, y %x, z %d\n"`，`ap`指向第一个可选参数`x`

### Exercise9.4

> Run the following code.
>
> ```c
 unsigned int i = 0x00646c72;
     cprintf("H%x Wo%s", 57616, &i);
 ```

> What is the output? Explain how this output is arrived out in the step-by-step manner of the previous exercise

output：

```
He110 World
```

原因：

- 57616 == 0xe110
- 0x64 == 'd', 0x6c == 'l', 0x72 == 'r'

> The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?

i需要改为`0x726c6400`,`57616`不需要改

### Exercise9.5

> In the following code, what is going to be printed after`y=`? (note: the answer is not a specific value.) Why does this happen?
>
> ```c
 cprintf("x=%d y=%d", 3);
 ```

一个在栈上相邻的值

### Exercise9.6

> Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf`or its interface so that it would still be possible to pass it a variable number of arguments?

分析：

- 对于`cprintf`而言，需要知道`fmt`的内容才能确定参数的数量
- 对于这一种传参方式，需要知道参数的数量才能确定第一个参数的位置

所以解决方案可能有：

1. 新增一个参数，内容为参数数量，在参数表的末尾
2. 只接受固定的两个参数，第一个为`fmt`，第二个为以NULL结尾的链表的首地址，链表中为参数，类似于`char **argv`
3. 将可选参数放在前面，`fmt`在最后

### Exercise10

> Modify the function `printnum()` in `lib/printfmt.c` to support `"%-"`when printing numbers. With the directives starting with "%-", the printed number should be left adjusted. (i.e., paddings are on the right side.) For example, the following function call:

> ```
 cprintf("test:[%-5d]", 3)
 ```
> , should give a result as
> ```
 "test:[3    ]"
 ```

> (4 spaces after '3'). Before modifying`printnum()`, make sure you know what happened in function`vprintffmt()`.

思路：先统计输出的数字的位数(`O(log n)`)，然后输出数字，再补足空格

```c
if (padc == '-') {
	int i = 0;
	int num_of_digit = 0;
	int temp = num;
	while(temp > 0) {
		num_of_digit += 1;
		temp /= base;
	}
	printnum(putch, putdat, num, base, num_of_digit, ' ');
	for (i = 0; i < width - num_of_digit; i++)
		putch(' ', putdat);
	return;
}
```

### Exercise11

> Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

/kern/entry.S:75

```assembly
# Set the stack pointer
movl	$(bootstacktop),%esp
```

### Exercise12

> To become familiar with the C calling conventions on the x86, find the address of the `test_backtrace` function in `obj/kern/kernel.asm`, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of `test_backtrace` push on the stack, and what are those words?

发现每次调用esp缩小0x20，即8 words

### Exercise13

>  Implement the backtrace function as specified above.

既然已经提供了`read_ebp`函数，那么顺着ebp一路摸上去就能得到`return address`和各个参数：

```c
ebp = (unsigned int *)read_ebp();
for (;ebp != NULL;) {
	cprintf("eip %8x  ebp %8x  args %08x %08x %08x %08x %08x\n", 
			ebp[1], ebp, ebp[2], ebp[3], ebp[4], ebp[5], ebp[6]);
	ebp = (unsigned int *)(*ebp);
}
```

这些在ics的buffer overflow中见的多了。另一个问题是什么时候停止循环。在entry.S中发现:(entry.S:73)

```assembly
movl	$0x0,%ebp			# nuke frame pointer
```

也就是说，初始化时ebp为0，当发现ebp为0(NULL)时即可停止循环。

### Exercise14

> Modify your stack backtrace function to display, for each `eip`, the function name, source file name, and line number corresponding to that `eip`.

仿照上下文，添加：

```c
stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
if (lline <= rline) {
	info->eip_line = rline;
}
else {
	return -1;
}
```

>  where do `__STAB_*` come from？

stab是gcc生成的调试信息，具体格式可以参考：[GCC 生成的符号表调试信息剖析](http://caobeixingqiu.is-programmer.com/posts/12361.html)

### Exercise15

> In this exercise, you need to implement a rather easy "time" command. The output of the "time" is the running time (in clocks cycles) of the command. The usage of this command is like this: "time [command]".

这个在大一暑假作业简易数据库的性能测试中就实现过一个类似，思路是：

1. 记录开始时时间
2. 运行
3. 记录结束时时间

实现如下：

```c
int
mon_time(int argc, char **argv, struct Trapframe *tf)
{
	if (argc != 2)
		return -1;

	uint64_t before, after;
	int i;
	struct Command command;
	/* search */
	for (i = 0; i < NCOMMANDS; i++) {
		if (strcmp(commands[i].name, argv[1]) == 0) {
			break;
		}
	}

	if (i == NCOMMANDS)
		return -1;

	/* run */
	before = read_tsc();
	(commands[i].func)(argc-1, argv+1, tf);
	after = read_tsc();
	cprintf("%s cycles: %d\n", commands[i].name, after - before);
	return 0;
}
```

## 总结

这个lab需要写代码的部分只是真正的冰山一角，甚至大多能在上下文中找到提示。而对这些问题的思考才是这个lab最大的意义。
