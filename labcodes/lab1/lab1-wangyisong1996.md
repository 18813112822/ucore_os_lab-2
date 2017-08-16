# Lab 1 实验报告


## 练习1

### 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

在 Makefile 中，首先定义了各种编译命令（如CC, LD）及其参数们（如CFLAGS），然后引用tools/function.mk定义了一些用来方便编译的函数。
接着定义了kernel中要编译的文件夹，并编译kernel（Makefile第122~第151行）。
然后编译bootblock和签名工具sign，并使用dd命令创建ucore.img（Makefile第155~第186行）。

### 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

通过阅读 tools/sign.c 得知，符合规范的硬盘主引导扇区的最后两个字节应该是 0x55AA 。


## 练习2

### 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

先make，然后make debug，然后发现QEMU执行到了kern\_init，说明需要该gdb的配置。
将tools/gdbinit文件的内容改为"set architecture i8086"和"target remote :1234"之后，make debug，即看到QEMU停在CPU加电后的第一条指令。
这条指令的地址应为0xffff0，而`x /2i $pc`的结果为"0xfff0:      add    %al,(%bx,%si)"，这说明没有考虑段寄存器CS。
改用`x /2i $cs*16+$pc`之后得到正确的输出"0xffff0:     ljmp   $0xf000,$0xe05b"。
接着用`si`命令进行单步执行，即可跟踪BIOS的执行。

### 在初始化位置0x7c00设置实地址断点,测试断点正常。

在gdb中`b *0x7c00`并`continue`，可以看到CPU在0x7c00位置停止执行，断点正常。

### 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

在此时`x /10i $pc`（注：因为这时候CS=0，所以直接用"$pc"而不是"$cs\*16+$pc"是可以的），得到0x7c00位置接下来10条指令。
将这些指令跟bootasm.S和bootblock.asm中的进行比较，发现是一致的。

### 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

"0x7c2a: mov %eax,%cr0" 这是从实模式切换到保护模式的那条指令。
用`b *0x7c2a`设置断点，然后用`continue`继续执行。
可以看到，在0x7c2a之后的一条指令是个ljmp，它的作用是从16位代码段跳到32位代码段。
继续运行之后，需要先`set architecture i386`才能正常使用gdb的反汇编功能。


## 练习3

### 请分析bootloader是如何完成从实模式进入保护模式的。

首先关中断，然后设置各数据段寄存器为0，然后开启A20。
因为一些历史原因，不开启A20地址线会导致无法访问偶数兆的内存地址，所以需要开启它。
开启的方法是向8042键盘控制器发送一些数据，具体可以见bootloader。
开启A20之后，要初始化GDT表，具体来说用一条lgdt指令将GDT表的地址传给CPU即可。
最后进入保护模式，将CR0寄存器的最低位设为1，然后跳转到保护模式的代码段即可。


## 练习4

### bootloader如何读取硬盘扇区的？

在 bootmain.c 中的 readseg 函数是用来读取磁盘扇区的。
具体来说，先根据传入的参数计算出要读取的磁盘扇区的编号，然后对每个扇区调用 readsect 函数来读取。
在 readsect 函数中，先设置读取的扇区的地址等信息（LBA模式的参数），然后等待磁盘准备好，再用一个循环（repne）来读取整个扇区的内容。

### bootloader是如何加载ELF格式的OS？

bootloader 先读取磁盘的第1个（下标从0开始）开始的8个扇区，得到ELF header的信息。
然后根据ELF header里的program header表的位置和长度，依次加载program header表的每一项，具体使用之前的readseg函数来实现。


## 练习5

### 实现kdebug.c里的print\_stackframe函数

大致实现过程：先通过read\_ebp()和read\_eip()得到当前函数的EBP和EIP，然后不断循环找到上一级函数的EBP和EIP，直到EBP=0。
在每一次循环的时候根据EBP指针的信息打印有关参数的值，并从\*EBP处取得新的EBP。


## 练习6

### 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

通过阅读“中断与异常”得知，中断描述符表的每一个表项的大小为8字节，其中16~31位表示段选择子，0~15位和48~63位合在一起表示段内部的偏移，这两部分表示了该中断的处理代码入口。

### 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt\_init。在idt\_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。

实现方法：首先定义数组\_\_vectors，然后对IDT的每一项调用SETGATE宏，注意代码段选择子要用KERNEL\_CS，最后调用lidt函数来加载IDT表。

### 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print\_ticks子程序，向屏幕上打印一行文字”100 ticks”。

实现方法：`if ((++ticks) % TICK_NUM == 0) print_ticks()`。


## 扩展练习

### Challenge 1 

用户态和内核态的切换的重点在于切换各段选择子和栈。
从内核态切换到用户态时，因为iret指令要多弹出用户态的ss和esp，所以在进行系统调用之前要先在栈上留出这8个字节的空位，然后在系统调用中填上相应的值。
从用户态切换到内核态时，因为iret指令会少弹出ss和esp，所以在系统调用的过程中要将整个trapframe移动8个字节，并填上内核的esp值使得iret能正确返回。
具体细节见代码。




