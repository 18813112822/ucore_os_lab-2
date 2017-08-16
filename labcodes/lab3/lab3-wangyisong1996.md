# Lab 3 实验报告

## 练习0：填写已有实验

已填写 Lab 1 和 Lab 2 所做的内容。

## 练习1：给未被映射的地址映射上物理页

简要实现过程：
- 先调用 `get_pte` 获取该地址的 pte 
- 如果 pte 中的内容为空，则调用 `pgdir_alloc_page` 给该地址分配一个页帧

### 问题一：请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

当PTE的最低位为1时，即`*pte & PTE_P`时，PTE中存的是页表项，
而当这一位为0时，该PTE要么没有对应的页，要么在高24位存了该PTE在swap中的页编号。

### 问题二：如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

硬件发生pgfault中断，保存现场并跳转到中断处理程序。


## 练习2：补充完成基于FIFO的页面替换算法

简要实现过程：
- 在`swap_fifo.c`中，`_fifo_map_swappable`中，只要将新来的页面插入链表的末尾，`_fifo_swap_out_victim`中，只要将链表第一个元素取出并删除即可
- 在`vmm.c`的`do_pgfault`中，如果PTE中的内容为空，并且swap已经初始化好，就可以从swap中换入该页帧，具体是先调用`swap_in`从swap中读入该页，然后调用`page_insert`将该页插入页表中，最后调用`swap_map_swappable`并设置该页的`pra_vaddr`来建立该页和swap之间的映射关系

### 问题：如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap\_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。

能。在每次访问的时候设置相应的页的使用位和修改位，在`map_swappable`时将页插入链表，在`swap_out_victim`时扫描链表即可，注意当前扫描的位置指针可以用一个全局变量来保存。


