

> 内核用 `struct page`来管理物理页

该结构体定义在 `</include/linux/mm_types.h>` 文件中



物理页信息：

* 是否已分配
* 谁拥有该页



> 将页划分了四个区

定义在 *`</include/linux/mm_zone.h>`* 文件中

* ZONE_DMA
* ZONE_DMA32
* ZONE_NORMAL
* ZONE_HIGHEM



> 获取页

核心函数：**`alloc_pages()`** 

路径：*`</include/linux/gfp.h>`* 

```C
struct page* alloc_pages(gfp_t gfp,
                         unsigned int order,
                         int preferred_nid,
                         nodemask_t *nodemask);
```

分配的是若干个（**2<sup>order</sup>**）连续的物理页，返回的是第一个物理页结构体 page 的指针。



