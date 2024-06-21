# 操作系统实验作业报告

## 一、适配测评平台
本组选取xv6-k210作为基础代码，对测评平台进行适配。适配过程主要包括以下步骤：

#### 1.1 修改`Makefile`
平台会在根目录中运行`kernel-qemu`，而xv6-k210原有的`Makefile`产生的可执行文件是`T/kernel`。因此我们需要将其重命名并移动到正确的位置。

此外，还需删除`make fs`对应的所有指令下的`sudo`，这是由于镜像默认以root权限运行指令。

#### 1.2 修改`init.c`
本部分按照答疑课上陈同学分享的适配方案进行。对于`init.c`的修改主要包括以下两部分：

1. 自动调用测例。这需要在`tests[]`中填入正确的测例名称，并且在`init.c`主函数所产生的子进程中依次执行它们，即
```c
pid = fork();
if (pid < 0) {
    printf("init: fork failed\n");
    exit(1);
}
if (pid == 0) {
    exec(tests[i], argv);   // * 此处调用了tests的第i项对应的测例
    printf("init: exec sh failed\n");
    exit(1);
}
```
2. 关闭内核。测评平台要求在所有测例运行结束后，操作系统要自行关机。这需要在`init.c`末尾添加`__syscall0(SYS_shutdown)`，其中`__syscall0(...)`定义为`riscv`的`syscall`函数。由于操作系统将要结束运行，此处需要使用汇编级别的代码来关机。

#### 1.3 生成`initcode.h`片段以自动调用测例
我们使用`make fs`指令编译`init.c`，产生`_init`可执行文件。通过如下指令可以将其转化为16进制文本的`initcode.h`存放在`kernel/include/`目录下。
```c
riscv64-linux-gnu-objcopy -S -O binary xv6-user/_init xxx
od -v -t x1 -An xxx | sed -E 's/ (.{2})/0x\1,/g' > kernel/include/initcode.h
```

在xv6-k210中，内核代码是在`proc.c`中加载16进制代码片段的，我们只需将对应片段进行替换即可。
```c
uchar initcode[] = {
    #include "include/initcode.h"
};
```

在此过程中，我们发现不同的虚拟机配置生成的`initcode.h`是不同的，只有一名小组成员生成的`initcode.h`能得到非零分数；我们推测这与本地编译器或链接器的配置版本有关。不过，由于`initcode.h`在声明系统调用处理函数后不再变化，另一名成员只需复制正确版本的`initcode.h`直接使用即可。

#### 1.4 修改调用号，支持系统调用
在测评平台上运行代码，必须使得内核的调用号与测评平台一致。通过查阅github.com/oscomp/testsuits-for-oskernel/blob/main/oscomp_syscalls.md，可以得知正确的系统调用号。

接下来，系统调用框架的基本修改逻辑可以概括如下：
1. `kernel/include/sysnum.h`以宏定义的方式声明了系统调用号，需要修改至与上述文档一致。
2. `kernel/syscall.c`中将处理系统调用的内核函数(例如`sys_fork()`)作为外部函数声明，并且用`syscalls[]`维护了系统调用号到具体的相应函数入口的映射。这一映射需要修改至与文档一致。
```c
static uint64 (*syscalls[])(void) = {
    [SYS_fork]      sys_fork,
    ...
};
```
3. 在对应的内核程序段中，具体实现那些被`syscall.c`所声明的内核函数，后文具体展开。

#### 1.5 其他
为了避免由于`sh.c`中的`cmd()`不返回所产生的Werror影响本地编译，我们在它之前加入了如下代码段：
```c
__attribute__((noreturn));
```

## 二、测评结果
在测评平台中，本组完整通过的测试点包括以下19组：
```
test_dup 	     	
test_clone 	 	
test_read 	 	
test_exit 	 	
test_fork 	 	
test_getdents 	 	
test_mkdir 	 	
test_chdir 	 	
test_pipe 	
test_getpid 	 	
test_wait 	
test_open 	 	
test_gettimeofday 	 	
test_getppid 	
test_openat 	
test_yield 	 	
test_getcwd 	 	
test_write 	 	
test_close 	 	
```
其中，`sys_fork()`等系统调用，无需对xv6-k210的基础代码做任何修改即可通过评测。接下来，将对所有需要在xv6-k210基础工程做修改才能通过的系统调用做介绍，分为三部分：
1. 与进程相关的系统调用，在`sysproc.c`中实现，见第三节。
2. 与文件相关的系统调用，在`sysfile.c`中实现，见第四节。
3. 其他系统调用，为了方便起见也在`sysfile.c`中实现，见第五节。

## 三、与进程相关的系统调用

#### 进程控制块(PCB)的定义
在实现进程相关的系统调用前，我们首先阅读并理解xv6-k210基础工程中PCB的定义和使用方法。其定义为结构体`struct proc`，成员如下：
1. `struct spinlock lock`: 自旋锁，用于保护进程状态信息，确保在多线程或多处理器环境中对进程状态的访问是同步的。
2. `enum procstate state`: 进程的状态，如运行、就绪、等待、终止等。
3. `struct proc *parent`: 指向父进程的指针。
4. `void *chan`: 如果这个字段非0，则那么进程可能在某个通道(channel)上睡眠(等待某个事件)。
5. `int killed`: 标识进程是否已经被杀死。
6. `int xstate`: 进程的退出状态。当进程结束时，这个状态会被返回给父进程。
7. `int pid`: 进程标识符。
8. `uint64 kstack`: 内核栈的虚拟地址。
9. `uint64 sz`: 进程内存的大小(以字节为单位)。
10. `pagetable_t pagetable`: 用户页表。用于将进程的虚拟地址转换为物理地址。
11. `pagetable_t kpagetable`: 内核页表。用于内核模式中的地址转换。
12. `struct trapframe *trapframe`: 指向陷阱帧(trapframe)的指针。陷阱帧保存了当进程从用户模式切换到内核模式时的CPU信息。
13. `struct context context`: 进程的上下文。当进程被切换出时，它的上下文(寄存器值、PC等)保存在这里，以便稍后回复执行。
14. `struct file *ofile`: 进程打开的文件数组。`NOFILE`是常量，定义了进程可以同时打开的最大文件数。
15. `struct dirent *cwd`: 当前工作目录的指针。
16. `char name[16]`: 进程名，主要用于调试。
17. `int tmask`: 跟踪掩码(trace mask)。用来控制哪些系统调用或事件应该被跟踪或记录。

#### sys_getppid()

**功能：** 获取父进程id。输入系统调用id，返回父进程id。

**步骤：** 实现思路与xv6-k210现成的`sys_getppid()`类似，直接调用 `myproc()`函数获取PCB，然后获取它的父进程的pid。

注意到，当前进程控制块的获取可以通过调用`myproc()`自动完成，从而在系统调用中无需指定参数。

#### sys_sched_yield()

**功能：** 让出处理器，即让当前线程放弃CPU。进程管理系统会把该线程放到其对应优先级的CPU静态队列的尾端，然后一个新的线程会占用CPU。

**步骤：** 调用xv6-k210现成的`yield()`函数。该函数获取进程锁，将当前进程的状态设置为`RUNNABLE`，调用调度器决定下一个运行的进程或线程，最后释放锁。

#### sys_clone()

**功能：** 创建一个子进程。其中，输入参数指定了创建子进程的标志`flags`，子进程的栈，本地存储描述符`TLS`等。

**步骤：** `sys_clone()`调用`proc.c`中的`clone()`函数，后者分配一个新的子进程，并且对其控制信息和资源进行初始化。初始化的内容包括：

1. 用户地址空间：使用`uvmcopy()`函数复制父进程的内存空间。如果发生内存错误，此时应注意释放子进程。
```c
uvmcopy(p->pagetable, np->pagetable, np->kpagetable, p->sz);
```
2. 打开文件信息：复制父进程所有打开的文件描述符，并调用`filedup()`函数在子进程中增加引用计数。然后用`edup()`函数复制当前工作目录。
```c
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = edup(p->cwd);
```
3. PCB中控制信息：设置新进程内存大小与父进程相同，设置新进程的父进程为当前进程，复制`tmask`(追踪系统调用)和`trapframe`(其中保存了CPU寄存器和状态，便于在中断或系统调用后恢复执行)，设置子进程的a0寄存器为0(这样在子进程中调用`clone()`时会正确地返回0)，设置栈指针。
```c
  np->sz = p->sz;
  np->parent = p;
  // copy mask from parent.
  np->tmask = p->tmask;
  // Copy trapframe
  *(np->trapframe) = *(p->trapframe);
  // The return value should be set to 0 to the children.
  np->trapframe->a0 = 0;
  // Build the child's stack
  if (st) np->trapframe->sp = st;
```
4. 在以上设置完成后，最终将新进程的状态标志位`RUNNABLE`，并且返回新的pid。

**调试：** 调试过程中修改`clone()`数次导致整个提交分数清零，其中要注意PCB控制信息设置的顺序，以及不能遗漏任何一项，例如`tmask`。

#### sys_wait4()
**功能：** 等待进程改变状态，子进程的pid可以指定。注意的是，当参数为`-1`时，等待的是所有子进程，这种情况必须单独处理。


**步骤：** 调用proc.c中的`wait4()`函数，后者的实现主要参考了xv6-210中现成的`wait()`函数。与`wait()`最主要的区别是，`wait4()`可以指定所要等待的子进程的pid，由此出发对`wait()`进行相应的修改。基本步骤与修改如下：

1. 参数定义。`wait4()`函数接受两个参数：`pid`(要等待的子进程的id)和`addr`(指向整数的指针，用于存储子进程的退出状态)。
2. 特殊判定。若参数中的`pid`为-1，那么我们要实现的`wait4()`的功能其实退化成了`wait()`，调用现成的`wait()`即可。
3. 获取当前进程p的锁，遍历所有进程，查找是否有子进程且其id与给定的pid匹配。因此，这里与`wait()`函数不同，判定条件应改为`(np->parent == p && np->pid==pid)`，添加与目标pid的匹配。如果找到匹配的子进程且其状态为ZOMBIE(已终止但尚未被父进程回收)，则获取该子进程的锁。
4. 处理`ZOMBIE`状态的子进程。这里的实现与`wait()`相同。如果`addr`不为0，则将子进程的`xstate`复制到`addr`所指向的位置。释放子进程和当前进程的锁，用`freeproc`函数释放子进程的资源。从`addr`中提取退出状态并打印相关信息。最后返回子进程的`pid`。
5. 最后，与`wait()`相同，若遍历完所有进程后没有找到匹配的子进程，或者当前进程被杀死，则释放当前进程的锁并返回-1。如果没有找到匹配的子进程，则当前进程会调用`sleep()`函数进入睡眠状态，等待某个子进程变为`ZOMBIE`状态。

## 四、与文件相关的系统调用

#### sys_dup3()
**功能：** 复制文件描述符，并指定新的文件描述符。输入中`old`是被复制的文件描述符，`new`是新的文件描述符。

**步骤：** 实现原理与xv6-k210现成的`dup()`相同。重要的区别是，`dup3()`指定了目标的文件描述符，所以应将其也指向打开文件。
```c
    p->ofile[fd]=f;
```
这一操作引起的问题是，`ofile[fd]`可能对应一个已经存在且打开的文件。对于这种情况，则在执行上述操作与调用filedup函数来增加文件的引用次数前，要先关闭这一打开的文件。
```c
    if(p->ofile[fd])
        fileclose(p->ofile[fd]);
```

#### sys_openat()
**功能：** 打开或创建一个文件。与xv6-k210现成的函数`sys_open()`相比，主要的区别是包含要打开或创建的文件名，可以是绝对路径或相对路径。因此，需要在现有的`sys_open()`基础上增加对路径的解析。文档中对参数`filename`的描述是“要打开或创建的文件名。若为绝对路径，则忽略fd。若为相对路径，且fd是`AT_FDCWD`，则filename是相对于当前工作目录来说的。若为相对路径，且fd是一个文件描述符，则filename是相对于fd所指向的目录来说的。”

**步骤：** 在`sys_open()`的基础上增加解析参数`filename`的步骤。
1. 目标fd是`AT_FDCWD`，即当前工作目录，并且path是一个相对路径。这时，我们需要将相对路径转化成绝对路径。路径转化的原件部分借鉴了往年的项目思路：递归逐层地进行，使用while循环，从`cwd`开始，从其当前目录逐层回溯到根目录，将每个目录的名称`filename`添加到`tmp_pathname`中（需要添加`/`；该变量递归地增长为最终的绝对路径名），最后加上原先的相对路径。
2. 目标fd不是`AT_FDCWD`。这时，我们通过`ep`成员获取fd所指向的目录，之后与第二步类似，沿父目录依次添加，加上相对路径得到绝对路径。
3. 在解析了`filename`后，其余的实现与`sys_open()`基本相同。一处不同是，此时检查的是参数`flags`中的`O_CREATE`标志。若需要创建文件，则调用`create()`函数在指定路径下创建一个新文件。如果不需要创建文件，则调用`ename()`函数根据路径来查找已存在的文件或目录条目(ep)。对ep加锁来防止并发访问时数据不一致。如果用户尝试将目录作为普通文件打开(即没有`O_DIRECTORY`)，并且没有指定只读模式(`O_RDONLY`)，则释放与ep相关的锁和资源，并返回错误-1。之后的操作与`sys_open()`完全相同。

#### sys_mkdirat()
**功能：** 创建目录；与`sys_openat()`类似，是有指定dirfd和path的版本：如果是相对路径，则它是相对于dirfd目录而言的。如果`path`是相对路径，且dirfd的值为AT_FDCWD，则它是相对于当前路径而言的。如果path是绝对路径，则dirfd被忽略。

**步骤：** 用create函数在path指定的路径上创建一个新目录。
```c
ep = create(path, T_DIR, 0);
```

### sys_getdents64()
**功能：** 获取目录的条目。若成功则返回读取的字节数，到目录结尾则返回0；失败返回-1。其中，参数`buf`指向了缓冲区。

**步骤：** 我们注意到，`sys_getdents64()`要实现的功能和xv6-k210现有的`sys_readdir()`非常类似，且都要求返回读到的字节数。不同的是，`sys_getdents64()`支持一个buffer。既然现成的`sys_readdir()`是通过调用`file.c`中的`dirnext(...)`函数实现的，我们类似地实现一个与之`dirnext64(...)`函数，其中支持有buffer的读目录项操作。

在此之前，我们定义了`dirent_64`的结构体，与xv6-k210原有的`dirent`相对应，方便进行查找。其结构为
```c
    struct dirent64 {
    uint64        dirent_inode; //索引节点号
    uint64         dirent_off;  //到下一项的offset偏移量
    unsigned short  dirent_recordlen; //当前记录的长度
    unsigned char   dirent_type; //记录类型
    char            dirent_name[FAT32_MAX_FILENAME + 1];
};
```

在`dirnext64()`函数中：
1. 用`while`循环遍历目录中的每个条目。设置`de64.dirent_off`为当前文件偏移量f->off，调用`enext()`函数读取下一个目录项，并存在结构体中。
2. 特殊判定，对空条目和文件结束进行处理。如果`enext`返回0，表示这是一个空条目，继续循环读取下一个条目。如果`enext`返回-1，表示已经到达文件末尾，函数返回0，表示没有更多的条目可以读取。
3. 填充`dirent64`结构体(转换为`dirent64`结构体并填充字段)，计算其大小。用安全的字符串复制函数`memmove()`完成文件名复制。
4. 检查缓冲区，若溢出则break循环。
5. 将dirent64结构体复制到用户空间的地址addr，更新用户空间缓冲区的地址和剩余大小，返回读出的字节数。

## 五、其它系统调用

为了在编译时方便，我们在`sysfile.c`中还实现了以下系统调用，尽管其与文件系统并无直接关系。

#### sys_gettimeofday()
**功能：**获取当前的时间。输入的`timespec`结构体指针用于装载所获得时间值。成功则返回0。


**步骤：**用`r_time()`函数获取时间。注意，这里使用的`r_time()`是SBI接口所提供的函数，其向硬件读取时间并返回，无需参数。
```c
    uint64 tmp_ticks = r_time();
```
然后根据系统时钟相关的常数计算计算秒(sec)和微秒(usec)的值。将内核获取的时间信息(`tod`)复制到用户空间地址(`tmp_val`)。


