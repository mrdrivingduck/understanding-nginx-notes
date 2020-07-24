# Chapter 9.1-9.3 - 事件处理框架

Created by : Mr Dk.

2020 / 07 / 20 20:19

Nanjing, Jiangsu, China

---

## 9.1 事件模块的层次定义

事件处理框架负责收集、管理、分发事件。事件以网络事件和定时器事件为主，网络事件中以 TCP 网络事件为主。Nginx 为不同的操作系统和不同的操作系统版本提供了多种 **事件驱动机制** (也叫 **I/O 多路复用**)。

Nginx 定义了核心模块 `ngx_events_module`，使得能够在启动时解析配置项。其次，Nginx 定义了事件模块 `ngx_event_core_module`，决定使用哪种事件管理机制。Nginx 目前定义了 9 个运行在不同 OS、不同 OS 版本上的事件驱动模块，在模块初始化过程中，将选取一个作为事件驱动模块。

正好看一看关于模块层次的抽象。Nginx 首先定义了模块的最抽象接口。这一层模块接口，我觉得可以从三个部分来看：

1. 模块标识 (编号和名称)，不仅包括模块在所有 Nginx 模块里的编号，还包括模块在某类 Nginx 模块里的编号
2. 模块类型、参数、上下文
3. 模块的生命周期函数指针 (`init_...` / `exit_...`)

```c
struct ngx_module_s {
    ngx_uint_t            ctx_index;
    ngx_uint_t            index;

    char                 *name;

    ngx_uint_t            spare0;
    ngx_uint_t            spare1;

    ngx_uint_t            version;
    const char           *signature;

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};
```

其中，与模块层次相关的 field 有 `ctx`、`commands`、`type`。其中，`type` 决定了模块的类型，`commands` 体现了这个模块感兴趣的配置项，而 `ctx` 决定了这类模块使用的通用接口。比如，对于核心模块来说，通用接口是这样的：

```c
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```

这也体现了核心模块的功能：创建、解析配置项。

而事件模块的通用接口：

```c
typedef struct {
    ngx_str_t              *name;

    void                 *(*create_conf)(ngx_cycle_t *cycle);
    char                 *(*init_conf)(ngx_cycle_t *cycle, void *conf);

    ngx_event_actions_t     actions;
} ngx_event_module_t;
```

其中，`actions` 中定义了事件驱动模块的抽象函数，共有 10 个。也就是说，对于不同的 I/O 多路复用机制，都应当以各自的方式实现这 10 个函数：

```c
typedef struct {
    // 将感兴趣的事件想 OS 的事件驱动机制中添加或删除
    ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    // 基本与上两个一致
    ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    // 向事件驱动机制添加或删除连接
    ngx_int_t  (*add_conn)(ngx_connection_t *c);
    ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

    ngx_int_t  (*notify)(ngx_event_handler_pt handler);
    
    // 处理事件，是处理、分发事件的核心
    ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
                                 ngx_uint_t flags);

    // 初始化事件驱动模块的函数
    ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
    // 退出事件驱动模块前调用的函数
    void       (*done)(ngx_cycle_t *cycle);
} ngx_event_actions_t;
```

## 9.2 Nginx 事件的定义

在 Nginx 中，每个事件都由 `ngx_event_t` 结构体来表示，其中绝大部分为标志位：

```c
struct ngx_event_s {
    void            *data;

    unsigned         write:1;

    unsigned         accept:1;

    /* used to detect the stale events in kqueue and epoll */
    unsigned         instance:1;

    /*
     * the event was passed or would be passed to a kernel;
     * in aio mode - operation was posted.
     */
    unsigned         active:1;

    unsigned         disabled:1;

    /* the ready event; in aio mode 0 means that no operation can be posted */
    unsigned         ready:1;

    unsigned         oneshot:1;

    /* aio operation is complete */
    unsigned         complete:1;

    unsigned         eof:1;
    unsigned         error:1;

    unsigned         timedout:1;
    unsigned         timer_set:1;

    unsigned         delayed:1;

    unsigned         deferred_accept:1;

    /* the pending eof reported by kqueue, epoll or in aio chain operation */
    unsigned         pending_eof:1;

    unsigned         posted:1;

    unsigned         closed:1;

    /* to test on worker exit */
    unsigned         channel:1;
    unsigned         resolver:1;

    unsigned         cancelable:1;

#if (NGX_HAVE_KQUEUE)
    unsigned         kq_vnode:1;

    /* the pending errno reported by kqueue */
    int              kq_errno;
#endif

    /*
     * kqueue only:
     *   accept:     number of sockets that wait to be accepted
     *   read:       bytes to read when event is ready
     *               or lowat when event is set with NGX_LOWAT_EVENT flag
     *   write:      available space in buffer when event is ready
     *               or lowat when event is set with NGX_LOWAT_EVENT flag
     *
     * iocp: TODO
     *
     * otherwise:
     *   accept:     1 if accept many, 0 otherwise
     *   read:       bytes to read when event is ready, -1 if not known
     */

    int              available;

    ngx_event_handler_pt  handler;


#if (NGX_HAVE_IOCP)
    ngx_event_ovlp_t ovlp;
#endif

    ngx_uint_t       index;

    ngx_log_t       *log;

    ngx_rbtree_node_t   timer;

    /* the posted queue */
    ngx_queue_t      queue;

#if 0

    /* the threads support */

    /*
     * the event thread context, we store it here
     * if $(CC) does not understand __thread declaration
     * and pthread_getspecific() is too costly
     */

    void            *thr_ctx;

#if (NGX_EVENT_T_PADDING)

    /* event should not cross cache line in SMP */

    uint32_t         padding[NGX_EVENT_T_PADDING];
#endif
#endif
};
```

其中，`data` 指针通常指向 `ngx_connection_t` 连接对象，表示与这个事件相关的对象。另外，`handler` 是一个函数指针，指向这个事件发生时的回调函数，由每个事件消费模块实现。即，每个 Nginx 模块只要处理事件就必然要设置 `handler` 回调。

```c
typedef void (*ngx_event_handler_pt) (ngx_event_t *ev);
```

在 Nginx 中，事件不需要创建。Nginx 在启动时已经在 `ngx_cycle_t` 中分配了所有的读事件和写事件。每一个连接会自动对应一个读事件和一个写事件 (三个数组大小相同，依靠下标直接对应)。因此只需要从 **连接池** 中拿到一个空闲连接，就能够拿到事件。最后将事件添加到 EPOLL 等事件驱动模块中即可。一旦该事件对应的 TCP 连接上出现可读或可写事件，就会回调 `handler` 指针指向的函数。

> 将事件添加到事件驱动模块中，或从模块中删除时，一般不建议使用 `ngx_event_actions_t` 中定义的 `add()` 或 `del()`，因为这些函数与具体的事件驱动模块强相关。使用 Nginx 封装的 `ngx_handle_read_event()` 和 `ngx_handle_write_event()` 可以屏蔽事件驱动模块之间的差异。

## 9.3 Nginx 连接的定义

Nginx 中定义 `ngx_connection_t` 来表示连接。连接默认指的是由客户端发起，由 Nginx 接受的连接 (被动连接)。另外 Nginx 还定义了 `ngx_peer_connection_t` 表示 Nginx 到上游服务器的连接 (主动连接)。主动连接的结构体时基于被动连接实现的，复用了其中的很多 field。这两种连接都不可以随意创建，必须从连接池中获取。

### 9.3.1 被动连接

```c
struct ngx_connection_s {
    // 连接未使用时，用于充当连接池中的空闲链表指针；连接使用时，具体含义由模块决定
    void               *data;
    
    // 读写事件
    ngx_event_t        *read;
    ngx_event_t        *write;

    // socket 句柄
    ngx_socket_t        fd;

    ngx_recv_pt         recv; // 接收网络字符的函数指针
    ngx_send_pt         send; // 发送网络字符的函数指针
    ngx_recv_chain_pt   recv_chain;
    ngx_send_chain_pt   send_chain;

    ngx_listening_t    *listening; // 由 listening 监听端口的事件建立

    off_t               sent; // 该连接上已经发送的字节数

    ngx_log_t          *log; // 日志对象

    ngx_pool_t         *pool; // 内存池

    int                 type;

    struct sockaddr    *sockaddr; // 连接客户端的 sockaddr 结构体
    socklen_t           socklen; // 结构体长度
    ngx_str_t           addr_text; // 客户端字符串形式的 IP 地址

    ngx_proxy_protocol_t  *proxy_protocol;

#if (NGX_SSL || NGX_COMPAT)
    ngx_ssl_connection_t  *ssl;
#endif

    ngx_udp_connection_t  *udp;

    struct sockaddr    *local_sockaddr; // 本机监听端口的 sockaddr 结构体
    socklen_t           local_socklen; // 结构体长度

    ngx_buf_t          *buffer; // 用于接收、缓存客户端发来的字符流

    // 将当前连接添加到 ngx_cycle_t 结构体的可重用连接双向链表中
    ngx_queue_t         queue;

    // 连接使用次数
    ngx_atomic_uint_t   number;

    // 处理的请求次数
    ngx_uint_t          requests;

    // 缓存中的业务类型
    unsigned            buffered:8;

    // 记录日志时的级别
    unsigned            log_error:3;     /* ngx_connection_log_error_e */

    // 连接超时 / 错误 / 销毁
    unsigned            timedout:1;
    unsigned            error:1;
    unsigned            destroyed:1;

    // 连接空闲 / 可重用 / 关闭
    unsigned            idle:1;
    unsigned            reusable:1;
    unsigned            close:1;
    unsigned            shared:1;

    unsigned            sendfile:1; // 正在发送文件
    unsigned            sndlowat:1; // 分发时间时，发送缓冲区的大小阈值
    unsigned            tcp_nodelay:2;   /* ngx_connection_tcp_nodelay_e */
    unsigned            tcp_nopush:2;    /* ngx_connection_tcp_nopush_e */

    unsigned            need_last_buf:1;

#if (NGX_HAVE_AIO_SENDFILE || NGX_COMPAT)
    unsigned            busy_count:2; // 使用异步 I/O
#endif

#if (NGX_THREADS || NGX_COMPAT)
    ngx_thread_task_t  *sendfile_task;
#endif
};
```

### 9.3.2 主动连接

Nginx 主动向其它服务器发起的连接。

```c
typedef struct ngx_peer_connection_s  ngx_peer_connection_t;

// 从连接池中获取一个新连接
typedef ngx_int_t (*ngx_event_get_peer_pt) (ngx_peer_connection_t *pc, void *data);
// 将使用完毕的连接归还给连接池
typedef void (*ngx_event_free_peer_pt) (ngx_peer_connection_t *pc, void *data, ngx_uint_t state);

struct ngx_peer_connection_s {
    ngx_connection_t                *connection; // 重用其中部分成员

    struct sockaddr                 *sockaddr; // 远程服务器的 socket 地址
    socklen_t                        socklen; // socket 长度
    ngx_str_t                       *name; // 远端服务器的名称

    ngx_uint_t                       tries; // 连接失败时的最大重复次数
    ngx_msec_t                       start_time;

    ngx_event_get_peer_pt            get; // 获取连接的函数
    ngx_event_free_peer_pt           free; // 释放连接的函数
    ngx_event_notify_peer_pt         notify;
    void                            *data; // 函数参数

#if (NGX_SSL || NGX_COMPAT)
    ngx_event_set_peer_session_pt    set_session;
    ngx_event_save_peer_session_pt   save_session;
#endif

    ngx_addr_t                      *local; // 本机地址信息

    int                              type;
    int                              rcvbuf; // socket 接收缓冲区大小

    ngx_log_t                       *log; // 日志对象

    // 标志位
    unsigned                         cached:1; // 是否缓存
    unsigned                         transparent:1;
    unsigned                         so_keepalive:1;
    unsigned                         down:1;

                                     /* ngx_connection_log_error_e */
    unsigned                         log_error:2;

    NGX_COMPAT_BEGIN(2)
    NGX_COMPAT_END
};
```

这个结构体在每次使用时被创建，但是其中的 `ngx_connection_t` 连接还是从连接池中获取。

### 9.3.3 `ngx_connection_t` 连接池

`ngx_connection_t` 结构体是 Nginx 在启动阶段就预分配好的，在接收客户端连接时，直接从连接池中获取即可。在 `ngx_cycle_t` 核心结构体中，以下两个成员共同构成连接池：

* `connections` 指向整个连接池数组的头部
* `free_connections` 指向第一个空闲的 `ngx_connection_t`，这些结构体以 `data` 指针串联为单链表

Nginx 认定每个连接至少需要一个读事件和一个写事件，因此有多少个连接就分配多少个读写事件。连接、读事件、写事件是三个大小相同的数组，只需要通过同一个 index 就能拿到三个对应的结构体。

---

