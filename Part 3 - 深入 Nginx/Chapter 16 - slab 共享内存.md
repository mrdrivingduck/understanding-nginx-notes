# Chapter 16 - slab 共享内存

Created by : Mr Dk.

2020 / 08 / 08 21:18

Nanjing, Jiangsu, China

---

## 16.1 操作 slab 共享内存的方法

- `void ngx_slab_init(ngx_slab_pool_t *pool);` - 初始化共享内存 (由 Nginx 框架自动调用)
- `void *ngx_slab_alloc(ngx_slab_pool_t *pool, size_t size);` - 加锁保护的内存分配函数
- `void *ngx_slab_alloc_locked(ngx_slab_pool_t *pool, size_t size);` - 不加锁保护的内存分配函数
- `void ngx_slab_free(ngx_slab_pool_t *pool, void *p);` - 加锁保护的内存释放函数
- `void ngx_slab_free_locked(ngx_slab_pool_t *pool, void *p);` 不加锁保护的内存释放函数

由于 slab 大部分时间都要跨进程通信，因此不加锁的分配、释放函数较少使用。

Slab 内存池在解析配置文件时被初始化。当执行 `-s reload` 时，Nginx 会重新加载配置文件，再次初始化 slab 共享内存池。

```c
typedef struct ngx_shm_zone_s  ngx_shm_zone_t;

typedef ngx_int_t (*ngx_shm_zone_init_pt) (ngx_shm_zone_t *zone, void *data);

struct ngx_shm_zone_s {
    void                     *data; // 回调函数的参数
    ngx_shm_t                 shm; // 描述共享内存的结构体
    ngx_shm_zone_init_pt      init; // slab 内存池创建成功后回调
    void                     *tag;
    void                     *sync;
    ngx_uint_t                noreuse;  /* unsigned  noreuse:1; */
};
```

## 16.3 slab 内存管理的实现原理

动态管理内存面对的主要问题：

- 时间上，使用者随时可能分配内存、释放内存
- 空间上，每次申请分配内存的大小随机

这两个问题使得内存中会产生碎片，从而导致内存的浪费。常见内存分配算法的两种设计方向：

- First-fit - 首次适应，从头遍历空闲内存块链表，找到第一个符合需求的内存块 (快，但可能产生碎片)
- Best-fit - 找到与所需分配的内存最为匹配的内存块 (遍历慢些，但是产生碎片的概率小)

Nginx 基于 best-fit 的思路分配内存，但是需要以更快的速度找到合适大小的内存。Nginx 假定所有需要使用 slab 内存的模块请求分配的内存都不超过 4KB。基于这个假设，Nginx 中的内存块分配方法如下：

1. 把整块内存按 4KB 分页，每页存放 **固定大小** 的内存块，页首存放 bitmap (通过 bitmap 遍历空闲内存块)
2. Slab 内存分配器把请求分配的内存大小简化为有限几种 (2 字节的倍数)
3. 让有限的几种大小相同的页面构成链表，按内存块大小排序保存在一个数组中
4. 页面中分为空闲页、半满页、全满页，全满页脱离链表，分配内存时不再访问
5. 对于大于 4KB 页的内存分配需求，寻找地址连续的几个空闲页来分配

### 16.3.1 内存结构布局

每一个 slab 内存池对应着一块共享内存。内存首地址开始的地方保存的是 slab 管理结构，之后才是分配给使用者的内存：

```c
typedef struct {
    ngx_shmtx_sh_t    lock; // 使用信号量进行进程同步时会用到

    size_t            min_size; // 最小内存块长度
    size_t            min_shift; // 最小内存块长度的偏移 (1 << ?)

    ngx_slab_page_t  *pages; // 所有的内存页
    ngx_slab_page_t  *last;
    ngx_slab_page_t   free; // 所有的空闲页链表

    ngx_slab_stat_t  *stats;
    ngx_uint_t        pfree;

    u_char           *start; // 所有的实际页面的首地址
    u_char           *end; // 共享内存尾部

    ngx_shmtx_t       mutex; // 互斥锁

    u_char           *log_ctx; // slab 操作失败时用于记录日志
    u_char            zero; // '\0'

    unsigned          log_nomem:1;

    void             *data; // 由使用 slab 的模块自由使用
    void             *addr; // 执行所属 ngx_shm_t 中的 addr 成员
} ngx_slab_pool_t;
```

为了描述每一个内存页，都要使用一个 `ngx_slab_page_t` 结构体：

```c
typedef struct ngx_slab_page_s  ngx_slab_page_t;

struct ngx_slab_page_s {
    uintptr_t         slab; // (多用途)
    ngx_slab_page_t  *next; // 下一页
    uintptr_t         prev; // (多用途)，同时指向双向链表的前一页
};
```

所有内存页可被分为全满页、半满页、空闲页。全满页不在任何链表中，因此其中的 `next` 和 `prev` 没有任何链表的功能。

所有的空闲页则构成一个双向链表，由 `free` 指针指向这个链表 - 但 **并不是每个空闲页都是链表中的元素**。首页面的 `slab` 成员大于 1 时表示其后有相邻的页面，这些相邻页面作为一个链表元素而存在。由于内存的分配与回收时机不同，可能出现相邻的内存块各占一个链表元素的情况。当 slab 内存池刚刚初始化完毕时，`free` 链表中只有一个元素 - 就是第一个页面，`slab` 的值为总页数。

对于半满页，存放相同大小内存块的页面会构成双向链表，存放在 `slot` 数组的相应位置上。

### 16.3.2 分配内存流程

首先需要判断用户申请的内存是否大于 `ngx_slab_max_size` - 如果大于这个值，说明要按照页面来分配内存；否则只用一个页面中的内存块就可以实现分配。如果要按页来分配内存，那么首先需要计算分配多少个页面才能存放用户申请的内存字节数 (不足一页时按一页计算)。

```c
ngx_uint_t pages = (size >> ngx_pagesize_shift) + ((size % ngx_pagesize) ? 1 : 0);
```

在计算出页数后，需要遍历 `free` 空闲页链表，找出能够容纳用户申请的字节数的 **连续内存**：

```c
static ngx_slab_page_t *
ngx_slab_alloc_pages(ngx_slab_pool_t *pool, ngx_uint_t pages)
{
    ngx_slab_page_t  *page, *p;

    // 遍历 free 链表
    for (page = pool->free.next; page != &pool->free; page = page->next) {

        // 链表元素的 slab 数量已经达到所需要分配的页数
        if (page->slab >= pages) {

            // slab 在分配后还会有空闲页
            if (page->slab > pages) {
                page[page->slab - 1].prev = (uintptr_t) &page[pages];

                // slab 中第一个不会被分配的空闲页
                // 修改其指针，使其作为链表元素
                page[pages].slab = page->slab - pages;
                page[pages].next = page->next;
                page[pages].prev = page->prev;

                // 修改链表前后结点的指针
                p = (ngx_slab_page_t *) page->prev;
                p->next = &page[pages];
                page->next->prev = (uintptr_t) &page[pages];

            } else {
                // slab 中的页被完全分配了
                // 直接将这个元素移出链表
                p = (ngx_slab_page_t *) page->prev;
                p->next = page->next;
                page->next->prev = page->prev;
            }

            // 设置被分配内存的属性
            page->slab = pages | NGX_SLAB_PAGE_START;
            page->next = NULL;
            page->prev = NGX_SLAB_PAGE;

            // 内存池中的空闲页数减少
            pool->pfree -= pages;

            // 如果只分配了一个页，那么可以直接返回了
            if (--pages == 0) {
                return page;
            }

            // 如果不止分配了一个页，那么设置后续内存页的属性
            for (p = page + 1; pages; pages--) {
                p->slab = NGX_SLAB_PAGE_BUSY;
                p->next = NULL;
                p->prev = NGX_SLAB_PAGE;
                p++;
            }

            return page;
        }
    }

    // 没有找到符合要求的内存页面
    if (pool->log_nomem) {
        ngx_slab_error(pool, NGX_LOG_CRIT,
                       "ngx_slab_alloc() failed: no memory");
    }

    return NULL;
}
```

如果被分配的内存大小在一页以内，那么就要计算符合这个内存大小的最小内存块了。Slab 页面上允许存放的内存块以 8 字节起步，以 2 为倍数递增。此时先要根据 best-fit 原则找到最适合的内存块大小 (比如 64B)。然后取出半满页的链表，取得相应块大小的内存页链表 (全是 64B 内存块的半满页)。遍历内存页链表，如果没有找到半满页，那么分配一个新的页用于存放这个大小的内存块。

在每个半满页中，根据页大小和内存块大小，可以计算出 bitmap 的长度。然后找到 bitmap 中的第一个空闲块；如果没有空闲块，那这就不是一个半满页了，因此需要把它从链表中移走，然后遍历下一个半满页。如果找了到空闲块，在记录该块被使用以后，还要判断这是不是当前页的最后一个空闲页，如果是，那么也要将其移出链表。

如果分配了新页用于存放内存块，那么还要计算出 bitmap 后，将 bitmap 本身占用的内存块标记为已使用。新页面由空闲页变为半满页，因此会被插入到半满页链表相应内存块大小的链表的首部。

### 16.3.3 释放内存流程

首先判断内存块地址是否合法，然后得到内存块所属页的页描述结构体，判断出页面是整页内存还是内存块页面。如果是内存块页面，那么还要计算出 bitmap 的长度。将 bitmap 中相应的位置为空闲，如果该页是一个全满页，在释放后将变为半满页，那么还要将其插入半满页链表中。另外，还要检测 bitmap 中是否还有 0 的位，如果没有，那么说明变为空闲页，还要插入回 `free` 链表中。

---

在内存分配和释放的过程中，Nginx 大量使用了位操作来提高性能。
