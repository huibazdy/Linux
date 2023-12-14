# Linux_mm

> 即使一个进程的可寻址虚拟内存***地址空间***是 4 GB，并不代表该进程有权访问所有的虚拟地址。我们更关心的是虚拟内存的***地址区间***（可以理解为能被进程所访问的地址空间，是 4 GB 的一个子集）。
>
> 这些可以被访问的虚拟内存地址区间也被称为***内存区域（Virtual Memory Area，VMA）***。进程可以通过内核动态调整内存区域的大小。



内存区域包含以下几种对象：

* 可执行文件代码的内存映射，称为代码段（text section）
* 可执行文件已初始化的全局变量的内存映射，称为数据段（data section）
* 未初始化的全局变量，称为 bss 段（的零页，页面中信息全为 0 值）
* 进程用户空间栈（不要和进程的内核栈混淆，内核的进程栈独立存在并由内核维护）
* C 库或动态链接库的代码段、数据段和 bss 段
* 内存映射文件
* 共享内存段
* 匿名的内存映射，如 malloc() 分配的内存



## 内存描述符

Linux 内核通过一个名为内存描述符（**`struct mm_struct`**）的结构体来描述一个进程的地址空间，定义在 *`<include/linux/mm_types.h>`* 文件中。



```c
struct mm_struct {
    struct vm_area_struct    *mmap;      // VMA 链表
    struct rb_root           mm_rb;      // VMA 红黑树
    atomic_t                 mm_users;   // 共有几个进程（线程）在共享该地址空间
    atomic_t                 mm_count;   // 引用计数器
    struct list_head         mmlist;     // 每个 mm_struct 通过 mmlist 域链接到一个双向链表中
};
```



### 撤销内存描述符

具体步骤如下：

1. **`exit_mm()`**

    执行常规撤销工作，并更新一些统计量

2. **`mmput()`**

    减少 mm_struct 中的 **mm_users** 直至 **0**

3. **`mmdrop()`**

    减少 mm_struct 中的 **mm_count** 直至 **0**

4. **`free_mm()`**

    调用 **`kmem_cache_free()`**将 mm_struct 归还到 **mm_cachep slab** 缓存中



### 内核线程的内存描述

内核线程没有进程地址空间，也没有相关内存描述符，所以它们对应的 task_struct 中的 mm 域为空



## 进程描述符

通过 **`struct task_struct`**结构体来描述，定义在 *`<include/linux/sched.h>`*中



```c
struct task_struct {
    struct mm_struct *mm;  // 记录该进程使用的内存描述符
    ...
};
```



通常，进程有唯一的内存描述符（mm_struct），即唯一的进程地址空间。



在Linux中，是否共享地址空间几乎是进程和线程的唯一区别。可以将线程理解为：共享特定资源（地址空间）的进程。