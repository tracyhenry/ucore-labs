### [练习0] 填写已有实验

- 已完成

### [练习1] 分配并初始化一个进程控制块

- 有三个成员变量需要特殊注意：
  - pid = -1 (代表不存在的进程）
  - cr3 = boot_cr3 (内核的页表起始地址）
  - state = UNINIT (未初始化的进程）

- 请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？
  - context用来存放用于进程切换的上下文，保存各种寄存器
  - tf则是中断帧，用来保存进程从用户空间跳到内核空间在中断前的状态


### [练习2] 为新创建的内核线程分配资源

- 基本实现流程是按照注释里的流程，需要注意的一点是为当前进程分配pid的操作，需要在hash_proc之前，因为hash_proc需要利用进程的id进行hash操作。

- 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。
  - 可以。kern/process/proc.c中的get_pid函数，为每一个新fork的进程分配了一个unique的id。

### [练习3] 阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。
- 执行流程：
  - proc_run用于进程之间的切换，他首先利用load_esp0函数将tss中特权态0下的栈顶指针esp0为当前线程的栈顶
  - 其次用lcr3指令切换页表
  - 再用switch_to切换上下文

- 在本实验的执行过程中，创建且运行了几个内核线程？
  - 两个，一个idle_proc，一个init_proc
  
- 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?请说明理由

  - 用local_intr_save() ... local_intr_restore() 括起来的代码中是不允许任何中断进入的。这样可以保证在切换进程的时候屏蔽了中断，保证操作的原子性。