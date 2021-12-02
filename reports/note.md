空闲块的头节点：
```C
/* free_area_t - maintains a doubly linked list to record free (unused) pages */
typedef struct {
 list_entry_t free_list;         // the list header
 unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```
只是一个头节点，nr_free是一共有多少个空闲页。   

物理页：
```C
struct Page {
 int ref;                        // page frame's reference counter
 uint32_t flags;                 // array of flags that describe the status of the page frame
 unsigned int property;          // the num of free block, used in first fit pm manager
 list_entry_t page_link;         // free list link
};
```
页大小`PGSIZE`=4096   property是空闲块大小---地址连续的空闲页的个数。  
只有空闲块的头一页用到page_link，是把多个连续内存空闲块链接在一起的双向链表指针
连续内存空闲块利用这个页的成员变量page_link来链接比它地址小和大的其他连续内存空闲块。

| 缩写      | 含义     |
| ----------- | -----------     |
| PDT         | 页目录基址                           |
| PDE         | 一级页表页表项，二级页表基址          |


用于管理虚拟内存页面
```C
// the control struct for a set of vma using the same PDT
struct mm_struct {
    list_entry_t mmap_list;        // 按照虚拟地址顺序双向连接的虚拟页链表
    struct vma_struct *mmap_cache; // 当前使用的虚拟页地址，该成员加速页索引速度。
    pde_t *pgdir;                  // 虚拟页对应的PDT
    int map_count;                 // 虚拟页个数
    void *sm_priv;                 // 用于指向swap manager的某个链表,在FIFO算法中，该双向链表用于将可交换的已分配物理页串起来
};
```
一个mm_struct管理若干个vma_struct 
```C
// 用于描述某个虚拟页的结构
struct vma_struct {
    struct mm_struct *vm_mm; // 管理该虚拟页的mm_struct
    uintptr_t vm_start;      // 虚拟页起始地址，包括当前地址  
    uintptr_t vm_end;        // 虚拟页终止地址，不包括当前地址（地址前闭后开）  
    uint32_t vm_flags;       // 相关标志位
    list_entry_t list_link;  // 用于连接各个虚拟页的双向指针
};
```
<div align="center">
<img src="./img/VMMM.png" alt="VMMM" width="500"/>
</div>