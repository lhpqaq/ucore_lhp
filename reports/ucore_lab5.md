# ucore_lab5
201907040101  刘昊鹏  

[参考](https://github.com/AngelKitty/review_the_national_post-graduate_entrance_examination/blob/master/books_and_notes/professional_courses/operating_system/sources/ucore_os_lab/docs/lab_report/lab5/lab5%20%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8A.md)

- [练习0](#练习0)
- [练习1](#练习1)  
- [练习2](#练习2)   
- [练习3](#练习3) 
- [总结](#总结)
    - [知识点](#知识点)
    - [思考](#思考)

## 练习0
>__填写已有实验__  
>>本实验依赖实验1/2/3。请把你做的实验1/2/3的代码填入本实验中代码中有“LAB1”,“LAB2”,“LAB3”的注释相应部分。     

使用`WinMerge`完成。

## 练习1
>__分配并初始化一个进程控制块（需要编码）__   
>>alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。     

<a name="pcb"><a/>

### 进程管理
进程管理信息用struct proc_struct表示，在kern/process/proc.h中定义如下：
```C
struct proc_struct {
    enum proc_state state; // Process state
    int pid; // Process ID
    int runs; // the running times of Proces
    uintptr_t kstack; // Process kernel stack
    volatile bool need_resched; // need to be rescheduled to release CPU?
    struct proc_struct *parent; // the parent process
    struct mm_struct *mm; // Process's memory management field
    struct context context; // Switch here to run process
    struct trapframe *tf; // Trap frame for current interrupt
    uintptr_t cr3; // the base addr of Page Directroy Table(PDT)
    uint32_t flags; // Process flag
    char name[PROC_NAME_LEN + 1]; // Process name
    list_entry_t list_link; // Process link list
    list_entry_t hash_link; // Process hash list
};
```

__对其中一些成员进行解释：__  
- state：进程所处的状态。  
在ucore中进程的状态用以下四种状态表示：生命周期通常有6种情况：进程创建、进程执行、进程等待、进程抢占、进程唤醒、进程结束。
```C
// process's state in his life cycle
enum proc_state {
    PROC_UNINIT = 0,  // 未初始化   uninitialized
    PROC_SLEEPING,    // 等待       sleeping
    PROC_RUNNABLE,    // 就绪或者运行 runnable(maybe running)
    PROC_ZOMBIE,      // 僵死，等待父进程回收 almost dead, and wait parent proc to reclaim his resource
};
```

- kstack：进程内核栈的位置  
每个线程都有一个内核栈，并且位于内核地址空间的不同位置。对于内核线程，该栈就是运行时的程序使用的栈；而对于普通进程，该栈是发生特权级改变的时候使保存被打断的硬件信息用的栈。  
uCore在创建进程时分配了 2 个连续的物理页作为内核栈的空间。这个栈很小，kstack记录了分配给该进程/线程的内核栈的位置。  
其作用如下：
    - 当内核准备从一个进程切换到另一个的时候，需要根据kstack的值正确的设置好 tss，以便在进程切换以后再发生中断时能够使用正确的栈。  
    - 内核栈位于内核地址空间，并且是不共享的（每个线程都拥有自己的内核栈），因此不受到mm的管理，当进程退出的时候，内核能够根据kstack的值快速定位栈的位置并进行回收。  

- parent：用户进程的父进程  
指向创建它的进程。
在所有进程中，只有内核创建的第一个内核线程idleproc没有父进程。内核根据这个父子关系建立一个树形结构，用于维护一些特殊的操作，例如确定某个进程是否可以对另外一个进程进行某种操作等等。  

<a name="context"></a>  
- context：进程的上下文，用于进程切换  
其结构体如下，成员为
```C
struct context { 
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```
其成员为8个寄存器的值，保存了当前进程在执行上下文切换时的寄存器的状态，这里面没有eax寄存器，原因是进程切换通过函数实现，eax在函数调用时的作用是保存返回值，eax的值可以在栈上找到，所以可以不保存。在切换时，执行的函数是switch_to：
<a name="switch"></a>  
```py
switch_to:                      # switch_to(from, to)
    # save from's registers
    movl 4(%esp), %eax          #保存from的首地址
    popl 0(%eax)                #将返回值保存到context的eip
    movl %esp, 4(%eax)          #保存esp的值到context的esp
    movl %ebx, 8(%eax)          #保存ebx的值到context的ebx
    movl %ecx, 12(%eax)         #保存ecx的值到context的ecx
    movl %edx, 16(%eax)         #保存edx的值到context的edx
    movl %esi, 20(%eax)         #保存esi的值到context的esi
    movl %edi, 24(%eax)         #保存edi的值到context的edi
    movl %ebp, 28(%eax)         #保存ebp的值到context的ebp

    # restore to's registers
    movl 4(%esp), %eax          #保存to的首地址到eax
    movl 28(%eax), %ebp         #保存context的ebp到ebp寄存器
    movl 24(%eax), %edi         #保存context的ebp到ebp寄存器
    movl 20(%eax), %esi         #保存context的esi到esi寄存器
    movl 16(%eax), %edx         #保存context的edx到edx寄存器
    movl 12(%eax), %ecx         #保存context的ecx到ecx寄存器
    movl 8(%eax), %ebx          #保存context的ebx到ebx寄存器
    movl 4(%eax), %esp          #保存context的esp到esp寄存器
    pushl 0(%eax)               #将context的eip压入栈中
    ret
```

- mm：内存管理的信息  
mm数据结构是用来实现用户空间的虚存管理的，但是内核线程没有用户空间，它执行的只是内核中的一小段代码（通常是一小段函数），所以它没有mm 结构，也就是NULL。mm中有一项pgdir记录的进程使用的一级页表的物理地址。由于mm=NULL，所以使用proc_struct数据结构中的cr3成员变量代替mm中的pgdir。  

- cr3保存页表的物理地址  
在进程切换的时候方便直接使用cr3实现页表切换，避免每次都根据mm来计算cr3。
当某个进程是一个普通用户态进程的时候，PCB中的 cr3就是mm中页表（pgdir）的物理地址；而当它是内核线程的时候，cr3 等于boot_cr3。而boot_cr3指向了uCore启动时建立好的饿内核虚拟空间的页目录表首地址。  

<a name="tf"></a>  
- tf：中断帧的指针  
指向内核栈的某个位置：当进程从用户空间跳到内核空间时，中断帧记录了进程在被中断前的状态。当内核需要跳回用户空间时，需要调整中断帧以恢复让进程继续执行的各寄存器值。除此之外，uCore内核允许嵌套中断。因此为了保证嵌套中断发生时tf 总是能够指向当前的trapframe，uCore在内核栈上维护了tf的链。
查看其结构体：
```C
struct trapframe {
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```

练习1要求填写alloc_proc函数，该函数作用是分配一个proc_struct并且初始化所有的成员。  
根据以上的解释，代码如下：
```C
// alloc_proc - alloc a proc_struct and init all fields of proc_struct
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if(proc != NULL) {
        proc->state = PROC_UNINIT; //进程为初始化状态
        proc->pid = -1;            //进程PID为-1
        proc->runs = 0;            //初始化时间片
        proc->kstack = 0;          //内核栈地址
        proc->need_resched = 0;    //不需要调度
        proc->parent = NULL;       //父进程为空
        proc->mm = NULL;           //虚拟内存为空
        memset(&(proc->context), 0, sizeof(struct context)); //初始化上下文
        proc->tf = NULL;           //中断帧指针为空
        proc->cr3 = boot_cr3;      //页目录为内核页目录表的基址
        proc->flags = 0;           //标志位为0
        memset(proc->name, 0, PROC_NAME_LEN);//进程名为0
    }
    return proc;
}
```
### 思考题
- 请说明proc_struct中`struct context context`和`struct trapframe *tf`成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）  
    - context储存进程当前状态，用于进程切换中上下文的保存与恢复。
    - tf记录进程从用户空间跳转到内核空间，即中断前的状态。
    - 详细描述跳转到<a href="#context">context</a>与<a href="#tf">tf</a>。  

## 练习2
>__为新创建的内核线程分配资源（需要编码）__  
>>创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：
>>- 调用alloc_proc，首先获得一块用户信息块。
>>- 为进程分配一个内核栈。
>>- 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
>>- 复制原进程上下文到新进程
>>- 将新进程添加到进程列表
>>- 唤醒新进程
>>- 返回新进程号

我门可以根据上述的步骤，和注释中提供的函数信息，比较容易的补充完整代码：
```C
int do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) 
{
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) 
    {
        goto fork_out;
    }
    ret = -E_NO_MEM;
    //LAB4:EXERCISE2 YOUR CODE
    //调用alloc_proc，首先获得一块用户信息块。
    if ((proc = alloc_proc()) == NULL)
        goto fork_out;
    //设置子进程的父进程
    proc->parent = current;
    //为进程分配一个内核栈。
    if (setup_kstack(proc) != 0)
        goto bad_fork_cleanup_proc;
    //复制原进程的内存管理信息到新进程
    if (copy_mm(clone_flags, proc) != 0)
        goto bad_fork_cleanup_kstack;
    //复制原进程上下文到新进程
    copy_thread(proc, stack, tf);

    //将新进程添加到进程列表
    bool lock;//类似锁，与锁不同指是否允许中断，即接下来不允许中断切换进程
    local_intr_save(lock);
    {
        proc->pid = get_pid();
        hash_proc(proc);
        list_add(&proc_list, &(proc->list_link));
        nr_process ++;
    }
    local_intr_restore(lock);
    //唤醒新进程
    wakeup_proc(proc);
    //返回新进程号
    ret = proc->pid;

fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

### 思考题
__请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。__    
可以做到！  
将新进程添加到进程列表前，通过get_pid()获得了新fork的线程的id。查看get_pid()的实现：
```C
// get_pid - alloc a unique pid for process
// 可以看到注释里就已经说明分配一个唯一的id
static int
get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    //维护了两个静态变量,初始化为MAX_PID
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    //如果last_pid<next_safe,则直接返回last_pid
    //如果不满足，则遍历列表，找到一个满足的范围。
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }
    }
    return last_pid;
}
```
总结一下就是函数维护了两个静态成员last_pid和next_safe，如果`(last_pid,next_safe)`区间里有值，则直接返回last_pid为唯一的pid，last_pid+1为下次准备。如果不是，就需要遍历proc_list，重新对last_pid和next_safe进行设置，为下一次的get_pid调用打下基础。同时要满足不能大于MAX_PID。 

<a name="ex3"><a/>  
## 练习3
>__阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。（无编码工作）__  
>>请在实验报告中简要说明你对proc_run函数的分析。  

在执行proc_run之前，ucore需要先知道需要切换的进程，即传给proc_run的参数，current指向需要被调度的进程，而切换到的进程由中断和schedule()找到。
```C
void schedule(void) {
    bool intr_flag; //用于禁止中断
    list_entry_t *le, *last;
    struct proc_struct *next = NULL;//下一进程
    local_intr_save(intr_flag); //禁止中断
    {
        current->need_resched = 0; //设置当前进程不需要调度

      //last是否是idle进程(第一个创建的进程),如果是，则从表头开始搜索，否则获取下一链表
        last = (current == idleproc) ? &proc_list : &(current->list_link);
        le = last; 
        do { //循环找到可以调度的进程
            if ((le = list_next(le)) != &proc_list) {
                next = le2proc(le, list_link);//获取下一进程
                if (next->state == PROC_RUNNABLE) {
                    break; //找到可以调度的进程，break
                }
            }
        } while (le != last); //循环条件
        if (next == NULL || next->state != PROC_RUNNABLE) {
            next = idleproc; //未找到可以调度的进程
        }
        next->runs ++; //运行次数加一
        if (next != current) {
            proc_run(next);//调用proc_run函数
        }
    }
    local_intr_restore(intr_flag);
}
```
schedule找到可以调用的进程后调用proc_run执行。  
根据代码的注释信息。proc_run是让传递来的参数代表的进程proc运行在cpu上，见注释：
```C
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {//需要切换的进程是不是当前正在运行的进程
        bool intr_flag; //用于禁止中断
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);//禁止中断
        {
            current = proc; //将当前进程换为要切换到的进程
            //设置任务状态段 tss 中的特权级0下的esp0指针为 next 内核线程 的内核栈的栈顶
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);//修改当前的 cr3寄存器成需要运行线程（进程）的页目录表
            switch_to(&(prev->context), &(next->context));//进行上下文的切换和保存
        }
        local_intr_restore(intr_flag);//恢复中断
    }
}
```
proc_run用到函数如下：
```C
/* *
 * load_esp0 - change the ESP0 in default task state segment,
 * so that we can use different kernel stack when we trap frame
 * user to kernel.
 * */
//设置任务状态段 tss 中的特权级0下的esp0指针
void
load_esp0(uintptr_t esp0) {
    ts.ts_esp0 = esp0;
}
```
```C
//修改cr3寄存器
static inline void
lcr3(uintptr_t cr3) {
    asm volatile ("mov %0, %%cr3" :: "r" (cr3) : "memory");
}
```
<a href="#switch">switch to</a>  

### 并回答如下问题：
1. 在本实验的执行过程中，创建且运行了几个内核线程？  
    两个内核线程，分别是idleproc和initproc。
    idle_proc为第0个内核线程，在完成新的内核线程的创建以及各种初始化工作之后，进入循环，用于调度其他进程或线程；调用了以下函数：
    ```C
    void cpu_idle(void) {
    while (1)
        if (current->need_resched)
            schedule();
    }
    ```
    init_proc被创建用于打印"Hello World"的线程。本次实验的内核线程，只用来打印字符串。

2. 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?请说明理由    
    两个语句的作用是关闭中断和恢复中断，使得在这个语句块内的内容不会被中断打断，成为一个原子操作；比如说在proc_run函数中，将current指向了要切换到的线程，但是此时还没有真正将控制权转移过去，如果在这个时候出现中断打断这些操作，就会出现 current 中保存的并不是正在运行的线程的中断控制块，从而出现错误；
    其实现方式，我通过手动DFS不断搜索调用的函数，终于找到了能说明问题的地方：
    ```C
    static inline void
    cli(void) {
        asm volatile ("cli" ::: "memory");
    }
    ```
    cli(Clear Interrupt)为中断标志置0指令使IF=0，也就是关闭中断。通过sti(Set Interrupt)中断标志置1指令使IF=1，恢复中断。

## 总结

### 知识点


#### 内核线程
内核线程是一种只运行在内核地址空间的线程。所有的内核线程共享内核地址空间，所以也共享同一份内核页表。**这也是为什么叫内核线程，而不叫*内核进程*的原因。** 

#### 进程控制块
进程控制块是操作系统管理控制进程运行所用的信息集合。操作系统用PCB来描述进程的基本情况以及运行变化的过程。
PCB是进程存在的唯一标志 ，每个进程都在操作系统中有一个对应的PCB。
进程控制块可以通过某个数据结构组织起来（例如链表）。同一状态进程的PCB连接成一个链表，多个状态对应多个不同的链表。各状态的进程形成不同的链表：就绪联链表，阻塞链表等等。  
ucore维持的进程控制块<a href="#pcb">如上<a/>。

### 进程切换
见<a href="#ex3">练习三<a/>。

### 思考
本次实验进入了进程的部分，虽然内容较少，还没牵涉到调度算法等方面的内容，但是与lab3相比属于全新的内容，又要入门一下，入门是最难的部分。下一个实验同样是关于进程的，应该入门时不那么困难。