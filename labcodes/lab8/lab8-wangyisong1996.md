# Lab 8 实验报告

## 练习0：填写已有实验

我已将 Lab 1 ~ Lab 7 中的代码填入本实验中，并确保能编译通过。

## 练习1: 完成读文件操作的实现（需要编码）

### 请在实验报告中给出设计实现“UNIX的PIPE机制”的概要设计方案，鼓励给出详细设计方案

PIPE机制为一种进程的间接通信机制，分为匿名管道和命名管道两种。具体来说：
- 匿名管道在创建的时候，会产生两个文件描述符，一个用于读，一个用于写，这样使用管道的两个进程就能一个向管道写数据，一个从管道读数据
- 命名管道在创建的时候要指定名称，这样不同进程之间可以之间通过该名称来对管道进行读写

在内核中，管道都被抽象成文件。不论是匿名管道还是命名管道，对于用户态程序，在创建之后都是文件的形式，可以用文件操作的函数来进行读写。
管道的实现方法跟常规文件不同。具体来说，管道有个有限大小的缓冲区，写入的时候如果缓冲区已满，则写入失败，读出的时候直接从缓冲区读。
这个缓冲区是先进先出（FIFO）的，可以用一个循环队列来实现。

当用户向系统申请创建一个管道时，只要从内存中分配一定的大小，将其初始化为一个先进先出缓冲区即可。
当用户向管道写入或读出时，只要操作相应的缓冲区，注意为保证操作的原子性，要对缓冲区加锁。
当用户需要删除一个管道时，只要将相应的缓冲区内存释放，这时另一方若尝试读写管道，会得到一个 Broken Pipe 错误。

## 练习2: 完成基于文件系统的执行程序机制的实现（需要编码）

### 请在实验报告中给出设计实现基于“UNIX的硬链接和软链接机制”的概要设计方案，鼓励给出详细设计方案

UNIX的硬链接机制和软连接机制为文件系统的特性，具体如下：
- 硬链接：一个硬链接指向一个具体的文件（磁盘上的 inode），这样一来，该文件（inode）就可以有多个硬链接（包括该文件本身）指向它，这些称作这个文件的别名，要删除所有别名才能删掉该文件
- 软连接：一个软连接指向一个文件名（或者文件夹名），可以将其理解成一个快捷方式，删除软连接并不会影响该链接指向的文件

实现方法如下：
- 硬链接：一个硬链接在文件系统中算作一个文件，但是不给它分配 inode，而是直接指向其链接的文件的 inode。需要对文件系统的每个 inode 进行引用计数，这样就能实现当且仅当删光一个文件的所有别名时才会删掉该文件本身。
- 软链接：一个软链接也是一个文件，但是是一个“快捷方式”，需要分配一个 inode 并储存其链接到的文件或目录名。在引用一个软链接的时候解析该 inode ，如果其引用的地址还是一个软链接就继续解析。如果解析的层数过多就停止并返回一个错误。

## 与参考答案的区别

- 在 sfs_io_nolock 中，我的实现与参考答案差别比较大，但是语义基本上是一样的。
- 在 do_fork 和 load_icode 中也有很大的差别，除了语义一致而在实现上的差别以外，主要在于，参考答案的实现中，对每种可能出错的情况都进行了处理，而我的实现中并不是所有的情况都进行了处理

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

本实验中重要的知识点为 sfs 中的读文件过程，以及应用程序启动的过程（load_icode），
这在OS原理课中都有直接的对应，区别在于本实验更注重代码的实现和细节。

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

原理课中提到了I/O子系统的各种原理和细节，包括磁盘调度和磁盘缓存，但是实验中没有涉及到。


## 附：实验感想

本次实验是我在OS课程中花费时间最多的一次实验。
一个原因是这次实验的内容很多，原理部分要理解的很多，要实现的代码也很多，特别是要实现整一个 load_icode （100多行）。
实现的过程中我遇到了不少困难和坑，比如说如何把 argc 和 argv 传给用户态的程序，最终我都解决了。
在实现完之后，我很快通过了所有的Testcase，但是始终没能在 sh 中执行所有可执行文件。经过一些调试和跟踪发现似乎是系统调用的时候一些寄存器被意外清零了，但没能解决，如果将来有空一定会继续尝试。
