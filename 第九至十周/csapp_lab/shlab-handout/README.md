### 异常处理

系统为每类异常都分配了一个唯一的非负异常号[^1]，其中一些号码是处理器的设计者分配的，另一些号码是操作系统内核的设计者分配的。前者包括**被零除，缺页，内存访问违例，断点以及算术溢出**；后者包括**系统调用和来自外部的I/O设备信号**

[^1]: 在x86-64中共有255号异常，0~31号由Intel架构师定义的异常，32~255号由操作系统对应的异常

上述这些异常号被统一存储在一张异常表中，表中存储了不同异常号的异常处理程序跳转地址。

![image-20200731222742066](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200731222742066.png)

异常分为四类：

| 类别 |       原因       | 异步/同步[^2] |         返回行为         |
| :--: | :--------------: | :-----------: | :----------------------: |
| 中断 | 来自I/O设备信号  |     异步      |   总是返回到下一条指令   |
| 陷阱 |    有意的异常    |     同步      |   总是返回到下一条指令   |
| 故障 | 潜在可恢复的错误 |     同步      | 可能会返回到当前指令[^3] |
| 终止 |  不可恢复的错误  |     同步      |         不会返回         |

[^2]: 异步是由处理器外部的I/O设备中的事件产生，同步是执行一条指令的直接产物
[^3]: 根据故障是否能被修复，故障处理程序要么重新执行引起故障的程序，要么终止程序

个人对于陷阱的理解就是陷入内核态去执行特权指令，故此叫陷阱。

**并发流**：一个逻辑流的执行在时间上与另一个流重叠，流X和Y互相并发，当且仅当X在Y开始之后和Y结束之前开始，或者Y在X开始之后和X结束之前开始。

**上下文**：内核重新启动一个被抢占的进程所需的状态。

### 进程控制

+ 每一个进程都有一个唯一的正数（非零）进程ID（PID）

  fork()只被调用一次 ，却返回两次：一次是在调用进程（父进程）中；一次是在新创建的子进程中。在父进程中，fork返回子进程的PID；在子进程中，fork返回0。因为子进程的PID总是非零，所以返回值就成为程序在父进程还是子进程中执行的分辨量。

+ 通过waitpid(pid_t pid，int statusp ，int options)函数来等待子进程的回收

  pid>0，等待集合就是一个单个的子进程；pid=-1，等待集合就是父进程下的所有子进程

  通过设置options可以修改默认行为：

  - WNOHANG：等待集合中的任何子进程都还没有停止，那就立即返回，默认行为是挂起调用进程，直到有子进程终止
  - WUNTRACED：挂起调用进程的执行，直到等待集合中的一个进程变成已终止或者被停止，返回的PID为导致返回的已终止或被停止子进程的PID，默认行为是只返回已终止的子进程的PID
  - WCONTINUED：挂起调用进程的执行，直到等待集合中一个正在运行的进程终止或者等待集合中一个被停止的进程收到SIGCONT信号重新开始执行
  - 可以用或运算把选项组合起来，例如：WNOHANG|WUNTRACED：立即返回，如果等待集合中的子进程都没有被停止或终止则返回值为0；如果有一个终止或停止，则返回值为该子进程的PID

  如果调用进程没有子进程，返回-1，并设置errno为ECHILD；被一个信号中断，返回-1，设置errno为EINTR

### 信号

![image-20200801191543700](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200801191543700.png)

+ 一个发出而没有被接收的信号叫待处理信号，任何时刻，一种类型至多只有一个待处理信号。当一个进程有一个类型为k的待处理信号时，任何接下来发送到该进程的类型为k的信号都会被丢弃
+ 隐式阻塞机制：内核默认阻塞任何当前处理程序正在处理信号类型的待处理的信号
+ 显示阻塞机制：应用程序可以使用sigprocmask函数和其辅助函数，明确地阻塞和解除阻塞选定的信号

### Shell lab

该lab的任务是自行实现一个shell，代码在tsh.c里编写，主要实现几个函数

+ eval函数
+ builtin_cmd函数
+ do_bgfg函数
+ waitfg函数
+ 三个信号的操作函数

#### eval函数

```
如果用户在shell中输入的是内置命令(quit,jobs,bg or fg)
则立即执行该命令，否则就视作是文件的路径名。在此情况下
fork子进程并在该子进程的上下文中执行。
如果程序在前台运行，就得等待其终止后返回
```

总结一下，就是读取输入的命令行，fork子进程，执行。

其中写这个函数前需要知道的一些知识点：

+ 对于每个fork的子进程，执行setgpid(0, 0)，这样就会以子进程号单独开一个进程组，也可以方便的使用kill(-pid, SIGNAL)来把信号发到pid所在的整个进程组

+ 函数中的addjob函数就是对于进程进行一个排序，方便后续对进程的查找，删除等操作

#### builtin_cmd函数

```
如果用户键入了一个内置命令，则立即执行它。
四种内置命令，quit,jobs,bg,fg
```

此处就是判断你的命令到底是内置命令还是一个文件的路径名

#### do_bgfg函数

```
执行内置的bg和fg命令
```

主要是解析参数，判断是否会出现命令错误，然后找到所对应的进程用kill函数对它发出SIGCONT信号（如果该进程停止则继续进程）

#### waitfg函数

等到前台进程执行结束的函数。如果前台进程号变化，就说明结束了，就跳出循环，否则忙等待。

#### sigchld_handler函数

收到SIGCHLD信号，代表该子进程的结束。对于子进程结束，一共有三个原因，正常结束，收到SIGINT终止，收到SIGTSTP停止。

对于正常和终止的程序，delete。

对于停止的程序，修改state为ST(stop)

#### sigint_handler函数

给前台进程发送SIGINT信号

#### sigtstp_handler

和sigint_handler类似