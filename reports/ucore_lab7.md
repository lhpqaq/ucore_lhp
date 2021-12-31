## lab7 实验报告
201907040101  刘昊鹏   

- [练习0](#练习0)
- [练习1](#练习1)  
- [练习2](#练习2)   
- [扩展练习1](#扩展练习1) 
- [扩展练习2](#扩展练习2)  
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
#### 哲学家就餐问题


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
## 练习2

## 扩展练习

## 总结
### 知识点
### 思考