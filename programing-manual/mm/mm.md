# 内存管理

内存管理是操作系统内核中重中之重的内容，本章将带你了解xbook2是如何管理内存的。xbook2的内存管理源于linux内存管理，但是我们做得更简单。

## 物理内存

物理内存是实际上可以使用的内存大小，比如4G的内存条，有4G内存。但是由于某些原因，一些设备的寄存器被映射到内存中，可以通过操作内存从而操作设备。那么就会导致这部分内存不能拿来储存数据，这就是一些内存空洞。因此，在我们的物理内存中，我们忽略了这些区域的内存地址，我们把可用的内存收集起来，将它们以页（4096 bytes）的形式进行管理。每个页就是最小的物理内存管理单位，分配物理内存时最小都是一个页。至于为什么要以页来分配物理内存，这是由于分页机制导致的，分页机制是以页面为管理的基础单位，物理内存正好以页面来管理，则可以很好地适配分页机制。

在xbook2中，物理内存是以节点为管理的，一个节点可以是一个页，也可以是两个，也可以是N个。

```c
typedef struct _mem_node {
    unsigned int count;
    unsigned int flags;         
    int reference; 
    list_t list;
    mem_section_t *section;
    mem_cache_t *cache;
    mem_group_t *group;
} mem_node_t;
```



除此之外，还有一个内存节，每个节中存放了相同大小的节点。目前有12种节，每个节的页数量分别是2^0,2^1,2^2,....2^10,2^11个页。分配内存时，就从某个节种去分配节点。

```c
#define MEM_SECTION_MAX_NR      12
#define MEM_SECTION_MAX_SIZE    2048    // (2 ^ 11) : 8 MB

typedef struct {
    list_t free_list_head;
    size_t node_count;
    size_t section_size;
} mem_section_t;
```

而节的管理，是通过内存区域管理的，每个区域，可以管理不同大小的节。目前支持3种类型的内存区域，分别是`MEM_RANGE_DMA、MEM_RANGE_NORMAL和MEM_RANGE_USER`，他们的定义如下：

```c
enum {
    MEM_RANGE_DMA = 0,
    MEM_RANGE_NORMAL,
    MEM_RANGE_USER,
    MEM_RANGE_NR
};

typedef struct {
    unsigned int start;
    unsigned int end;
    size_t pages;
    mem_section_t sections[MEM_SECTION_MAX_NR];
    mem_node_t *node_table;
} mem_range_t;
```

最后，通过2个接口来进行物理页的分配与释放。

```c
/* 
分配count个页
flags传入不同范围的页：
MEM_NODE_TYPE_DMA： 从MEM_RANGE_DMA中分配页
MEM_NODE_TYPE_NORMAL： 从MEM_RANGE_NORMAL中分配页
MEM_NODE_TYPE_USER： 从MEM_RANGE_USER中分配页
*/
unsigned long mem_node_alloc_pages(unsigned long count, unsigned long flags);
/* 释放页地址对应的页 */
int mem_node_free_pages(unsigned long page);
```

## 分页机制

分页机制使得虚拟内存管理得以实现。支持分页的处理器都有一个特点，就是有一个记录页表的寄存器，在开启分页后，寻址就需要通过页表进行转换，才能访问到物理地址。因此，我们需要提供页表映射和解除映射，以及地址转换的函数。

```c
int page_map_addr(unsigned long start, unsigned long len, unsigned long prot);
int page_unmap_addr(unsigned long vaddr, unsigned long len);

#define kern_vir_addr2phy_addr(x) ((unsigned long)(x) - KERN_BASE_VIR_ADDR)
#define kern_phy_addr2vir_addr(x) ((void *)((unsigned long)(x) + KERN_BASE_VIR_ADDR)) 
```

不过这种内存地址转换只适合与内核，因为内核在执行时，做了内存映射，直接把物理地址和虚拟地址做了一一映射，只要和内核基地址进行加或者减，就能得到对应的物理地址和虚拟地址了。

内核在打开分页模式前，需要做页表映射，映射后才能正常执行，不然访问的地址都是未映射的，就出错了。

## 内核内存管理

内核内存管理是使用的memcache来管理的，每一个mem_cache管理着大小一样的内存对象，而这些内存对象则由mem_group来管理。

```c
typedef struct mem_group {
    list_t list;           // 指向cache中的某个链表（full, partial, free）
    bitmap_t map;          // 管理对象分配状态的位图
    unsigned char *objects;     // 指向对象群的指针
    unsigned long using_count;    // 正在使用中的对象数量
    unsigned long free_count;     // 空闲的对象数量
    unsigned long flags;         // group的标志
} mem_group_t;

#define SIZEOF_MEM_GROUP sizeof(mem_group_t)

#define MEM_CACHE_NAME_LEN 24

typedef struct mem_cache {
    list_t full_groups;      // group对象都被使用了，就放在这个链表
    list_t partial_groups;   // group对象一部分被使用了，就放在这个链表
    list_t free_groups;      // group对象都未被使用了，就放在这个链表

    unsigned long object_size;    // group中每个对象的大小
    flags_t flags;              // cache的标志位
    unsigned long object_count;  // 每个group中有多少个对象
    mutexlock_t mutex;
    char name[MEM_CACHE_NAME_LEN];     // cache的名字
} mem_cache_t;
```

在初始化的时候，会初始化一些最基础的mem_cache，在使用过程中可以额外增加。对于cache的大小，可以使用cache_size来进行描述。

```c
typedef struct cache_size {
    unsigned long cache_size;         // 描述cache的大小
    mem_cache_t *mem_cache;      // 指向对应cache的指针
} cache_size_t;

static cache_size_t cache_size[] = {
	#if PAGE_SIZE == 4096
	{32, NULL},
	#endif
	{64, NULL},
	{128, NULL},
	{256, NULL},
	{512, NULL},
	{1024, NULL},
	{2048, NULL},
	{4096, NULL},
	{8*1024, NULL},
	{16*1024, NULL},
	{32*1024, NULL},
	{64*1024, NULL},
	{128*1024, NULL},	
	/* 配置大内存的分配 */
	#ifdef CONFIG_LARGE_ALLOCS
	{256*1024, NULL},
	{512*1024, NULL},
	{1024*1024, NULL},
	{2*1024*1024, NULL},
	#endif
	{0, NULL},
};
```

初始化mem_cache时，根据cache_size进行初始化。

```c
static int mem_caches_build()
{
	cache_size_t *cachesz = cache_size;
	mem_cache_t *mem_cache = &mem_caches[0];
	while (cachesz->cache_size) {
		if (mem_cache_init(mem_cache, "mem cache", cachesz->cache_size, 0)) {
			keprint("create mem cache failed!\n");
			return -1;
		}
		cachesz->mem_cache = mem_cache;
		mem_cache++;
		cachesz++;
	}
	return 0;
}
```

在内核中，分配和释放内存，只要在初始化memcache后就可以使用了。

```c
/* 分配内存 */
void *mem_alloc(size_t size);
/* 释放内存 */
void mem_free(void *object);
```

## 用户内存管理

用户内存的管理即进程的内存管理，使用的是mem_space。进程的执行空间布局如下：

```c
#define USER_SPACE_START_ADDR              0x00000000
#define USER_SPACE_SIZE                    0x80000000

USER_SPACE_START_ADDR + USER_SPACE_SIZE
栈
映射区域
堆
数据
代码
USER_SPACE_START_ADDR
```

每一个space对应着一个区域。比如有代码space，数据space，堆space，栈space。

```c
typedef struct mem_space {
    unsigned long start;        /* 空间开始地址 */
    unsigned long end;          /* 空间结束地址 */
    unsigned long page_prot;    /* 空间保护 */
    unsigned long flags;        /* 空间的标志 */
    vmm_t *vmm;                 /* 空间对应的虚拟内存管理 */
    struct mem_space *next;     /* 所有空间构成单向链表 */
} mem_space_t;
```

当执行一个应用程序的时候，会初始化code和data的space，以及还要创建一个stack的space。在程序执行过程中，可以使用sys_brk系统调用来修改堆的位置，扩展堆。然后就可以对当前堆进行分配和释放了。

