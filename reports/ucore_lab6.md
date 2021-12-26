## lab6 实验报告
201907040101  刘昊鹏   

- [练习0](#练习0)
- [练习1](#练习1)  
- [练习2](#练习2)   
- [扩展练习](#扩展练习) 
- [总结](#总结)
    - [知识点](#知识点)
    - [思考](#思考)


## 练习0
### 填写已有实验  
本实验依赖实验1/2/3/4/5。请把你做的实验2/3/4/5的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”“LAB5”的注释相应部分。并确保编译通过。注意：为了能够正确执行lab6的测试应用程序，可能需对已完成的实验1/2/3/4/5的代码进行进一步改进。    
LAB6的proc_struct结构体在LAB5的基础上新增了几个成员：
```C
    struct run_queue *rq;                   //运行队列中包含进程
    list_entry_t run_link;                  //进程在调度链表中的节点
    int time_slice;                         //进程剩余的时间片
    skew_heap_entry_t lab6_run_pool;        //进程在优先队列中的节点
    uint32_t lab6_stride;                   //进程的调度步进值
    uint32_t lab6_priority;                 //进程的调度优先级
```

因此在alloc_proc需要对这些新增的成员初始化：
```C
// alloc_proc - alloc a proc_struct and init all fields of proc_struct
static struct proc_struct *alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
        ...
        proc->rq = NULL;//初始化运行队列为空
        list_init(&(proc->run_link));
        proc->time_slice = 0;//初始化时间片
        proc->lab6_run_pool.left = proc->lab6_run_pool.right = proc->lab6_run_pool.parent = NULL;//初始化指针为空
        proc->lab6_stride = 0;//设置步长为0
        proc->lab6_priority = 0;//设置优先级为0
    }
    return proc;
}
```
以及时钟中断处理：
```C
    case IRQ_OFFSET + IRQ_TIMER:
        ticks ++;
        assert(current != NULL);
        sched_class_proc_tick(current);
        break;
```

## 练习1
### 使用 Round Robin 调度算法（不需要编码）
`Round Robin` 调度算法的调度思想是让所有 runnable 态的进程分时轮流使用 CPU 时间。`Round Robin` 调度器维护当前 runnable 进程的有序运行队列。当前进程的时间片用完之后，调度器将当前进程放置到运行队列的尾部，再从其头部取出进程进行调度。  

__首先考虑思考题1__  
请理解并分析sched_class中各个函数指针的用法，并结合Round Robin 调度算法描ucore的调度执行过程。  
sched_class结构如下：
```C
struct sched_class {
    //调度算法的名字
    const char *name;
    //初始化运行队列
    void (*init)(struct run_queue *rq);
    //将进程加入运行队列
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    //将进程移出运行队列
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    //选择下一个要执行的进程
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    //时间片处理例程
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
};
```
ucore当前使用的调度算法框架：
```C
struct sched_class default_sched_class = {
    .name = "RR_scheduler",  //调度算法名字
    .init = RR_init,        //初始化函数RR_init
    .enqueue = RR_enqueue,
    .dequeue = RR_dequeue,
    .pick_next = RR_pick_next,
    .proc_tick = RR_proc_tick,
};
```

`Round Robin` 调度算法的主要实现在 `default_sched.c` 之中，接下来逐个分析其中的函数。  

`RR_init`函数，用于对进程队列的初始化。  

```c
static void RR_init(struct run_queue *rq) { //初始化进程队列
    list_init(&(rq->run_list));//初始化运行队列
    rq->proc_num = 0;//初始化进程数为0
}
```
`RR_enqueue`函数将指定的进程的状态变为RUNNABLE，并且放入调用算法中的可执行队列中。
```C
//将进程加入就绪队列
static void RR_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(list_empty(&(proc->run_link)));//进程控制块指针非空
    //把进程的进程控制块指针放入到rq队列末尾
    list_add_before(&(rq->run_list), &(proc->run_link));
    //如果进程的时间片为0或者大于分配给进程的最大时间片
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;//修改时间片为最大时间片
    }
    proc->rq = rq;//加入进程池
    rq->proc_num ++;//就绪进程数加一
}
```
`RR_dequeue`函数将某个在队列中的进程取出。
```C
//将进程从就绪队列中移除
static void RR_dequeue(struct run_queue *rq, struct proc_struct *proc) {
    //保证进程控制块指针非空并且进程在就绪队列中
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
    //将进程从就绪队列中删除
    list_del_init(&(proc->run_link));
    rq->proc_num --;//就绪进程数减一
}
```
`RR_pick_next`函数选择要执行的下个进程
```C
//选择下一调度进程
static struct proc_struct *RR_pick_next(struct run_queue *rq) {
    //选取就绪进程队列rq中的队头队列元素
    list_entry_t *le = list_next(&(rq->run_list));
    if (le != &(rq->run_list)) {//取得就绪进程
        //返回进程控制块指针
        return le2proc(le, run_link);
    }
    return NULL;
}
```
`RR_proc_tick`函数会在时间中断处理例程中被调用，以减小当前运行进程的剩余时间片。若时间片耗尽，则设置当前进程的need_resched为1。
```C
static void RR_proc_tick(struct run_queue *rq, struct proc_struct *proc) {//时间片
    if (proc->time_slice > 0) {//到达时间片
        proc->time_slice --;//执行进程的时间片 time_slice 减一
    }
    if (proc->time_slice == 0) {//时间片为 0
        proc->need_resched = 1;//设置此进程成员变量 need_resched 标识为 1，进程需要调度
    }
}
```

uCore的调度执行过程：

- 首先，uCore调用sched_init函数用于初始化相关的就绪队列。

- 在proc_init函数中，建立第一个内核进程，并将其添加至就绪队列中。

- uCore执行cpu_idle函数，在其内部的schedule函数中，调用enqueue将当前进程添加进就绪队列中
然后，调用pick_next获取就绪队列中可被轮换至CPU的进程。如果存在可用的进程，则调用dequeue函数，将该进程移出就绪队列，并在之后执行proc_run函数进行进程上下文切换。

- 每次时间中断都会调用函数proc_tick。该函数会减少当前运行进程的剩余时间片。如果时间片减小为0，则设置need_resched为1，并在时间中断例程完成后，在trap函数的剩余代码中进行进程切换。


__思考题2__   
请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计;

设计如下：

- 在 proc_struct 中添加总共 N 个多级反馈队列的入口，每个队列都有着各自的优先级，编号越大的队列优先级约低，并且优先级越低的队列上时间片的长度越大，为其上一个优先级队列的两倍；并且在 PCB 中记录当前进程所处的队列的优先级；

- 处理调度算法初始化的时候需要同时对 N 个队列进行初始化；

- 在处理将进程加入到就绪进程集合的时候，观察这个进程的时间片有没有使用完，如果使用完了，就将所在队列的优先级调低，加入到优先级低 1 级的队列中去，如果没有使用完时间片，则加入到当前优先级的队列中去；

- 在同一个优先级的队列内使用时间片轮转算法；

- 在选择下一个执行的进程的时候，有限考虑高优先级的队列中是否存在任务，如果不存在才转而寻找较低优先级的队列；（有可能导致饥饿）

- 从就绪进程集合中删除某一个进程就只需要在对应队列中删除即可；

- 处理时间中断的函数不需要改变；

至此完成了多级反馈队列调度算法的具体设计；

## 练习2
### 实现Stride Scheduling调度算法（需要编码）
首先需要换掉RR调度器的实现，即用default_sched_stride_c覆盖default_sched.c。然后根据此文件和后续文档对Stride度器的相关描述，完成Stride调度算法的实现。  

首先，根据要求，先用default_sched_stride_c覆盖default_sched.c，即覆盖掉Round Robin调度算法的实现。 

覆盖掉之后需要在该框架上实现 Stride Scheduling 调度算法。 

Stride Scheduling调度算法思想如下：

- 为每个runnable的进程设置一个当前状态stride，表示该进程当前的调度权。另外定义其对应的pass 值，表示对应进程在调度后，stride需要进行的累加值。

- 每次需要调度时，从当前runnable态的进程中选择stride最小的进程调度。对于获得调度的进程P，将对应的 stride加上其对应的步长pass（只与进程的优先权有关系）。

- 在一段固定的时间之后，回到上个步骤，重新调度当前stride最小的进程。  

令`pass = BigStride / priority` 其中priority表示进程的优先级（大于 1），BigStride 表示一个预先定义的大常数，则该调度方案为每个进程分配的时间将与其优先级成正比。  

为了方便，直接将default_sched_stride.c在的代码复制到default_sched.c中，我们将实验如下的调度框架
```C
struct sched_class default_sched_class = {
     .name = "stride_scheduler",
     .init = stride_init,
     .enqueue = stride_enqueue,
     .dequeue = stride_dequeue,
     .pick_next = stride_pick_next,
     .proc_tick = stride_proc_tick,
};
```
接下来逐个补充里面的函数。  
首先是stride_init    
– 初始化调度器类的信息（如果有的话）。
– 初始化当前的运行队列为一个空的容器结构。（比如和RR调度算法一样，初始化为一个有序列表）
```C
static void stride_init(struct run_queue *rq) {
     list_init(&(rq->run_list));//初始化调度器类
     rq->lab6_run_pool = NULL;//对斜堆进行初始化，表示有限队列为空
     rq->proc_num = 0;//设置运行队列为空
}
```
enqueue:
– 初始化刚进入运行队列的进程 proc的stride属性。
– 将 proc插入放入运行队列中去（注意：这里并不要求放置在队列头部）。  
```C
static void stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
#if USE_SKEW_HEAP   //如果使用斜堆
     rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &(proc->lab6_run_pool),proc_stride_comp_f);
     //将新的进程插入到表示就绪队列的斜堆中，该函数的返回结果是斜堆的新的根
#else
     assert(list_empty(&(proc->run_link)));
     list_add_before(&(rq->run_list), &(proc->run_link));
#endif
     if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
          proc->time_slice = rq->max_time_slice;//将该进程剩余时间置为时间片大小
     }
     proc->rq = rq;//更新进程的就绪队列
     rq->proc_num ++;//维护就绪队列中进程的数量加一
}
```
dequeue：
– 从运行队列中删除相应的元素。
```C
static void stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
#if USE_SKEW_HEAP
     rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);//删除斜堆中的指定进程
#else
     assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
     list_del_init(&(proc->run_link));
#endif
     rq->proc_num --;//维护就绪队列中的进程总数
}
```
pick next:
– 扫描整个运行队列，返回其中stride值最小的对应进程。
– 更新对应进程的stride值，即pass = BIG_STRIDE / P->priority; P->stride += pass。
```C
static struct proc_struct *stride_pick_next(struct run_queue *rq) {
#if USE_SKEW_HEAP
     if (rq->lab6_run_pool == NULL) return NULL;
     struct proc_struct *p = le2proc(rq->lab6_run_pool, lab6_run_pool);
     //选择 stride 值最小的进程
#else
     list_entry_t *le = list_next(&(rq->run_list));

     if (le == &rq->run_list)
          return NULL;
     
     struct proc_struct *p = le2proc(le, run_link);
     le = list_next(le);
     while (le != &rq->run_list)
     {
          struct proc_struct *q = le2proc(le, run_link);
          if ((int32_t)(p->lab6_stride - q->lab6_stride) > 0)
               p = q;
          le = list_next(le);
     }
#endif
     if (p->lab6_priority == 0)//优先级为 0
          p->lab6_stride += BIG_STRIDE;//步长设置为最大值
     else p->lab6_stride += BIG_STRIDE / p->lab6_priority;
     //步长设置为优先级的倒数，更新该进程的 stride 值
     return p;
}
```
proc tick:
– 检测当前进程是否已用完分配的时间片。如果时间片用完，应该正确设置进程结构的相关标记来引起进程切换。
– 一个 process 最多可以连续运行 rq.max_time_slice个时间片。
```C
static void stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
     if (proc->time_slice > 0) {//到达时间片
          proc->time_slice --;//执行进程的时间片 time_slice 减一
     }
     if (proc->time_slice == 0) {//时间片为 0
          proc->need_resched = 1;//设置此进程成员变量 need_resched 标识为 1，进程需要调度
     }
}
```



相比于 RR 调度，Stride Scheduling 函数定义了一个比较器 proc_stride_comp_f。优先队列的比较函数 `proc_stride_comp_f` 的实现，主要思路就是通过步数相减，然后根据其正负比较大小关系。

```c
//proc_stride_comp_f：优先队列的比较函数，主要思路就是通过步数相减，然后根据其正负比较大小关系
static int proc_stride_comp_f(void *a, void *b)
{
     struct proc_struct *p = le2proc(a, lab6_run_pool);//通过进程控制块指针取得进程 a
     struct proc_struct *q = le2proc(b, lab6_run_pool);//通过进程控制块指针取得进程 b
     int32_t c = p->lab6_stride - q->lab6_stride;//步数相减，通过正负比较大小关系
     if (c > 0) return 1;
     else if (c == 0) return 0;
     else return -1;
}
```


> 如何证明STRIDE_MAX – STRIDE_MIN <= PASS_MAX？

假如该命题不成立，则可以知道就绪队列在上一次找出用于执行的进程的时候，假如选择的进程是 P，那么存在另外一个就绪的进程 P'，并且有 P' 的 stride 比 P 严格地小，这也就说明上一次调度出了问题，这和 stride 算法的设计是相违背的；因此通过反证法证明了上述命题的成立；

> 在 ucore 中，目前 Stride 是采用无符号的32位整数表示。则 BigStride 应该取多少，才能保证比较的正确性？

需要保证 ![](http://latex.codecogs.com/gif.latex?BigStride<2^{32}-1)

> 注：BIG_STRIDE 的值是怎么来的?

Stride 调度算法的思路是每次找 stride 步进值最小的进程，每个进程每次执行完以后，都要在 stride步进 += pass步长，其中步长是和优先级成反比的因此步长可以反映出进程的优先级。但是随着每次调度，步长不断增加，有可能会有溢出的风险。

因此，需要设置一个步长的最大值，使得他们哪怕溢出，还是能够进行比较。

在 ucore 中，BIG_STRIDE 的值是采用无符号 32 位整数表示，而 stride 也是无符号 32 位整数。也就是说，最大值只能为 ![](http://latex.codecogs.com/gif.latex?2^{32}-1)。

如果一个 进程的 stride 已经为 ![](http://latex.codecogs.com/gif.latex?2^{32}-1) 时，那么再加上 pass 步长一定会溢出，然后又从 0 开始算，这样，整个调度算法的比较就没有意义了。

这说明，我们必须得约定一个最大的步长，使得两个进程的步进值哪怕其中一个溢出或者都溢出还能够进行比较。

首先 因为 步长 和 优先级成反比 可以得到一条：`pass = BIG_STRIDE / priority <= BIG_STRIDE`

进而得到：`pass_max <= BIG_STRIDE`

最大步长 - 最小步长 一定小于等于步长：`max_stride - min_stride <= pass_max`

所以得出：`max_stride - min_stride <= BIG_STRIDE`

前面说了 ucore 中 BIG_STRIDE 用的无符号 32 位整数，最大值只能为 ![](http://latex.codecogs.com/gif.latex?2^{32}-1)

而又因为是无符号的，因此，最小只能为 0，而且我们需要把 32 位无符号整数进行比较，需要保证任意两个进程 stride 的差值在 32 位有符号数能够表示的范围之内，故 BIG_STRIDE 为 ![](http://latex.codecogs.com/gif.latex?(2^{32}-1)/2)。



## 扩展练习
### 实现 Linux 的 CFS 调度算法

CFS 算法的基本思路就是尽量使得每个进程的运行时间相同，所以需要记录每个进程已经运行的时间：

```c
struct proc_struct {
	...
    int fair_run_time;                          // FOR CFS ONLY: run time
};
```

每次调度的时候，选择已经运行时间最少的进程。所以，也就需要一个数据结构来快速获得最少运行时间的进程， CFS 算法选择的是红黑树，但是项目中的斜堆也可以实现，只是性能不及红黑树。CFS是对于**优先级**的实现方法就是让优先级低的进程的时间过得很快。

### 数据结构

首先需要在 run_queue 增加一个斜堆：

```c
struct run_queue {
	...
    skew_heap_entry_t *fair_run_pool;
};
```

在 proc_struct 中增加三个成员：

- 虚拟运行时间
- 优先级系数：从 1 开始，数值越大，时间过得越快
- 斜堆

```c
struct proc_struct {
	...
    int fair_run_time;                          // FOR CFS ONLY: run time
    int fair_priority;                          // FOR CFS ONLY: priority
    skew_heap_entry_t fair_run_pool;            // FOR CFS ONLY: run pool
};
```



##### 算法实现

###### proc_fair_comp_f

首先需要一个比较函数，同样根据 ![](http://latex.codecogs.com/gif.latex?MAX_RUNTIME-MIN_RUNTIE<MAX_PRIORITY) 完全不需要考虑虚拟运行时溢出的问题。

```
static int proc_fair_comp_f(void *a, void *b)
{
     struct proc_struct *p = le2proc(a, fair_run_pool);
     struct proc_struct *q = le2proc(b, fair_run_pool);
     int32_t c = p->fair_run_time - q->fair_run_time;
     if (c > 0) return 1;
     else if (c == 0) return 0;
     else return -1;
}
```

###### fair_init

```c
static void fair_init(struct run_queue *rq) {
    rq->fair_run_pool = NULL;
    rq->proc_num = 0;
}
```

###### fair_enqueue

和 Stride Scheduling 类型，但是不需要更新 stride。

```c
static void fair_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    rq->fair_run_pool = skew_heap_insert(rq->fair_run_pool, &(proc->fair_run_pool), proc_fair_comp_f);
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice)
        proc->time_slice = rq->max_time_slice;
    proc->rq = rq;
    rq->proc_num ++;
}
```

###### fair_dequeue

```c
static void fair_dequeue(struct run_queue *rq, struct proc_struct *proc) {
    rq->fair_run_pool = skew_heap_remove(rq->fair_run_pool, &(proc->fair_run_pool), proc_fair_comp_f);
    rq->proc_num --;
}
```

###### fair_pick_next

```c
static struct proc_struct * fair_pick_next(struct run_queue *rq) {
    if (rq->fair_run_pool == NULL)
        return NULL;
    skew_heap_entry_t *le = rq->fair_run_pool;
    struct proc_struct * p = le2proc(le, fair_run_pool);
    return p;
}
```

###### fair_proc_tick

需要更新虚拟运行时，增加的量为优先级系数。

```c
static void
fair_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
    if (proc->time_slice > 0) {
        proc->time_slice --;
        proc->fair_run_time += proc->fair_priority;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
}
```

##### 兼容调整

为了保证测试可以通过，需要将 Stride Scheduling 的优先级对应到 CFS 的优先级：

```c
void lab6_set_priority(uint32_t priority)
{
    ...
    // FOR CFS ONLY
    current->fair_priority = 60 / current->lab6_priority + 1;
    if (current->fair_priority < 1)
        current->fair_priority = 1;
}
```

由于调度器需要通过虚拟运行时间确定下一个进程，如果虚拟运行时间最小的进程需要 yield，那么必须增加虚拟运行时间，例如可以增加一个时间片的运行时。

```
int do_yield(void) {
    ...
    // FOR CFS ONLY
    current->fair_run_time += current->rq->max_time_slice * current->fair_priority;
    return 0;
}
```

> 遇到的问题：**为什么 CFS 调度算法使用红黑树而不使用堆来获取最小运行时进程？**
>
> 查阅了网上的资料以及自己分析，得到如下结论：
>
> - 堆基于数组，但是对于调度器来说进程数量不确定，无法使用定长数组实现的堆；
> - ucore 中的 Stride Scheduling 调度算法使用了斜堆，但是斜堆没有维护平衡的要求，可能导致斜堆退化成为有序链表，影响性能。
>
> 综上所示，红黑树因为平衡性以及非连续所以是CFS算法最佳选择。

- 堆基于数组，但是对于调度器来说进程数量不确定，无法使用定长数组实现的堆；
- ucore 中的 Stride Scheduling 调度算法使用了斜堆，但是斜堆没有维护平衡的要求，可能导致斜堆退化成为有序链表，影响性能。

综上所示，红黑树因为平衡性以及非连续所以是CFS算法最佳选择。

## 总结

### 知识点

#### 斜堆
斜堆可递归的定义如下：
- 只有一个元素的堆是斜堆。
- 两个斜堆通过斜堆的合并操作，得到的结果仍是斜堆。
- 合并时如果两个斜堆都非空，那么比较两个根节点，取较小堆的根节点为新的根节点。将"较小堆的根节点的右孩子"和"较大堆"进行合并。


#### [CFS](https://www.cnblogs.com/linhaostudy/p/10298511.html#:~:text=CFS%E6%98%AFCompletely%20Fair%20Scheduler%E7%AE%80%E7%A7%B0%EF%BC%8C%E5%8D%B3%E5%AE%8C%E5%85%A8%E5%85%AC%E5%B9%B3%E8%B0%83%E5%BA%A6%E5%99%A8%E3%80%82,CFS%E7%9A%84%E8%AE%BE%E8%AE%A1%E7%90%86%E5%BF%B5%E6%98%AF%E5%9C%A8%E7%9C%9F%E5%AE%9E%E7%A1%AC%E4%BB%B6%E4%B8%8A%E5%AE%9E%E7%8E%B0%E7%90%86%E6%83%B3%E7%9A%84%E3%80%81%E7%B2%BE%E7%A1%AE%E7%9A%84%E5%A4%9A%E4%BB%BB%E5%8A%A1CPU%E3%80%82%20CFS%E8%B0%83%E5%BA%A6%E5%99%A8%E5%92%8C%E4%BB%A5%E5%BE%80%E7%9A%84%E8%B0%83%E5%BA%A6%E5%99%A8%E4%B8%8D%E5%90%8C%E4%B9%8B%E5%A4%84%E5%9C%A8%E4%BA%8E%E6%B2%A1%E6%9C%89%E6%97%B6%E9%97%B4%E7%89%87%E7%9A%84%E6%A6%82%E5%BF%B5%EF%BC%8C%E8%80%8C%E6%98%AF%E5%88%86%E9%85%8Dcpu%E4%BD%BF%E7%94%A8%E6%97%B6%E9%97%B4%E7%9A%84%E6%AF%94%E4%BE%8B%E3%80%82%20%E4%BE%8B%E5%A6%82%EF%BC%9A2%E4%B8%AA%E7%9B%B8%E5%90%8C%E4%BC%98%E5%85%88%E7%BA%A7%E7%9A%84%E8%BF%9B%E7%A8%8B%E5%9C%A8%E4%B8%80%E4%B8%AAcpu%E4%B8%8A%E8%BF%90%E8%A1%8C%EF%BC%8C%E9%82%A3%E4%B9%88%E6%AF%8F%E4%B8%AA%E8%BF%9B%E7%A8%8B%E9%83%BD%E5%B0%86%E4%BC%9A%E5%88%86%E9%85%8D50%25%E7%9A%84cpu%E8%BF%90%E8%A1%8C%E6%97%B6%E9%97%B4%E3%80%82%20%E8%BF%99%E5%B0%B1%E6%98%AF%E8%A6%81%E5%AE%9E%E7%8E%B0%E7%9A%84%E5%85%AC%E5%B9%B3%E3%80%82)

#### 红黑树

- 节点不是黑色，就是红色（非黑即红）
- 根节点为黑色
- 叶节点为黑色（叶节点是指末梢的空节点 Nil或Null）
- 一个节点为红色，则其两个子节点必须是黑色的（根到叶子的所有路径，不可能存在两个连续的红色节点）
- 每个节点到叶子节点的所有路径，都包含相同数目的黑色节点（相同的黑色高度）

### 思考
进程调度是操作系统课程最先讲到的内容，时间过去比较久忘记的差不多了。实验刚好能回顾之前的知识，一举两得。