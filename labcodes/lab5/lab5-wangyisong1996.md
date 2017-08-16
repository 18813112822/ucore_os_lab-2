# Lab 5 实验报告


## 练习 0：填写已有实验
已将 LAB1-4 的代码填入本实验的代码中，并按照注释要求进行改进，能正常编译运行。


## 练习 1: 加载应用程序并执行（需要编码）	
do_execv函数调用load_icode（位于kern/process/proc.c中）来加载并解析一个处于内存中的ELF执行文件格式的应用程序，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc_struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。需设置正确的trapframe内容。

### 简要设计实现过程
1. 设置 tf_cs 为 USER_CS ，tf_ds, tf_es 和 tf_ss 为 USER_DS，这样能切换到用户态代码段和数据段
2. 设置 tf_esp 为 USTACKTOP，即用户态的栈顶地址
3. 设置 tf_eip 为 elf->e_entry，即该用户态进程的第一条指令地址
4. 将 tf_eflags 或上 FL_IF ，即允许产生中断

### 描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。
用户态进程进入RUNNING态时，ucore调用 proc_run 来切换到该进程，具体来说，首先关中断，然后切换栈、页表、上下文，最后开中断，等到这次系统调用或中断结束时，就通过设置好的上下文和trapframe，使用iret返回到新进程去。


## 练习 2: 父进程复制自己的内存空间给子进程（需要编码）

### 请补充copy_range的实现，确保能够正确执行
以补充copy_range的实现，并使用make grade测试，能正确执行所有测试用例。

### 请在实验报告中简要说明如何设计实现”Copy on Write 机制“，给出概要设计，鼓励给出详细设计
在fork的时候，将父进程和子进程的页表都改为只有读权限，然后当这两个进程中的任意一个想要修改其中的内容时，就会产生一个page fault，
这时候就可以将对应的页复制一份，并修改复制过的页，同时赋予其写权限。这样即可实现COW机制。

## 练习 3: 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）

### 请在实验报告中简要说明你对 fork/exec/wait/exit函数的分析。
fork/exec/wait/exit都是系统调用，在ucore lib的实现中，最终都会调用syscall函数，然后通过软中断的方式来实现。

### 请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？
通过调用sys_fork/sys_exec/sys_wait/sys_exit，然后调用syscall函数，再通过软中断的方式进入ucore内核，然后由内核来改变进程的执行状态。

### 请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。（字符方式画即可）              
  alloc_proc                                 RUNNING
      +                                   +--<----<--+
      +                                   + proc_run +
      V                                   +-->---->--+ 
PROC_UNINIT -- proc_init/wakeup_proc --> PROC_RUNNABLE -- try_free_pages/do_wait/do_sleep --> PROC_SLEEPING --
                                           A      +                                                           +
                                           |      +--- do_exit --> PROC_ZOMBIE                                +
                                           +                                                                  + 
                                           -----------------------wakeup_proc----------------------------------


**如果不能编译，请看https://asciinema.org/a/a83u3isa1l0e9s4dnptw56tlo**

