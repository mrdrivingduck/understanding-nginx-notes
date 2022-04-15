# Chapter 8.7 - Nginx 内存池

Created by : Mr Dk.

2020 / 07 / 20 20:19

Nanjing, Jiangsu, China

---

## 8.7 `ngx_pool_t` 内存池

Nginx 已经对 `malloc()` 和 `free()` 进行了封装。另外提供了内存池的原因是降低程序员犯错的概率：只需要关心内存的分配，释放交由内存池来负责。通常来说，只有在内存池被销毁时，内存才会被释放回操作系统。在一个内存池上，可以任意次地申请内存不用释放，只需要最后销毁即可。

> 如果内存池生命周期很长，而其中每块内存的生命周期很短，会造成内存浪费吗？这与 Nginx 的应用场景有关。每个 TCP 连接都有明确的生命周期，每个 HTTP 请求有非常短暂的生命周期。如果每个请求、连接都有各自的内存池，那么请求结束时内存就会被回收了。

另外，内存池还对 **小块内存** 的频繁分配做了优化。小块内存的标准：

```c
#define NGX_MAX_ALLOC_FROM_POOL (ngx_pagesize - 1)
```

Nginx 处理大块内存和小块内存的方式是不一样的。内存池的定义如下：

```c
typedef struct ngx_pool_s ngx_pool_t;

struct ngx_pool_s {
    ngx_pool_data_t d; // 小块内存池单链表
    size_t max; // 小块内存与大块内存的阈值
    ngx_pool_t *current; // 指向分配内存池时遍历的第一个小块内存池
    ngx_pool_large_t *large; // 大块内存链表
    ngx_pool_cleanup_t *cleanup; // 所有待清理资源构成的单链表
    ngx_log_t *log; // 日志
};
```

当分配大块内存时，直接调用 `ngx_alloc()` 从进程堆上分配，同时还会分配一个 `ngx_pool_large_t` 结构体，挂到 `large` 链表中：

```c
typedef struct ngx_pool_large_s ngx_pool_large_t;

struct ngx_pool_large_s {
    ngx_pool_large_t *next; // 连接下一个大块内存结构体
    void *alloc; // 指向大块内存
};
```

如果大块内存的生命周期远小于内存池，那么提前释放是很有必要的。`ngx_pfree()` 实现了这个功能，它会遍历 `large` 链表，找到待释放的 `alloc` 并释放内存，但会保留 `ngx_pool_large_t` 结构体以便复用。

对于小块内存，内存池从进程堆中预分配更多的内存，然后直接使用这块内存的一小部分返回给申请者，减少了内存碎片与调用 `malloc()` 的次数。预分配的内存通过维护如下结构体来进行记录：

```c
typedef struct {
    u_char *last; // 未分配的空闲内存首地址
    u_char *end; // 当前小块内存的尾部

    ngx_pool_t *next; // 下一个小块内存结构体

    ngx_uint_t failed; // 在这个小块内存中分配失败的次数
} ngx_pool_data_t;
```

当这一小块预分配内存的空闲空间不足使用时，会再分配一块相同大小的内存池，通过 `next` 指针相连，并将 `failed` + 1。如果 `failed` 已经大于 4，那么内存池结构体中的 `current` 指针指向下一个小块内存池。下次开始分配内存时，就不会再从第一个内存池开始分配了。

另外，内存池还希望能够代替程序员释放 (文件) 资源。`ngx_pool_cleanup_t` 结构体实现了这个功能：

```c
typedef void (*ngx_pool_cleanup_pt) (void *data);

typedef struct ngx_pool_cleanup_s ngx_pool_cleanup_t;

struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt handler; // 清理函数指针
    void *data; // 函数参数
    ngx_pool_cleanup_t *next; // 链接下一个结构体
};
```

从内存池中分配内存的步骤：

1. 将申请的内存大小与 `max` 成员比较，决定申请大块内存还是小块内存
2. 如果申请小块内存，则首先取得 `current` 指针指向的内存池，并试图在其中分配内存；如果 `failed` 已经超过 4，则剩余空间多半已经无法满足申请内存
3. 找到 `last` 指针之后最近的对齐地址
4. 比较对齐地址与 `end` 指针之间是否可以容纳申请内存的大小，如果不行，则检查 `next` 指针是否为空，如果为空，则申请新的小块内存池；如果不为空，则向下一个小块内存池申请内存
5. 将 `last` 指针置为下一个空闲内存首地址，返回分配内存的起始地址，结束
6. 否则分配一个相同大小的小块内存池
7. 将新内存的空闲地址对齐，返回给申请内存的进程，再设置 `last` 指针到空闲内存首地址
8. 从 `current` 指向的小块内存池开始遍历到当前的新内存池，依次将 `failed` 成员 + 1，并使 `current` 指向首个 `failed <= 4` 的小块内存池
9. 否则开始分配大块内存
10. 调用 `ngx_alloc()` 从进程的堆内存中分配
11. 遍历 `large` 链表，找到一个 `alloc` 成员为 `NULL` 的项 (最多找 4 个)
12. 将 `alloc` 成员设置为分配的内存地址并返回，结束
13. 如果没有空闲的 `ngx_pool_large_t` 项，那么从内存池中分配出新的结构体，并添加到 `large` 链表的头部，返回地址，结束
