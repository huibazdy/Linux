# Linux 内存管理

## 内存空间

> 一个进程的可寻址虚拟内存***地址空间***是 4 GB，但并不代表该进程有权访问所有的虚拟地址。
>
> 我们更关心的是能被进程所访问的地址空间，即虚拟内存的***地址区间***。这个地址区间可以理解为是可寻址地址空间（4 GB） 的一个子集。本文之后提到的“地址空间”多是这个含义。
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



## 进程描述符

Linux 中的进程或线程都是通过 **`task_struct`**结构体来描述和管理的，定义在`<include/linux/sched.h>`：

```c
struct task_struct {
    struct mm_struct *mm;            // 记录该进程使用的内存描述符
    struct mm_struct *active_mm;     // 当进程被调度（地址空间被装载到内存），这个域会被更新
    ...
};
```

是否共享地址空间几乎是进程和线程的唯一区别。可以将线程理解为：共享地址空间的进程。



## 内存描述符

Linux 内核通过一个名为内存描述符（**`struct mm_struct`**）的结构体来描述一个进程的地址空间，定义在 *`<include/linux/mm_types.h>`* 中。通常来说，进程的内存描述符（mm_struct）是唯一的，即进程只有唯一的地址空间。

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

内核线程没有进程地址空间，也没有相关内存描述符，所以它们对应的 task_struct 中的 mm 域为空（NULL）。

> 内核线程直接使用前一个进程的内存描述符。





## 虚拟内存区域

一个进程使用的虚拟内存区域（**Virtual Memory Areas，VMA**）由结构体 **`vm_area_struct`** 描述，它定义在文件*`<include/linux/mm_types.h>`*中：
```c
struct vm_area_struct {
    struct mm_struct       *vm_mm;          // 与当前 VMA 相关的 mm_struct 结构体
    unsigned long          vm_start;        // 该 VMA 虚拟地址区间的首地址，在区间内
    unsigned long          vm_end;          // 该 VMA 虚拟地址区间的尾地址，在区间外
    struct vm_area_struct  *vm_next;        // 多个 VMA 组成双向链表，该域指明下一个 VMA 节点
    struct vm_area_struct  *vm_prev;        // 多个 VMA 组成双向链表，该域指明上一个 VMA 节点
    struct rb_root         vm_rb;           // 每个 vm_area_struct 通过这个
    pgprot_t               vm_page_prot;    // 该 VMA 的访问控制权限
    unsigned long          vm_flags;        // 标识位
    ...
    const struct vm_operations_struct *vm_ops  // 用于处理 VMA 的操作函数指针的集合
    ...
};
```



> ***怎样理解 VMA 结构体、内存区域和地址空间？***

每个 vm_area_struct 管理进程的一个虚拟内存区域，所属这个进程的多个 vm_area_struct 所管理的总的虚拟内存区域就是该进程的地址空间。



> ***一个 VMA 的长度是多少？***

其中 `vm_end - vm_start` 就是内存区间的长度，单位是字节。



> ***多个 VMA 使用哪种数据结构来组织？***

从结构体的定义可以看出 Linux 内核提供了两种组织 VMA 的方式：双向链表以及红黑树。链表用于需要遍历全部 VMA 节点时，红黑树用于在地址空间中定位特定内存区域时。

### VMA 访问控制

vm_flags 标识了一个VMA所管理的虚拟内存区域所包含页面的行为和信息，以下是几个最常见的 VMA 标识：

|    标识     | 对VMA的及其页面的影响 |
| :---------: | :-------------------: |
|  `VM_READ`  |      页面可读取       |
| `VM_WRITE`  |       页面可写        |
|  `VM_EXEC`  |      页面可执行       |
| `VM_SHARED` |      页面可共享       |
|     ...     |                       |

这些标识位通过组合构成 **VMA** 的访问控制权限。例如进程的可执行代码中的数据段或代码段，映射到相应的区域 时，该  **VMA** 仅标志为 **VM_READ** 。



### VMA 操作集合

对虚拟内存区域 VMA 的操作的一系列集合由结构体 `vm_opearations_struct` 描述，其中每一个函数指针就是一个对应的操作，定义在 `<include/linux/mm.h>` 中。

```c
struct vm_operations_struct {
    void (*open)(struct vm_area_struct * area);      // 指定内存区域被加入到地址空间时调用
    void (*close)(struct vm_area_struct * area);     // 指定内存区域被从地址空间删除时调用
    ...
};
```



## 实例分析

我们可以通过编写一个用户空间的程序实例来分析下内存管理的结果。

```c
#include <stdio.h>
#include <unistd.h>

int main()
{
	while(1) {
		printf("Hello!\n");
		sleep(2);  // print once every two second
	} 
	return 0;
}
```

该进程pid为2086，再执行命令:`cat /proc/2086/maps`输出该进程地址空间中的全部内存区域，结果如下：

![](https://raw.githubusercontent.com/huibazdy/TyporaPicture/main/1.png)

数据格式：开始-结束      访问权限      偏移      主设备号 : 次设备号      文件索引节点号      文件

*  C 库中 libc.so 的代码段/数据段和bss段
* 可执行程序的代码段和数据段
* 动态链接库的代码段和数据段
* 进程的栈（倒数第四行）



> 代码段：可读可执行
>
> 数据段和bss段：可读可写不可执行
>
> 堆栈：可读可写可执行



也可以使用命令：`pmap 2086`，获取更方阅读的输出形式，结果如下：

![](https://raw.githubusercontent.com/huibazdy/TyporaPicture/main/pmap2.png)



## 常用 VMA 函数



### find_vma( )

> **给定一个内存地址，第一个 vm_end 大于 addr 的 VMA**

函数声明在*`<include/linux/mm.h>`*中：

```c
extern struct vm_area_strcut * find_vma(struct mm_struct * mm, unsigned long addr);
```

返回结果缓存在内存描述符（mm_struct）的 **mmap_cache** 域中。

函数实现在*`<mm/mmap.c>`*中：

```c
/* Look up the first VMA which satisfies  addr < vm_end,  NULL if none. */
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)
{
	struct rb_node *rb_node;
	struct vm_area_struct *vma;

	mmap_assert_locked(mm);
	/* Check the cache first. */
	vma = vmacache_find(mm, addr);
	if (likely(vma))
		return vma;

	rb_node = mm->mm_rb.rb_node;

	while (rb_node) {
		struct vm_area_struct *tmp;

		tmp = rb_entry(rb_node, struct vm_area_struct, vm_rb);

		if (tmp->vm_end > addr) {
			vma = tmp;
			if (tmp->vm_start <= addr)
				break;
			rb_node = rb_node->rb_left;
		} else
			rb_node = rb_node->rb_right;
	}

	if (vma)
		vmacache_update(addr, vma);
	return vma;
}
```



### find_vma_pre( )

> **给定一个地址，返回第一个小于 addr  的 VMA**

函数声明在*`<include/linux/mm.h>`*中：

函数定义在*`<mm/mmap.c>`*中：





### find_vma_insetion( )

> **找到第一个和给定地址所在区间相交的 VMA**

因为是内联函数，所以定义在*`<include/linux/mm.h>`*中





## 虚拟内存与物理内存的关系

一图说明：

![](https://raw.githubusercontent.com/huibazdy/TyporaPicture/main/111.png)



## 创建新区间

### do_mmap( )

内核创建一个新的内存区域或者扩展已经存在的内存区域，都会调用这个函数，将新的内存区域加到进程的虚拟地址空间中去。该函数定义在`<include/linux/mm.h>`中：

```c
extern unsigned long do_mmap(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot, unsigned long flags,
	unsigned long pgoff, unsigned long *populate, struct list_head *uf);
```



## 内存分布图解

在 32 位机器上，对一个进程来说，理论可寻址虚拟地址为 4 GB：

![](https://raw.githubusercontent.com/huibazdy/TyporaPicture/main/%E5%86%85%E5%AD%98%E5%88%86%E5%B8%831.png)

需要注意的是其中绿色区域（高地址向低地址增长）：

1. 程序运行所依赖的动态链接库中的代码段、数据段和 bss 段；
2. 调用 mmap 映射出来的虚拟内存



实际上 32 位机器完整虚拟内存分布图如下（箭头代表内存增长方向）：

![](https://raw.githubusercontent.com/huibazdy/TyporaPicture/main/%E5%86%85%E5%AD%98%E5%88%86%E5%B8%8303.png)

* 堆上面的一段待分配区域用于扩展堆。当堆申请新的内存空间时，只需增加 brk 指针来增加对应空间大小，回收时减小 brk 指针即可
*  



## 参考资料

1. [一步一图理解 Linux 虚拟内存](https://www.cnblogs.com/binlovetech/p/16824522.html)