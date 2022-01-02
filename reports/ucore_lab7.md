## lab7 实验报告
201907040101  刘昊鹏   

- [练习0](#练习0)
- [练习1](#练习1)  
- [练习2](#练习2)   
- [总结](#总结)
    - [知识点](#知识点)
    - [思考](#思考)

## 练习0
需要修改的地方位于trap.c
```C
static void trap_dispatch(struct trapframe *tf) {
    ++ticks;
    run_timer_list();
}
```

## 练习1
### 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

查看信号量的数据结构：
```C
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```
根据value成员的值判断是否满足条件：
```
value> 0，表示共享资源的空闲数
vlaue< 0，表示该信号量的等待队列里的进程数
value= 0，表示等待队列为空
```
wait_queue为使用信号量在等待中的线程。  
当多个线程可以进行互斥或同步合作时，某个线程会由于无法满足信号量设置的某条件而在某一位置停止，直到它接收到一个特定的信号（表明条件满足了）。为了发信号，需要使用一个称作信号量的特殊变量。为通过信号量s传送信号，信号量通过wait和signal操作来修改传送信号量。  

在ucore初始化过程，开始的执行流程都与实验六相同，直到执行到创建第二个内核线程init_main时，增加了check_sync函数的调用，check_sync函数是实验七的起始执行点，是实验七的总控函数。这个函数主要分为了两个部分，第一部分是实现基于信号量的哲学家问题，第二部分是实现基于管程的哲学家问题。
```C
void check_sync(void)
{
    int i;
    //check semaphore
    sem_init(&mutex, 1);
    for(i=0;i<N;i++) {            //N是哲学家的数量
        sem_init(&s[i], 0);       //初始化信号量
        int pid = kernel_thread(philosopher_using_semaphore, (void *)i, 0);//线程需要执行的函数名、哲学家编号、0表示共享内存
        //创建哲学家就餐问题的内核线程
        if (pid <= 0) {     //创建失败的报错
            panic("create No.%d philosopher_using_semaphore failed.\n");
        }
        philosopher_proc_sema[i] = find_proc(pid);
        set_proc_name(philosopher_proc_sema[i], "philosopher_sema_proc");
    }
 
    //check condition variable
    monitor_init(&mt, N);
    for(i=0;i<N;i++){
        state_condvar[i]=THINKING;
        int pid = kernel_thread(philosopher_using_condvar, (void *)i, 0);
        if (pid <= 0) {
            panic("create No.%d philosopher_using_condvar failed.\n");
        }
        philosopher_proc_condvar[i] = find_proc(pid);
        set_proc_name(philosopher_proc_condvar[i], "philosopher_condvar_proc");
    }
}
```
我们分析check_sync函数的第一部分，首先实现初始化了一个互斥信号量，然后创建了对应5个哲学家行为的5个信号量，并创建5个内核线程代表5个哲学家，每个内核线程完成了基于信号量的哲学家吃饭睡觉思考行为实现。
逐个分析其调用的函数：  
sem_init初始化一个信号量：
```C
void
sem_init(semaphore_t *sem, int value) {
    sem->value = value;//设置value值
    wait_queue_init(&(sem->wait_queue));//初始化该队列
}
```
kernel_thread创建一个内核线程，使其运行philosopher_using_semaphore，查看philosopher_using_semaphore：
```C
int philosopher_using_semaphore(void * arg) /* i：哲学家号码，从0到N-1 */
{
    int i, iter=0;
    i=(int)arg;
    cprintf("I am No.%d philosopher_sema\n",i);
    while(iter++<TIMES)
    { /* 无限循环 */
        cprintf("Iter %d, No.%d philosopher_sema is thinking\n",iter,i); /* 哲学家正在思考 */
        do_sleep(SLEEP_TIME);
        phi_take_forks_sema(i); 
        /* 需要两只叉子，或者阻塞 */
        cprintf("Iter %d, No.%d philosopher_sema is eating\n",iter,i); /* 进餐 */
        do_sleep(SLEEP_TIME);
        phi_put_forks_sema(i); 
        /* 把两把叉子同时放回桌子 */
    }
    cprintf("No.%d philosopher_sema quit\n",i);
    return 0;    
}
```
这个函数模拟哲学家思考和进餐的过程，do_sleep用于模拟持续思考和进餐的过程。进餐前需要拿起刀叉，进餐结束方向刀叉。先看phi_take_forks_sema(i)拿起刀叉：
```C
void phi_take_forks_sema(int i) /* i：哲学家号码从0到N-1 */
{
        down(&mutex); /* 进入临界区 */
        state_sema[i]=HUNGRY; /* 记录下哲学家i饥饿的事实 */
        phi_test_sema(i); /* 试图得到两只叉子 */
        up(&mutex); /* 离开临界区 */
        down(&s[i]); /* 如果得不到叉子就阻塞 */
}
```
接着看down：
```C
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);//关闭中断
    if (sem->value > 0) {
        // value值递减
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    // 如果在上一步中，值已经为0了，则将当前进程添加进等待队列中
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);
    schedule();// 进程调度
    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);// 从等待队列中删除当前进程
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```
down函数会递减当前信号量的value值。如果value在递减前为0，则将其加入至等待队列wait_queue中，并使当前线程立即放弃CPU资源，调度至其他线程。这是实现互斥的核心部分。  

在哲学家进入临界区后，试图拿起刀叉：
```C
void phi_test_sema(i) /* i：哲学家号码从0到N-1 */
{
    if(state_sema[i]==HUNGRY && state_sema[LEFT]!=EATING && state_sema[RIGHT]!=EATING)
    {
        state_sema[i]=EATING;
        up(&s[i]);
    }
}
```
当哲学家处饥饿状态且左右的哲学家没有使用刀叉，该哲学家可以进餐。然后使用up处理信号量
```C
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        //如果当前等待队列中没有线程等待，则value照常+1
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {  //否则如果当前等待队列中存在线程正在等待，则唤醒该线程并开始执行对应代码
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
```  
进餐结束后讲放下刀叉：
```C
void phi_put_forks_sema(int i) /* i：哲学家号码从0到N-1 */
{
        down(&mutex); /* 进入临界区 */
        state_sema[i]=THINKING; /* 哲学家进餐结束 */
        phi_test_sema(LEFT); /* 看一下左邻居现在是否能进餐 */
        phi_test_sema(RIGHT); /* 看一下右邻居现在是否能进餐 */
        up(&mutex); /* 离开临界区 */
}
```
然后判断左右邻居是否饥饿，可以进餐。

__请在实验报告中给出内核级信号量的设计描述，并说其大致执行流流程。__

- sem_init：对信号量进行初始化的函数，根据在原理课上学习到的内容，信号量包括了等待队列和一个整型数值变量，该函数只需要将该变量设置为指定的初始值，并且将等待队列初始化即可；
- up：表示释放了一个该信号量对应的资源，如果有等待在了这个信号量上的进程，则将其唤醒执行；结合函数的具体实现可以看到其采用了禁用中断的方式来保证操作的原子性，函数中操作的具体流程为：
查询等待队列是否为空，如果是空的话，给整型变量加 1；
如果等待队列非空，取出其中的一个进程唤醒；
- down：表示请求一个该信号量对应的资源，同样采用了禁用中断的方式来保证原子性，具体流程为：
查询整型变量来了解是否存在多余的可分配的资源，是的话取出资源（整型变量减 1），之后当前进程便可以正常进行；
如果没有可用的资源，整型变量不是正数，当前进程的资源需求得不到满足，因此将其状态改为 SLEEPING 态，然后将其挂到对应信号量的等待队列中，调用 schedule 函数来让出 CPU，在资源得到满足，重新被唤醒之后，将自身从等待队列上删除掉；


__请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。__   

内核为用户态进程/线程提供信号量机制时，需要设计多个应用程序接口，而用户态线程只能通过这些内核提供的接口来使用内核服务。类似实验5中跟进程有关的一些系统调用。   
- 相同点：
其核心的实现逻辑是一样的
- 不同点：
内核态的信号量机制可以直接调用内核的服务，而用户态的则需要通过内核提供的接口来访问内核态服务，这其中涉及到了用户态转内核态的相关机制。  
内核态的信号量存储于内核栈中；但用户态的信号量存储于用户栈中。

## 练习2
### 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题（需要编码）
__首先掌握管程机制，然后基于信号量实现完成条件变量实现，然后用管程机制实现哲学家就餐问题的解决方案（基于条件变量）。__  

首先看管程的结构体：
```C
typedef struct monitor{
    semaphore_t mutex; //用来互斥访问管程，只允许一个进程进入管程
    semaphore_t next;//用于条件同步（进程同步操作的信号量），发出 signal 操作的进程等条件为真之前进入睡眠
    int next_count;//睡眠的进程数量
    condvar_t *cv;//条件变量
} monitor_t;
```  
管程中的成员变量mutex是一个二值信号量，是实现每次只允许一个进程进入管程的关键元素，确保了互斥访问性质。  
管程中的条件变量cv通过执行wait_cv，会使得等待某个条件C为真的进程能够离开管程并睡眠，且让其他进程进入管程继续执行；而进入管程的某进程设置条件 C 为真并执行 signal_cv 时，能够让等待某个条件 C 为真的睡眠进程被唤醒，从而继续进入管程中执行。  

管程中的成员变量信号量 next 和整形变量 next_count 是配合进程对条件变量 cv 的操作而设置的，这是由于发出signal_cv 的进程 A 会唤醒睡眠进程 B，进程 B 执行会导致进程 A 睡眠，直到进程 B 离开管程，进程 A 才能继续执行，这个同步过程是通过信号量 next 完成的；  

而 next_count 表示了由于发出 singal_cv 而睡眠的进程个数。    

条件变量cv的数据结构如下：
```C
typedef struct condvar{ 
    semaphore_t sem;//用于条件同步 用于发出wait操作的进程等待条件为真之前进入睡眠
    int count;//在这个条件变量上的睡眠进程的个数
    monitor_t * owner;//此条件变量所属管程
} condvar_t;
```
条件变量的定义中也包含了一系列的成员变量，信号量 sem 用于让发出 wait_cv 操作的等待某个条件 C 为真的进程睡眠，而让发出 signal_cv 操作的进程通过这个 sem 来唤醒睡眠的进程。count 表示等在这个条件变量上的睡眠进程的个数。owner 表示此条件变量的宿主是哪个管程。  
其实本来条件变量中需要有等待队列的成员，以表示有多少线程因为当前条件得不到满足而等待，但这里，直接采用了信号量替代，因为信号量数据结构中也含有等待队列。  
在check_sync的第二部分，为条件编写实现哲学家就餐问题：
```C
monitor_init(&mt, N);   //初始化管程
for(i=0;i<N;i++){
    state_condvar[i]=THINKING;
    int pid = kernel_thread(philosopher_using_condvar, (void *)i, 0);
    if (pid <= 0) {
        panic("create No.%d philosopher_using_condvar failed.\n");
    }
    philosopher_proc_condvar[i] = find_proc(pid);
    set_proc_name(philosopher_proc_condvar[i], "philosopher_condvar_proc");
}
```
与练习1不同之处在于。创建的进程为philosopher_using_condvar：
```C
int philosopher_using_condvar(void * arg) { /* arg is the No. of philosopher 0~N-1*/
  
    int i, iter=0;
    i=(int)arg;
    cprintf("I am No.%d philosopher_condvar\n",i);
    while(iter++<TIMES)
    { /* iterate*/
        cprintf("Iter %d, No.%d philosopher_condvar is thinking\n",iter,i); /* thinking*/
        do_sleep(SLEEP_TIME);
        phi_take_forks_condvar(i); 
        /* need two forks, maybe blocked */
        cprintf("Iter %d, No.%d philosopher_condvar is eating\n",iter,i); /* eating*/
        do_sleep(SLEEP_TIME);
        phi_put_forks_condvar(i); 
        /* return two forks back*/
    }
    cprintf("No.%d philosopher_condvar quit\n",i);
    return 0;    
}
```
与练习1不同之处在phi_take_forks_condvar和phi_put_forks_condvar：
```C
void phi_take_forks_condvar(int i) {
     down(&(mtp->mutex));//进入临界区，加锁
//--------into routine in monitor--------------
     // LAB7 EXERCISE1: YOUR CODE
     // I am hungry
     // try to get fork
      state_condvar[i]=HUNGRY; // 饥饿状态，准备进食
      phi_test_condvar(i);  //测试哲学家是否能拿到刀叉，若不能拿，则阻塞自己，等其它进程唤醒
      if (state_condvar[i] != EATING) { //没拿到，需要等待，调用 wait 函数
          cprintf("phi_take_forks_condvar: %d didn't get fork and will wait\n",i);
          cond_wait(&mtp->cv[i]);
      }
//--------leave routine in monitor--------------
      if(mtp->next_count>0)
         up(&(mtp->next));
      else
         up(&(mtp->mutex));
}
```
mtp为全局变量管程。   
phi_test_condvar拿起刀叉，条件与练习1相同，进餐时将通过cond_signal唤醒在该条件变量下等待的进程  

```C
void phi_test_condvar (i) { 
    if(state_condvar[i]==HUNGRY&&state_condvar[LEFT]!=EATING
            &&state_condvar[RIGHT]!=EATING) {
        cprintf("phi_test_condvar: state_condvar[%d] will eating\n",i);
        state_condvar[i] = EATING ;
        cprintf("phi_test_condvar: signal self_cv[%d] \n",i);
        cond_signal(&mtp->cv[i]) ;
    }
}
```

cond_signal() 函数实现思路：
1. 判断条件变量的等待队列是否为空
2. 修改next变量上等待进程计数
3. 唤醒等待队列中的某一个进程
4. 把自己等待在 next 条件变量上
5. 当前进程被唤醒，恢复 next 上的等待进程计数  


```C
void cond_signal (condvar_t *cvp) {
   //LAB7 EXERCISE1: YOUR CODE
   cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);       //这是一个输出信息的语句，可以不管
   if(cvp->count>0) {
       cvp->owner->next_count ++;  //管程中睡眠的数量
       up(&(cvp->sem));            //唤醒在条件变量里睡眠的进程
       down(&(cvp->owner->next));  //将在管程中的进程睡眠
       cvp->owner->next_count --;
   }
   cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```
如果没有拿到刀叉，则睡眠，调用cond_wait  
cond_wait() 函数实现思路：
1. 修改等待在条件变量的等待队列上的进程计数
2. 释放锁
3. 将自己等待在条件变量上
4. 被唤醒，修正等待队列上的进程计数   

```C
void cond_wait (condvar_t *cvp) {
    //LAB7 EXERCISE1: YOUR CODE
    cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
    cvp->count++;                  //条件变量中睡眠的进程数量加 1
    if(cvp->owner->next_count > 0)
       up(&(cvp->owner->next));//如果当前有进程正在等待，且睡在宿主管程的信号量上，此时需要唤醒
    else
       up(&(cvp->owner->mutex));//如果没有进程睡眠，那么当前进程无法进入管程的原因就是互斥条件的限制。因此唤醒mutex互斥锁，代表现在互斥锁被占用。
    down(&(cvp->sem));//自己睡眠
    cvp->count --;
    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```
接下来是放下刀叉：
```C
void phi_put_forks_condvar(int i) {
     down(&(mtp->mutex));
//--------into routine in monitor--------------
     // LAB7 EXERCISE1: YOUR CODE
     // I ate over
     state_condvar[i]=THINKING; // 哲学家进餐结束
     // test left and right neighbors
     phi_test_condvar(LEFT); // 看一下左邻居现在是否能进餐
     phi_test_condvar(RIGHT); // 看一下右邻居现在是否能进餐
//--------leave routine in monitor--------------
     if(mtp->next_count>0)
        up(&(mtp->next));
     else
        up(&(mtp->mutex));
}
```
结果：  
<div align="center">
<img src="https://i.bmp.ovh/imgs/2022/01/e1161ccc36d2c16d.png" alt="VMMM" width="500"/>
</div>

  
## 总结
### 知识点
#### 信号量
信号量是一种同步互斥机制的实现，普遍存在于现在的各种操作系统内核里。相对于spinlock 的应用对象，信号量的应用对象是在临界区中运行的时间较长的进程。等待信号量的进程需要睡眠来减少占用 CPU 的开销。  
对于以下信号量设计：
```C
struct semaphore {
int count;
queueType queue;
};
void semWait(semaphore s)
{
s.count--;
if (s.count < 0) {
/* place this process in s.queue */;
/* block this process */;
}
}
void semSignal(semaphore s)
{
s.count++;
if (s.count<= 0) {
/* remove a process P from s.queue */;
/* place process P on ready list */;
}
}
```
当多个（>1）进程可以进行互斥或同步合作时，一个进程会由于无法满足信号量设置的某条件而在某一位置停止，直到它接收到一个特定的信号（表明条件满足了）。为了发信号，需要使用一个称作信号量的特殊变量。为通过信号量s传送信号，信号量的V操作采用进程可执行原语semSignal(s)；为通过信号量s接收信号，信号量的P操作采用进程可执行原语semWait(s)；如果相应的信号仍然没有发送，则进程被阻塞或睡眠，直到发送完为止。

#### 管程机制
一个管程定义了一个数据结构和能为并发进程所执行（在该数据结构上）的一组操作，这组操作能同步进程和改变管程中的数据。管程由四部分组成：  
- 管程内部的共享变量；
- 管程内部的条件变量；
- 管程内部并发执行的进程；
- 对局部于管程内部的共享数据设置初始值的语句。  
局限在管程中的数据结构，只能被局限在管程的操作过程所访问，任何管程之外的操作过程都不能访问它；另一方面，局限在管程中的操作过程也主要访问管程内的数据结构。由此可见，管程相当于一个隔离区，它把共享变量和对它进行操作的若干个过程围了起来，所有进程要访问临界资源时，都必须经过管程才能进入，而管程每次只允许一个进程进入管程，从而需要确保进程之间互斥。  
在单处理器情况下，将会导致所有其它进程都无法进入临界区使得该条件Cond为真，该管程的执行将会发生死锁。为此，可引入条件变量（Condition Variables，简称CV）。一个条件变量CV可理解为一个进程的等待队列，队列中的进程正等待某个条件Cond变为真。每个条件变量关联着一个条件，如果条件Cond不为真，则进程需要等待，如果条件Cond为真，则进程可以进一步在管程中执行。需要注意当一个进程等待一个条件变量CV（即等待Cond为真），该进程需要退出管程，这样才能让其它进程可以进入该管程执行，并进行相关操作，比如设置条件Cond为真，改变条件变量的状态，并唤醒等待在此条件变量CV上的进程。因此对条件变量CV有两种主要操作：  
- wait_cv： 被一个进程调用，以等待断言Pc被满足后该进程可恢复执行. 进程挂在该条件变量上等待时，不被认为是占用了管程。
- signal_cv：被一个进程调用，以指出断言Pc现在为真，从而可以唤醒等待断言Pc被满足的进程继续执行。  

##### 管程中函数的入口出口设计
为了让整个管程正常运行，还需在管程中的每个函数的入口和出口增加相关操作，即：
```C
function_in_monitor （…）
{
  sem.wait(monitor.mutex);
//-----------------------------
  the real body of function;
//-----------------------------
  if(monitor.next_count > 0)
     sem_signal(monitor.next);
  else
     sem_signal(monitor.mutex);
}
```
这样带来的作用有两个，（1）只有一个进程在执行管程中的函数。（2）避免由于执行了cond_signal函数而睡眠的进程无法被唤醒。对于第二点，如果进程A由于执行了cond_signal函数而睡眠（这会让monitor.next_count大于0，且执行sem_wait(monitor.next)），则其他进程在执行管程中的函数的出口，会判断monitor.next_count是否大于0，如果大于0，则执行sem_signal(monitor.next)，从而执行了cond_signal函数而睡眠的进程被唤醒。上诉措施将使得管程正常执行。

### 思考
本次实验考察同步与互斥，其中信号量和条件变量是在课堂上学过的内容，而对管程机制比较陌生。所以通过实验有学习了一个新概念。  