# ucore_lab5
201907040101  刘昊鹏  

- [练习0](#练习0)
- [练习1](#练习1)  
- [练习2](#练习2)   
- [练习3](#练习3) 
- [总结](#总结)
    - [知识点](#知识点)
    - [思考](#思考)

## 练习0
### 填写已有实验
本实验依赖实验1/2/3/4。请把你做的实验1/2/3/4的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”的注释相应部分。注意：为了能够正确执行lab5的测试应用程序，可能需对已完成的实验1/2/3/4的代码进行进一步改进。     
需要修改的地方有：
```C
static struct proc_struct *alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
        ...
        proc->wait_state = 0; //初始化进程等待状态  
        proc->cptr = proc->optr = proc->yptr = NULL;//进程相关指针初始化
    }
    return proc;
}
```
设置用户进程访问系统调用中断的权限，在idt_init()添加:
```c
    SETGATE(idt[T_SYSCALL], 0, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
```
设置没100个时间片进行一次进程调度，即lab1中的打印ticks。
```C
    case IRQ_OFFSET + IRQ_TIMER:
        ticks++;
        if(ticks % TICK_NUM == 0){
            assert(current != NULL);
            current->need_resched = 1;
        }
```

设置进程的相关链接：
```C
int do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    ...
    //设置子进程的父进程
    proc->parent = current;
    assert(current->wait_state == 0);//确保当前进程正在等待
    ...
    local_intr_save(lock);
    {
        proc->pid = get_pid();//获取当前进程 PID
        hash_proc(proc);
        set_links(proc);//设置进程的相关链接
    }
    ...
}
```
## 练习1
### 加载应用程序并执行（需要编码）  
需函数作用是要补充的load_icode函数作用是加载并解析一个处于内存中的ELF执行文件格式的应用程序，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc_struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。  
其函数说明为：
```C
/* load_icode - load the content of binary program(ELF format) as the new content of current process
 * @binary:  the memory addr of the content of binary program
 * @size:  the size of the content of binary program
 */
```
该函数共有六个部分，已经提供的1-5部分是一系列对用户内存空间的初始化，需要补充的第六部分就是伪造中断返回现场，使得系统调用返回之后可以正确跳转到需要运行的程序入口。

```C
static int load_icode(unsigned char *binary, size_t size) {
    if (current->mm != NULL) { //当前进程的内存为空
        panic("load_icode: current->mm must be empty.\n");
    }

    int ret = -E_NO_MEM;
    struct mm_struct *mm;
    //(1) create a new mm for current process
    if ((mm = mm_create()) == NULL) { //分配内存
        goto bad_mm;
    }
    //(2) create a new PDT, and mm->pgdir= kernel virtual addr of PDT
    if (setup_pgdir(mm) != 0) {  //申请一个页目录表所需的空间
        goto bad_pgdir_cleanup_mm; //申请失败
    }
    //(3) copy TEXT/DATA section, build BSS parts in binary to memory space of process
    ...
    //(4) build user stack memory
    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;
    }
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);
    
    //(5) set current process's mm, sr3, and set CR3 reg = physical addr of Page Directory
    mm_count_inc(mm);
    current->mm = mm;
    current->cr3 = PADDR(mm->pgdir);
    lcr3(PADDR(mm->pgdir));

    //(6) setup trapframe for user environment
    struct trapframe *tf = current->tf;
    memset(tf, 0, sizeof(struct trapframe));
    /* LAB5:EXERCISE1 YOUR CODE
     * should set tf_cs,tf_ds,tf_es,tf_ss,tf_esp,tf_eip,tf_eflags
     * NOTICE: If we set trapframe correctly, then the user level process can return to USER MODE from kernel. So
     *          tf_cs should be USER_CS segment (see memlayout.h)
     *          tf_ds=tf_es=tf_ss should be USER_DS segment
     *          tf_esp should be the top addr of user stack (USTACKTOP)
     *          tf_eip should be the entry point of this binary program (elf->e_entry)
     *          tf_eflags should be set to enable computer to produce Interrupt
     */
    tf->tf_cs = USER_CS;   //将段寄存器初始化为用户态的代码段、数据段、堆栈段；
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP; //esp指向先前的步骤中创建的用户栈的栈顶；
    tf->tf_eip = elf->e_entry;//eip指向ELF可执行文件加载到内存之后的入口处；
    tf->tf_eflags = FL_IF; //FL_IF为中断打开状态
    ret = 0;
out:
    return ret;
bad_cleanup_mmap:
    exit_mmap(mm);
bad_elf_cleanup_pgdir:
    put_pgdir(mm);
bad_pgdir_cleanup_mm:
    mm_destroy(mm);
bad_mm:
    goto out;
}
```
## 练习2
### 父进程复制自己的内存空间给子进程


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