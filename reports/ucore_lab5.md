# ucore_lab5
201907040101  刘昊鹏  

- [练习0](#练习0)
- [练习1](#练习1)  
- [练习2](#练习2)   
- [练习3](#练习3) 
- [扩展练习](#扩展练习) 
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
新增的内容是对proc_struct新增的几个成员进行初始化：
```C
    int exit_code;                              // exit code (be sent to parent proc)
    uint32_t wait_state;                        // waiting state
    struct proc_struct *cptr, *yptr, *optr;     // relations between processes
```
设置用户进程访问系统调用中断的权限，在idt_init()添加:
```c
    SETGATE(idt[T_SYSCALL], 0, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
```
表示用户态可以进行系统调用中断。  
设置每100个时间片进行一次进程调度，即lab1中的打印ticks。
```C
    case IRQ_OFFSET + IRQ_TIMER:
        ticks++;
        if(ticks % TICK_NUM == 0){
            assert(current != NULL);
            current->need_resched = 1;
        }
```
设置进程的相关链接，实验set_links函数：
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

check_boot_pgdir(void)
```C
    tlb_invalidate(boot_pgdir, 0x100);
    tlb_invalidate(boot_pgdir, 0x100+PGSIZE);
```
## 练习1
### 加载应用程序并执行（需要编码）  
需函数作用是要补充的load_icode函数作用是加载并解析一个处于内存中的ELF执行文件格式的应用程序，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc_struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。

<a name = "exec1" ></a>

首先介绍do_execve函数函数，用来完成用户进程的创建工作，其工作流程如下：
- 首先为加载新的执行码做好用户态内存空间清空准备。如果mm不为NULL，则设置页表为内核空间页表，且进一步判断mm的引用计数减1后是否为0，如果为0，则表明没有进程再需要此进程所占用的内存空间，为此将根据mm中的记录，释放进程所占用户空间内存和进程页表本身所占空间。最后把当前进程的mm内存管理指针为空。由于此处的initproc是内核线程，所以mm为NULL，整个处理都不会做。  
- 接下来的一步是调用load_icode函数加载应用程序执行码到当前进程的新创建的用户态虚拟空间中。涉及到读ELF格式的文件，申请内存空间，建立用户态虚存空间，加载应用程序执行码等。  

load_icode函数说明为：
```C
/* load_icode - load the content of binary program(ELF format) as the new content of current process
 * @binary:  the memory addr of the content of binary program
 * @size:  the size of the content of binary program
 */
```
该函数共有六个部分，已经提供的1-5部分是一系列对用户内存空间的初始化，需要补充的第六部分就是伪造中断返回现场，使得系统调用返回之后可以正确跳转到需要运行的程序入口。
- 调用mm_create函数来申请进程的内存管理数据结构mm所需内存空间，并对mm进行初始化；
```C
    //(1) create a new mm for current process
    if ((mm = mm_create()) == NULL) {
        goto bad_mm;
    }
```
- 申请一个页目录表所需的一个页大小的内存空间，并把描述ucore内核虚空间映射的内核页表的内容拷贝到此新目录表中，最后让mm->pgdir指向此页目录表，这就是进程新的页目录表了，且能够正确映射内核虚空间；
```C
    //(2) create a new PDT, and mm->pgdir= kernel virtual addr of PDT
    if (setup_pgdir(mm) != 0) {
        goto bad_pgdir_cleanup_mm;
    }
```
- 根据应用程序执行码的起始位置来解析此ELF格式的执行程序；
```C
    //(3) copy TEXT/DATA section, build BSS parts in binary to memory space of process
    struct Page *page;
    //(3.1) get the file header of the bianry program (ELF format)
    struct elfhdr *elf = (struct elfhdr *)binary;
    //(3.2) get the entry of the program section headers of the bianry program (ELF format)
    struct proghdr *ph = (struct proghdr *)(binary + elf->e_phoff);
    //(3.3) This program is valid?
    if (elf->e_magic != ELF_MAGIC) {
        ret = -E_INVAL_ELF;
        goto bad_elf_cleanup_pgdir;
    }
    uint32_t vm_flags, perm;
    struct proghdr *ph_end = ph + elf->e_phnum;
    for (; ph < ph_end; ph ++) {
    //(3.4) find every program section headers
        if (ph->p_type != ELF_PT_LOAD) {
            continue ;
        }
        if (ph->p_filesz > ph->p_memsz) {
            ret = -E_INVAL_ELF;
            goto bad_cleanup_mmap;
        }
        if (ph->p_filesz == 0) {
            continue ;
        }
```
- 调用mm_map函数根据ELF格式的执行程序说明的各个段（代码段、数据段、BSS段等）的起始位置和大小建立对应的vma结构，并把vma插入到mm结构中，从而表明了用户进程的合法用户态虚拟地址空间，调用根据执行程序各个段的大小分配物理内存空间，并根据执行程序各个段的起始位置确定虚拟地址，并在页表中建立好物理地址和虚拟地址的映射关系，然后把执行程序各个段的内容拷贝到相应的内核虚拟地址中，至此应用程序执行码和数据已经根据编译时设定地址放置到虚拟内存中了；
```C      
    //(3.5) call mm_map fun to setup the new vma ( ph->p_va, ph->p_memsz)
        vm_flags = 0, perm = PTE_U;
        if (ph->p_flags & ELF_PF_X) vm_flags |= VM_EXEC;
        if (ph->p_flags & ELF_PF_W) vm_flags |= VM_WRITE;
        if (ph->p_flags & ELF_PF_R) vm_flags |= VM_READ;
        if (vm_flags & VM_WRITE) perm |= PTE_W;
        if ((ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)) != 0) {
            goto bad_cleanup_mmap;
        }
        unsigned char *from = binary + ph->p_offset;
        size_t off, size;
        uintptr_t start = ph->p_va, end, la = ROUNDDOWN(start, PGSIZE);

        ret = -E_NO_MEM;

     //(3.6) alloc memory, and  copy the contents of every program section (from, from+end) to process's memory (la, la+end)
        end = ph->p_va + ph->p_filesz;
     //(3.6.1) copy TEXT/DATA section of bianry program
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            memcpy(page2kva(page) + off, from, size);
            start += size, from += size;
        }

      //(3.6.2) build BSS section of binary program
        end = ph->p_va + ph->p_memsz;
        if (start < la) {
            /* ph->p_memsz == ph->p_filesz */
            if (start == end) {
                continue ;
            }
            off = start + PGSIZE - la, size = PGSIZE - off;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
            assert((end < la && start == end) || (end >= la && start == la));
        }
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
        }
```
- 给用户进程设置用户栈，为此调用mm_mmap函数建立用户栈的vma结构，明确用户栈的位置在用户虚空间的顶端，并分配一定数量的物理内存且建立好栈的虚地址<-->物理地址映射关系；
```C
    //(4) build user stack memory
    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;
    }
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);
```
- mm->pgdir赋值到cr3寄存器中，即更新了用户进程的虚拟内存空间。
```C
    //(5) set current process's mm, sr3, and set CR3 reg = physical addr of Page Directory
    mm_count_inc(mm);
    current->mm = mm;
    current->cr3 = PADDR(mm->pgdir);
    lcr3(PADDR(mm->pgdir));
```
- 先清空进程的中断帧，再重新设置进程的中断帧，使得在执行中断返回指令“iret”后，能够让CPU转到用户态特权级，并回到用户态内存空间，使用用户态的代码段、数据段和堆栈，且能够跳转到用户进程的第一条指令执行，并确保在用户态能够响应中断；
```C
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
    tf->tf_cs = USER_CS;   //将段寄存器初始化为用户态的代码段；
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;//数据段、附加段、堆栈段
    tf->tf_esp = USTACKTOP; //esp指向先前的步骤中创建的用户栈的栈顶；
    tf->tf_eip = elf->e_entry;//eip指向ELF可执行文件加载到内存之后的入口处；
    tf->tf_eflags = FL_IF; //FL_IF为中断打开状态
    ret = 0;
```

### 思考题
__描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。__  

用户态进程调用了exec系统调用，从而转入到了系统调用的处理例程，在经过了正常的中断处理例程之后，最终控制权转移到了syscall.c中的syscall函数，然后根据系统调用号转移给了sys_exec函数，调用do_execve函数来完成指定应用程序的加载；  
do_execve和其调用的load_icode进行了用户态进程的初始化，设置当前进程的中断帧为用户态进程的信息，尤其eip已经被修改成了应用程序的入口处，然后中断返回的时候会将堆栈切换到用户的栈，并且完成特权级的切换，并且跳转到要求的应用程序的入口处，然后执行应用程序第一条指令。


## 练习2
### 父进程复制自己的内存空间给子进程
创建子进程的函数do_fork在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过copy_range函数（位于kern/mm/pmm.c中）实现的，请补充copy_range的实现，确保能够正确执行。  
copy_range()经过do_fork()->copy_mm()->dup_mmap()->copy_range()的调用过程。其作用和参数描述如下：
```C
/* copy_range - copy content of memory (start, end) of one process A to another process B
 * @to:    the addr of process B's Page Directory
 * @from:  the addr of process A's Page Directory
 * @share: flags to indicate to dup OR share. We just use dup method, so it didn't be used.
 *
 * CALL GRAPH: copy_mm-->dup_mmap-->copy_range
 */
```

```C
int copy_range(pde_t *to, pde_t *from, uintptr_t start, uintptr_t end, bool share) {
    assert(start % PGSIZE == 0 && end % PGSIZE == 0);
    assert(USER_ACCESS(start, end));
    // copy content by page unit.
    do {
        //call get_pte to find process A's pte according to the addr start
        pte_t *ptep = get_pte(from, start, 0), *nptep;//找到进程的A的PTE的起始地址
        if (ptep == NULL) {
            start = ROUNDDOWN(start + PTSIZE, PTSIZE);
            continue ;
        }
        //call get_pte to find process B's pte according to the addr start. If pte is NULL, just alloc a PT
        if (*ptep & PTE_P) {
            if ((nptep = get_pte(to, start, 1)) == NULL) {
                return -E_NO_MEM;
            }
        uint32_t perm = (*ptep & PTE_USER);
        //get page from ptep
        struct Page *page = pte2page(*ptep);
        // alloc a page for process B
        struct Page *npage=alloc_page();
        assert(page!=NULL);
        assert(npage!=NULL);
        int ret=0;
        /* LAB5:EXERCISE2 YOUR CODE
         * replicate content of page to npage, build the map of phy addr of nage with the linear addr start
         *
         * Some Useful MACROs and DEFINEs, you can use them in below implementation.
         * MACROs or Functions:
         *    page2kva(struct Page *page): return the kernel vritual addr of memory which page managed (SEE pmm.h)
         *    page_insert: build the map of phy addr of an Page with the linear addr la
         *    memcpy: typical memory copy function
         *
         * (1) find src_kvaddr: the kernel virtual address of page
         * (2) find dst_kvaddr: the kernel virtual address of npage
         * (3) memory copy from src_kvaddr to dst_kvaddr, size is PGSIZE
         * (4) build the map of phy addr of  nage with the linear addr start
         */
        void * kva_src = page2kva(page); //找到父进程需要复制的物理页在内核地址空间中的虚拟地址，这是由于这个函数执行的时候使用的时内核的地址空间
        void * kva_dst = page2kva(npage); //找到子进程需要被填充的物理页的内核虚拟地址
        memcpy(kva_dst, kva_src, PGSIZE); //将父进程的物理页的内容复制到子进程中去
        ret = page_insert(to, npage, start, perm); // 建立子进程的物理页与虚拟页的映射关系
        assert(ret == 0);
        }
        start += PGSIZE;
    } while (start != 0 && start < end);
    return 0;
}
```
### 思考题
简要说明如何设计实现”Copy on Write 机制“，给出概要设计，鼓励给出详细设计。
>Copy-on-write（简称COW）的基本概念是指如果有多个使用者对一个资源A（比如内存块）进行读操作，则每个使用者只需获得一个指向同一个资源A的指针，就可以使用该资源了。若某使用者需要对这个资源A进行写操作，系统会对该资源进行拷贝操作，从而使得该“写操作”使用者获得一个该资源A的“私有”拷贝—资源B，可对资源B进行写操作。该“写操作”使用者对资源B的改变对于其他的使用者而言是不可见的，因为其他使用者看到的还是资源A。  

其具体实现见扩展练习。  

### 验证
<div align="center">
<img src="https://i.bmp.ovh/imgs/2021/12/30925b006957fca3.png" alt="check" width="500"/>
</div>

## 练习3
### 阅读分析源代码，理解进程执行fork/exec/wait/exit的实现，以及系统调用的实现
请在实验报告中简要说明你对 fork/exec/wait/exit函数的分析。
#### fork
fork调用sys_fork：
```c
static int
sys_fork(uint32_t arg[]) {
    struct trapframe *tf = current->tf;
    uintptr_t stack = tf->tf_esp;
    return do_fork(0, stack, tf);
}
```
sys_fork主要是对do_fork的调用，do_fork在lab4中实现了大部分，lab5里进行了略微的修改，主要是以下的第五步：  
- 调用alloc_proc，首先获得一块用户信息块。
- 为进程分配一个内核栈。
- 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
- 复制原进程上下文到新进程
- 设置进程间的关系
- 唤醒新进程
- 返回新进程号

#### exec
```C
static int
sys_exec(uint32_t arg[]) {
    const char *name = (const char *)arg[0];
    size_t len = (size_t)arg[1];
    unsigned char *binary = (unsigned char *)arg[2];
    size_t size = (size_t)arg[3];
    return do_execve(name, len, binary, size);
}
```
主要是调用do_execve函数，其作用在上面有<a href="#exec1">描述</a>，其实现的代码如下：
```C
int
do_execve(const char *name, size_t len, unsigned char *binary, size_t size) {
    struct mm_struct *mm = current->mm;
    if (!user_mem_check(mm, (uintptr_t)name, len, 0))
        return -E_INVAL;
    if (len > PROC_NAME_LEN)
        len = PROC_NAME_LEN;
    char local_name[PROC_NAME_LEN + 1];
    memset(local_name, 0, sizeof(local_name));
    memcpy(local_name, name, len);
    //释放内存
    if (mm != NULL) {
        lcr3(boot_cr3);
        if (mm_count_dec(mm) == 0) {
            exit_mmap(mm);
            //删除该内存管理所对应的PDT
            put_pgdir(mm);
            mm_destroy(mm);
        }
        current->mm = NULL;
    }
    //加载可执行文件代码，重设mm_struct，以及重置trapframe
    int ret;
    if ((ret = load_icode(binary, size)) != 0)
        goto execve_exit;
    //设置进程名称
    set_proc_name(current, local_name);
    return 0;
execve_exit:
    do_exit(ret);
    panic("already exit: %e.\n", ret);
}
```
#### wait

do_wait程序会使某个进程一直等待，直到（特定）子进程退出后，该进程才会回收该子进程的资源并函数返回。该函数的具体操作如下：

- 检查当前进程所分配的内存区域是否存在异常。
- 查找特定/所有子进程中是否存在某个等待父进程回收的子进程（PROC_ZOMBIE）。
    - 如果有，则回收该进程并函数返回。
    - 如果没有，则设置当前进程状态为PROC_SLEEPING并执行schedule调度其他进程运行。当该进程的某个子进程结束运行后，当前进程会被唤醒，并在do_wait函数中回收子进程的PCB内存资源。


```C
// do_wait - wait one OR any children with PROC_ZOMBIE state, and free memory space of kernel stack
//         - proc struct of this child.
// NOTE: only after do_wait function, all resources of the child proces are free.
int do_wait(int pid, int *code_store) {
    struct mm_struct *mm = current->mm;
    if (code_store != NULL) {
        if (!user_mem_check(mm, (uintptr_t)code_store, sizeof(int), 1)) {
            return -E_INVAL;
        }
    }

    struct proc_struct *proc;
    bool intr_flag, haskid;
repeat:
    haskid = 0;
    if (pid != 0) {
        proc = find_proc(pid);
        if (proc != NULL && proc->parent == current) {
            haskid = 1;
            if (proc->state == PROC_ZOMBIE) {
                goto found;
            }
        }
    }
    else {
        proc = current->cptr;
        for (; proc != NULL; proc = proc->optr) {
            haskid = 1;
            if (proc->state == PROC_ZOMBIE) {
                goto found;
            }
        }
    }
    if (haskid) {
        current->state = PROC_SLEEPING;
        current->wait_state = WT_CHILD;
        schedule();
        if (current->flags & PF_EXITING) {
            do_exit(-E_KILLED);
        }
        goto repeat;
    }
    return -E_BAD_PROC;

found:
    if (proc == idleproc || proc == initproc) {
        panic("wait idleproc or initproc.\n");
    }
    if (code_store != NULL) {
        *code_store = proc->exit_code;
    }
    local_intr_save(intr_flag);
    {
        unhash_proc(proc);
        remove_links(proc);
    }
    local_intr_restore(intr_flag);
    put_kstack(proc);
    kfree(proc);
    return 0;
}

```
#### exit

从一个进程回收资源并退出，其具体操作如下：

- 回收所有内存（除了PCB，该结构只能由父进程回收）
- 设置当前的进程状态为PROC_ZOMBIE
- 设置当前进程的退出值current->exit_code。
- 如果有父进程，则唤醒父进程，使其准备回收该进程的PCB。
- 如果当前进程存在子进程，则设置所有子进程的父进程为initproc。这样倘若这些子进程进入结束状态，则initproc可以代为回收资源。
- 执行进程调度。一旦调度到当前进程的父进程，则可以马上回收该终止进程的PCB。

```C
int do_exit(int error_code) {
    if (current == idleproc)
        panic("idleproc exit.\n");
    if (current == initproc)
        panic("initproc exit.\n");
    // 释放所有内存空间
    struct mm_struct *mm = current->mm;
    if (mm != NULL) {
        lcr3(boot_cr3);
        if (mm_count_dec(mm) == 0) {
            exit_mmap(mm);
            put_pgdir(mm);
            mm_destroy(mm);
        }
        current->mm = NULL;
    }
    // 设置当前进程状态
    current->state = PROC_ZOMBIE;
    current->exit_code = error_code;
    // 请求父进程回收剩余资源
    bool intr_flag;
    struct proc_struct *proc;
    local_intr_save(intr_flag);
    {
        proc = current->parent;
        // 唤醒父进程。父进程准备回收该进程的PCB资源。
        if (proc->wait_state == WT_CHILD)
            wakeup_proc(proc);
        // 如果当前进程存在子进程，则设置所有子进程的父进程为init。
        while (current->cptr != NULL) {
            proc = current->cptr;
            current->cptr = proc->optr;

            proc->yptr = NULL;
            if ((proc->optr = initproc->cptr) != NULL)
                initproc->cptr->yptr = proc;
            proc->parent = initproc;
            initproc->cptr = proc;
            if (proc->state == PROC_ZOMBIE) {
                if (initproc->wait_state == WT_CHILD)
                    wakeup_proc(initproc);
            }
        }
    }
    local_intr_restore(intr_flag);
    // 该进程的生命周期即将结束，调度其他进程执行。
    schedule();
    panic("do_exit will not return!! %d.\n", current->pid);
}
```
并回答如下问题：
1. 请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？  

    fork会修改其子进程的状态为PROC_RUNNABLE，而当前进程状态不变。  
    exec不修改当前进程的状态，但会替换内存空间里所有的数据与代码。  
    wait会先检测是否存在子进程。如果存在进入PROC_ZOMBIE的子进程，则回收该进程并函数返回。但若存在尚处于PROC_RUNNABLE的子进程，则当前进程会进入PROC_SLEEPING状态，并等待子进程唤醒。  
    exit会将当前进程状态设置为PROC_ZOMBIE，并唤醒父进程，使其处于PROC_RUNNABLE的状态，之后主动让出CPU。  

2. 请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。（字符方式画即可）

<div align="center">
<img src="https://s3.bmp.ovh/imgs/2021/12/0177bdf8a52dc9c5.png" alt="check" width="500"/>
</div>

## 扩展练习
### 实现 Copy on Write（COW）机制

基本思想见[练习2](#练习2)  
在copy_range中添加：
```C
        if(share)
        {
            // 物理页面共享，并设置两个PTE上的标志位为只读
            cprintf("test cow\n");//用于测试
            page_insert(from, page, start, perm & ~PTE_W);
            ret = page_insert(to, page, start, perm & ~PTE_W);
        }
        else
```
把`struct Page *npage=alloc_page();`和练习2的代码移到else里。  
当程序尝试修改只读的内存页面的时候，会触发Page Fault中断，这时内核需要重新分配页面、拷贝页面内容、建立映射关系,在错误代码中P=1，W/R=1。因此，当错误代码为3：
在do_pgfault中添加错误码为3时的处理：
```C
    //vmm.c 395
    if (*ptep == 0) { // if the phy addr isn't exist, then alloc a page & map the phy addr with logical addr

    }
    else if (error_code & 3 == 3) {
        cprintf("test cow write\n");//用于测试
        struct Page *page = pte2page(*ptep);
        struct Page *npage = pgdir_alloc_page(mm->pgdir, addr, perm);
        uintptr_t src_kvaddr = page2kva(page);
        uintptr_t dst_kvaddr = page2kva(npage);
        memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
    }
    else {
```
COW机制的开关在dup_mmap中，即bool变量：
```C
//vmm.c  line204
int dup_mmap(struct mm_struct *to, struct mm_struct *from) {
    ...
        bool share = 1;//1时使用cow机制
    ...
}
```

测试：
<div align="center">
<img src="https://i.bmp.ovh/imgs/2021/12/adbb1256e59afd6e.png" alt="cow" width="500"/>
</div>

打印出了测试信息。
## 总结

### 知识点

- ELF可执行文件的格式
- 用户进程的创建和管理
- 进程调度
- 统调用框架的实现机制
- <a href="#练习3">如何实现系统调用sys_fork/sys_exec/sys_exit/sys_wait来进行进程管理</a>
- <a href="#cow">COW</a>

### 思考
这周任务比较多，时间紧，实验做的有些赶，这次实验也是比较难的一次，不光要对概念的理解，还要有对过程的理解。有些内容理解不够透彻，下次实验时会回头再详细看一遍。