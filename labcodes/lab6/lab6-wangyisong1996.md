# Lab 6 实验报告

## 练习0：填写已有实验

已将 Lab 1 - 5 的代码填写到相应位置，并按照要求修改代码，使其能通过编译并通过 `make grade` 中除 priority 之外的测试用例。

## 练习1: 使用 Round Robin 调度算法（不需要编码）

### 请理解并分析sched_class中各个函数指针的用法，并结合 Round Robin 调度算法描ucore的调度执行过程

- init(struct run_queue *rq) : 初始化一个进程调度队列，传入的参数为该队列的指针
- enqueue(struct run_queue *rq, struct proc_struct *proc) : 将 proc 这个进程插入 rq 这个进程调度队列
- dequeue(struct run_queue *rq, struct proc_struct *proc) : 将 proc 这个进程从 rq 这个进程调度队列中删除
- struct proc_struct * pick_next(struct run_queue *rq) : 从 rq 这个队列中取出下一个可以运行的进程
- proc_tick(struct run_queue *rq, struct proc_struct *proc) : 处理 proc 这个进程收到的时钟中断

ucore 的调度执行过程
- 首先在 kern_init 中调用 sched_init ，即创建一个进程调度队列，并初始化 sched_class （在这里是 Round Robin）
- 然后会调用 wakeup_proc ，将处于就绪态的进程加入队列中。在有进程变为就绪态的时候，也会调用该函数
- 从第一个进程运行开始，每发生一次中断/系统调用，在从内核返回进程（用户态，除 idleproc 外）之前，都会调用 schedule 
- schedule 的执行过程为，先将当前运行的进程加入队列，然后使用调度算法，从队列中选择下一个要运行的进程，然后切换到该进程的上下文
- 当发生时钟中断时，内核会调用 sched_class_proc_tick ，使调度算法处理该时钟中断，如 Round Robin 算法中，一个进程收到一定数量的时钟中断，将时间片用完后，就会让出CPU资源给其他进程

### 请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

- 首先将单个进程调度队列改为多个，其中第 i 个的时间片长度为 2^i
- 所有进程刚开始运行的时候，都在第一个进程调度队列中
- 如果某个进程的时间片用完而没有执行完，就将其移到下一个队列中（即，下次调度它的时间片长度 * 2）
- 如果某个进程在时间片用完之前就执行完（即变为等待状态），就将其移到上一个队列中（即，下次调度它的时间片长度 / 2）
- 多个调度队列之间应该存在不同的优先级，但是必须保证低优先级的队列也能得到调度，否则会产生饥饿

## 练习2: 实现 Stride Scheduling 调度算法（需要编码）

简要设计实现过程：
- BIG_STRIDE 的值设为 100，因为在各种测试用例中，进程的优先级实际上都不超过 10
- 在 stride_init 中，初始化 run_queue 中的就绪队列，并初始化一个斜堆的指针（为 NULL）
- 在 stride_enqueue 中，将 proc 的 lab6_run_pool 插入 run_queue 的斜堆中，其他操作参考 Round Robin 的实现
- 在 stride_dequeue 中，将 proc 的 lab6_run_pool 从 run_queue 的斜堆中删除，其他操作同 Round Robin 的实现
- 在 stride_pick_next 中，从斜堆的顶端取得 stride 值最小的进程，然后给它的 stride 加上 BIG_STRIDE / 它的优先级，并返回它的指针
- stride_proc_tick 的实现同 Round Robin 中相应的实现

# 与参考答案的区别

我阅读了 Lab 6 参考答案的相关部分代码，发现只有一处细节上的区别：
在增加一个进程的 stride 值得时候，为避免发生除零错误，参考答案特判了优先级为 0 的情况，而我是在初始化一个进程的优先级时，就设置为 1 。
我认为参考答案的处理方法更好，能处理用户将优先级设置为 0 的情况。

