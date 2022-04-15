# Chapter 9.4-9.6 - 事件驱动模块与 EPOLL

Created by : Mr Dk.

2020 / 07 / 24 10:44

Nanjing, Jiangsu, China

---

## 9.4 事件核心模块

`ngx_events_module` 是一个 **核心模块**。其中，定义了新的 **事件模块**，以及所有事件模块都要实现的 `ngx_event_module_t` 通用接口。回顾核心模块的接口定义：

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

为了定义一个新的核心模块，要满足三个要素：

- 模块类型 `type`
- 模块感兴趣的配置项 `commands`
- 实现核心模块的通用接口 `ngx_core_module_t` 于 `ctx` 指针中

那么，`ngx_events_module` 核心模块感兴趣的配置项有哪些呢？如下所示，该模块只对 `events{...}` 配置项感兴趣。

```c
static ngx_command_t ngx_events_commandsp[] = {
    {
        ngx_string("events"),
        NGX_MAIN_CONF | NGX_CONF_BLOCK | NGX_CONF_NOARGS,
        ngx_events_block, // 解析配置项的回调
        0,
        0,
        NULL
    },

    ngx_null_command
};
```

作为核心模块，`ngx_events_module` 需要实现核心模块的通用接口。其中只是定义了模块名称，并没有实现 `create_conf()` 和 `init_conf()` - 因为这个模块不解析具体的配置项参数，由各个事件模块实现这两个函数：

```c
static ngx_core_module_t ngx_events_module_ctx = {
    ngx_string("events"),
    NULL,
    NULL
};
```

最终，`ngx_events_module` 核心模块的定义如下：

```c
ngx_module_t ngx_events_module = {
    NGX_MODULE_V1,
    &ngx_events_module_ctx, // 核心模块通用接口
    ngx_events_commands, // 模块感兴趣的配置项
    NGX_CORE_MODULE, // 模块类型
    NULL, // init master
    NULL, // init module
    NULL, // init process
    NULL, // init thread
    NULL, // exit thread
    NULL, // exit process
    NULL, // exit master
    NGX_MODULE_V1_PADDING
}
```

### 9.4.1 管理所有事件模块的配置项

每个事件模块都必须实现 `ngx_event_module_t` 通用接口，其中的 `create_conf()` 函数允许每个事件模块自己分配内存建立自己的配置项结构体。但各个模块的配置项结构体的内存指针如何由 `ngx_events_module` 模块管理呢？每一个事件模块的配置结构体都会被存放到 `ngx_events_module` 模块的指针数组中。这个指针数组又会被存放到 `ngx_cycle_t` 核心结构体的的 `conf_ctx` 成员中：(四级指针哦 😪)

```c
void ****conf_ctx;
```

也就是说，在 `ngx_cycle_t` 核心结构体中，可以看到每一个核心模块下属的每一个扩展模块中的每一个配置项。每个事件模块通过如下的宏就可以从 `conf_ctx` 中获得当前模块建立的配置项结构体：

```c
#define ngx_event_get_conf(conf_ctx, module) \
    (*(ngx_get_conf(conf_ctx, ngx_events_module))) [module.ctx_index]
#define ngx_get_conf(conf_ctx, module) conf_ctx[module.index]
```

### 9.4.2 管理事件模块

所有配置项的保存都在 `ngx_events_block()` 函数中进行。首先会初始化每个事件模块的 `ctx_index` 成员。这里注意每个模块的 `index` 指的是该模块在所有模块中的序号，而 `ctx_index` 是指该模块在同类型模块中的序号。

所有模块编号确定，个数也就确定了，那么分配配置项指针数组。然后对每一个事件模块，依次调用事件模块通用接口中的 `create_conf()` 函数，产生的结构体指针保存在上述指针数组中。配置项解析完毕后，依次调用每一个事件模块的 `init_conf()` 函数。

---

## 9.5 核心事件模块

> `ngx_events_module` - 事件核心模块 - 是一个核心模块，但定义了事件模块的信息
>
> `ngx_event_core_module` - 核心事件模块 - 是一个事件模块，但是是最核心的事件模块

`ngx_event_core_module` 是所有事件模块中排第一位的模块。它负责创建连接池，并决定使用哪种事件驱动机制，并初始化相应的事件驱动机制模块。该模块对以下配置项感兴趣：

```c
static ngx_command_t  ngx_event_core_commands[] = {

    // 连接池大小 (每个 worker 进程的最大连接数)
    { ngx_string("worker_connections"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_connections,
      0,
      0,
      NULL },

    // 选择哪一个事件驱动模块
    { ngx_string("use"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_use,
      0,
      0,
      NULL },

    // 尽可能多地接收连接
    { ngx_string("multi_accept"),
      NGX_EVENT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_event_conf_t, multi_accept),
      NULL },

    // 是否启用负载均衡锁
    { ngx_string("accept_mutex"),
      NGX_EVENT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_event_conf_t, accept_mutex),
      NULL },

    // 启用负载均衡锁后，在试图处理新连接之前，延迟的毫秒数
    { ngx_string("accept_mutex_delay"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_msec_slot,
      0,
      offsetof(ngx_event_conf_t, accept_mutex_delay),
      NULL },

    // 对指定 IP 的 TCP 连接打印 debug 级别的调试日志
    { ngx_string("debug_connection"),
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_event_debug_connection,
      0,
      0,
      NULL },

      ngx_null_command
};
```

在该模块中，定义了存储上述所有配置的结构体 (两者对应)：

```c
typedef struct {
    ngx_uint_t    connections; // 连接池大小
    ngx_uint_t    use; // 选用的事件驱动模块在所有事件模块中的序号

    ngx_flag_t    multi_accept; // 标志位，是否尽可能多地建立连接
    ngx_flag_t    accept_mutex; // 标志位，是否启动负载均衡锁

    ngx_msec_t    accept_mutex_delay;

    u_char       *name; // 使用事件驱动模块的名称

#if (NGX_DEBUG)
    ngx_array_t   debug_connection;
#endif
} ngx_event_conf_t;
```

作为一个 **事件模块**，`ngx_event_core_module` 也要实现事件模块的通用接口 `ngx_event_module_t`。

```c
static ngx_str_t  event_core_name = ngx_string("event_core");

static ngx_event_module_t  ngx_event_core_module_ctx = {
    &event_core_name, // "event_core"
    ngx_event_core_create_conf,            /* create configuration */
    ngx_event_core_init_conf,              /* init configuration */

    // 并不实现 ngx_event_actions_t，因为这个模块不真正负责网络事件驱动
    { NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL }
};
```

最后将这个事件模块接口上下文并入整体模块信息中：

```c
ngx_module_t  ngx_event_core_module = {
    NGX_MODULE_V1,
    &ngx_event_core_module_ctx,            /* module context */ // 事件模块通用接口
    ngx_event_core_commands,               /* module directives */ // 该模块感兴趣的配置项
    NGX_EVENT_MODULE,                      /* module type */ // 模块类型为事件模块
    NULL,                                  /* init master */
    ngx_event_module_init,                 /* init module */
    ngx_event_process_init,                /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

在 Nginx 启动过程中，还没有 `fork()` 出 worker 子进程时，调用 `ngx_event_module_init()` 进行一些初始化工作；在 worker 子进程生成后，每个子进程都会调用 `ngx_event_process_init()` 函数后进入工作循环。`ngx_event_process_init()` 函数干了很多事情：

1. 确认 Nginx 使用了 master 模式，且 worker 进程数量大于 1，打开负载均衡锁
2. 如果不满足条件，则关闭负载均衡锁 (标志位)
3. 初始化红黑树实现的定时器
4. 在指定使用的事件驱动模块中，调用事件模块通用接口中的 `init()` 进行这个模块的初始化
5. 设置更新系统事件的回调
6. 如果使用 EPOLL，则预分配句柄
7. 预分配 `ngx_conntection_t` 数组作为连接池
8. 预分配 `ngx_event_t` 数组作为读事件池
9. 预分配 `ngx_event_t` 数组作为写事件池
10. 将读写事件设置到对应的连接池中
11. 设置 `ngx_cycle_t` 中的空闲连接链表
12. 为所有 `ngx_listening_t` 对象中的 `connection` 成员分配连接，设置读事件处理函数为 `ngx_event_accept()`
13. 将所有 `ngx_listening_t` 对象的读事件添加到事件驱动模块中 (比如加入 EPOLL 文件系统)

---

## 9.6 EPOLL 事件驱动模块

### 9.6.1 EPOLL 的原理

EPOLL 适用的典型场景：大量用户 (1M) 与一个进程保持着 TCP 连接，每个时刻只有几百个 TCP 连接是活跃的 (可以不阻塞读取数据)。那么进程要将这 1M 个连接告诉 OS，由 OS 负责查询哪些连接是活跃的。在 Linux kernel 2.4 之前，SELECT 和 POLL 两个事件驱动方式都是这么做的。

由于这 1M 个连接大部分都是不活跃的，因此每次将 1M 个连接通过系统调用传递给 OS 非常低效！EPOLL 在内核中申请了一个简易的文件系统。每次只需创建一个 EPOLL 对象，初始化一个文件系统，然后向文件系统中添加或删除连接即可。EPOLL 的效率非常高，因为：

- 调用 EPOLL 时不再需要传递 1M 个连接
- 内核也不需要全量遍历所有连接来找出其中的活跃连接

进程调用 `epoll_create()` 函数时，内核会创建一个 `eventpoll` 结构体。其中两个数据结构非常重要：

```c
struct eventpoll {
    struct rb_root rbr; // 红黑树根结点，红黑树中保存了所有监控的事件
    struct list_head rdllist; // 将要返回给用户的 (满足条件的) 事件
};
```

这个结构体开辟于内核中，当用户使用 `epoll_ctl()` 函数向该结构体中添加事件时，都会被加入到结构体的红黑树中。因此增删改的效率很高，重复添加的事件也能被快速识别。所有添加的事件都会与 **设备驱动程序** (如网卡驱动程序) 建立回调关系，在相应事件发生时触发回调。在回调中，会把相应事件放到 `rdllist` 链表中。在用户调用 `epoll_wait()` 查询事件时，只需要检查 `rdllist` 链表上是否存在事件，并将链表上的事件复制到用户态内存中。EPOLL 可以轻易处理百万级别的并发连接。

### 9.6.3 `ngx_epoll_module` 模块的实现

EPOLL 模块感兴趣的配置项：

```c
static ngx_command_t  ngx_epoll_commands[] = {

    { ngx_string("epoll_events"), // epoll_wait() 时最多可以返回的事件数
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      0,
      offsetof(ngx_epoll_conf_t, events),
      NULL },

    { ngx_string("worker_aio_requests"), // 初始化分配的异步 I/O 事件个数
      NGX_EVENT_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      0,
      offsetof(ngx_epoll_conf_t, aio_requests),
      NULL },

      ngx_null_command
};
```

模块中对应的存储配置项的结构体：

```c
typedef struct {
    ngx_uint_t  events;
    ngx_uint_t  aio_requests;
} ngx_epoll_conf_t;
```

作为一个事件模块，这个模块自然需要实现事件模块的通用接口：

```c
static ngx_str_t      epoll_name = ngx_string("epoll");

static ngx_event_module_t  ngx_epoll_module_ctx = {
    &epoll_name,
    ngx_epoll_create_conf,               /* create configuration */
    ngx_epoll_init_conf,                 /* init configuration */

    {
        ngx_epoll_add_event,             /* add an event */
        ngx_epoll_del_event,             /* delete an event */
        ngx_epoll_add_event,             /* enable an event */
        ngx_epoll_del_event,             /* disable an event */
        ngx_epoll_add_connection,        /* add an connection */
        ngx_epoll_del_connection,        /* delete an connection */
#if (NGX_HAVE_EVENTFD)
        ngx_epoll_notify,                /* trigger a notify */
#else
        NULL,                            /* trigger a notify */
#endif
        ngx_epoll_process_events,        /* process the events */
        ngx_epoll_init,                  /* init the events */
        ngx_epoll_done,                  /* done the events */
    }
};
```

其中，`ngx_epoll_create_conf()` 和 `ngx_epoll_init_conf()` 仅为了解析配置项。`ngx_epoll_init()` 函数主要会做两件事：

1. 调用 `epoll_create()` 创建在内核中创建 EPOLL 结构体
2. 创建 `event_list` 数组，用于 `epoll_wait()` 时存放就绪的事件

```c
static ngx_int_t
ngx_epoll_init(ngx_cycle_t *cycle, ngx_msec_t timer)
{
    ngx_epoll_conf_t  *epcf;

    // 取得已经解析完毕的配置项
    epcf = ngx_event_get_conf(cycle->conf_ctx, ngx_epoll_module);

    if (ep == -1) {
        // 调用 epoll_create()
        ep = epoll_create(cycle->connection_n / 2);

        if (ep == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "epoll_create() failed");
            return NGX_ERROR;
        }

#if (NGX_HAVE_EVENTFD)
        if (ngx_epoll_notify_init(cycle->log) != NGX_OK) {
            ngx_epoll_module_ctx.actions.notify = NULL;
        }
#endif

#if (NGX_HAVE_FILE_AIO)
        ngx_epoll_aio_init(cycle, epcf);
#endif

#if (NGX_HAVE_EPOLLRDHUP)
        ngx_epoll_test_rdhup(cycle);
#endif
    }

    if (nevents < epcf->events) {
        if (event_list) {
            ngx_free(event_list);
        }

        // 根据配置项中的最大返回事件数，分配 event_list 数组
        event_list = ngx_alloc(sizeof(struct epoll_event) * epcf->events,
                               cycle->log);
        if (event_list == NULL) {
            return NGX_ERROR;
        }
    }

    nevents = epcf->events;

    ngx_io = ngx_os_io;

    ngx_event_actions = ngx_epoll_module_ctx.actions;

#if (NGX_HAVE_CLEAR_EVENT)
    ngx_event_flags = NGX_USE_CLEAR_EVENT
#else
    ngx_event_flags = NGX_USE_LEVEL_EVENT
#endif
                      |NGX_USE_GREEDY_EVENT
                      |NGX_USE_EPOLL_EVENT;

    return NGX_OK;
}
```

从通用接口的角度来看，EPOLL 不存在 enable 一个事件或 disable 一个事件。因此 enable 和 add 接口都是由 `ngx_epoll_add_event()` 实现的，disable 和 del 接口都是由 `ngx_epoll_del_event()` 实现的。这两个函数分别用于向内核中注册或解除注册事件。

```c
static ngx_int_t
ngx_epoll_add_event(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags)
{
    int                  op;
    uint32_t             events, prev;
    ngx_event_t         *e;
    ngx_connection_t    *c;
    struct epoll_event   ee;

    c = ev->data;

    events = (uint32_t) event;

    if (event == NGX_READ_EVENT) {
        e = c->write;
        prev = EPOLLOUT;
#if (NGX_READ_EVENT != EPOLLIN|EPOLLRDHUP)
        events = EPOLLIN|EPOLLRDHUP; // 读事件的触发标志
#endif

    } else {
        e = c->read;
        prev = EPOLLIN|EPOLLRDHUP;
#if (NGX_WRITE_EVENT != EPOLLOUT)
        events = EPOLLOUT; // 写事件的触发标志
#endif
    }

    // 确定到底是添加事件还是修改事件
    if (e->active) {
        op = EPOLL_CTL_MOD;
        events |= prev;

    } else {
        op = EPOLL_CTL_ADD;
    }

#if (NGX_HAVE_EPOLLEXCLUSIVE && NGX_HAVE_EPOLLRDHUP)
    if (flags & NGX_EXCLUSIVE_EVENT) {
        events &= ~EPOLLRDHUP;
    }
#endif

    ee.events = events | (uint32_t) flags;
    ee.data.ptr = (void *) ((uintptr_t) c | ev->instance);

    ngx_log_debug3(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                   "epoll add event: fd:%d op:%d ev:%08XD",
                   c->fd, op, ee.events);

    // 调用 epoll_ctl 向内核中添加事件或修改事件
    if (epoll_ctl(ep, op, c->fd, &ee) == -1) {
        ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_errno,
                      "epoll_ctl(%d, %d) failed", op, c->fd);
        return NGX_ERROR;
    }

    // 当前事件活跃
    ev->active = 1;
#if 0
    ev->oneshot = (flags & NGX_ONESHOT_EVENT) ? 1 : 0;
#endif

    return NGX_OK;
}
```

另外，`ngx_epoll_add_connection()` 和 `ngx_epoll_del_connection()` 也与上述函数类似。上述函数只是单独添加某个连接的读或写事件，而 `ngx_epoll_add_connection()` 是添加整个连接 (即同时添加读和写事件)，具体实现方式类似。

`ngx_epoll_process_events()` 函数实现了收集、分发事件的接口：

```c
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    int                events;
    uint32_t           revents;
    ngx_int_t          instance, i;
    ngx_uint_t         level;
    ngx_err_t          err;
    ngx_event_t       *rev, *wev;
    ngx_queue_t       *queue;
    ngx_connection_t  *c;

    /* NGX_TIMER_INFINITE == INFTIM */

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "epoll timer: %M", timer);

    // 调用 epoll_wait() 从内核中收集就绪事件
    events = epoll_wait(ep, event_list, (int) nevents, timer);

    err = (events == -1) ? ngx_errno : 0;

    // 更新时间
    if (flags & NGX_UPDATE_TIME || ngx_event_timer_alarm) {
        ngx_time_update();
    }

    if (err) {
        if (err == NGX_EINTR) {

            if (ngx_event_timer_alarm) {
                ngx_event_timer_alarm = 0;
                return NGX_OK;
            }

            level = NGX_LOG_INFO;

        } else {
            level = NGX_LOG_ALERT;
        }

        ngx_log_error(level, cycle->log, err, "epoll_wait() failed");
        return NGX_ERROR;
    }

    // 没有就绪事件
    if (events == 0) {
        if (timer != NGX_TIMER_INFINITE) {
            return NGX_OK;
        }

        ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                      "epoll_wait() returned no events without timeout");
        return NGX_ERROR;
    }

    // 开始遍历内核返回的所有事件
    for (i = 0; i < events; i++) {
        c = event_list[i].data.ptr;

        instance = (uintptr_t) c & 1;
        c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);

        // 读事件
        rev = c->read;

        // 读事件是否过期
        if (c->fd == -1 || rev->instance != instance) {

            /*
             * the stale event from a file descriptor
             * that was just closed in this iteration
             */

            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll: stale event %p", c);
            continue;
        }

        // 获得事件类型
        revents = event_list[i].events;

        ngx_log_debug3(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "epoll: fd:%d ev:%04XD d:%p",
                       c->fd, revents, event_list[i].data.ptr);

        if (revents & (EPOLLERR|EPOLLHUP)) {
            ngx_log_debug2(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll_wait() error on fd:%d ev:%04XD",
                           c->fd, revents);

            /*
             * if the error events were returned, add EPOLLIN and EPOLLOUT
             * to handle the events at least in one active handler
             */

            revents |= EPOLLIN|EPOLLOUT;
        }

#if 0
        if (revents & ~(EPOLLIN|EPOLLOUT|EPOLLERR|EPOLLHUP)) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                          "strange epoll_wait() events fd:%d ev:%04XD",
                          c->fd, revents);
        }
#endif

        // 事件是读事件，而且活跃
        if ((revents & EPOLLIN) && rev->active) {

#if (NGX_HAVE_EPOLLRDHUP)
            if (revents & EPOLLRDHUP) {
                rev->pending_eof = 1;
            }
#endif

            rev->ready = 1;
            rev->available = -1;

            if (flags & NGX_POST_EVENTS) {
                // 延后处理事件
                queue = rev->accept ? &ngx_posted_accept_events
                                    : &ngx_posted_events;

                ngx_post_event(rev, queue);

            } else {
                // 立刻调用读事件的回调函数处理事件
                rev->handler(rev);
            }
        }

        // 写事件
        wev = c->write;

        if ((revents & EPOLLOUT) && wev->active) {

            // 写事件过期
            if (c->fd == -1 || wev->instance != instance) {

                /*
                 * the stale event from a file descriptor
                 * that was just closed in this iteration
                 */

                ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                               "epoll: stale event %p", c);
                continue;
            }

            wev->ready = 1;
#if (NGX_THREADS)
            wev->complete = 1;
#endif

            if (flags & NGX_POST_EVENTS) {
                // 延后处理写事件
                ngx_post_event(wev, &ngx_posted_events);

            } else {
                // 调用写事件的回调函数处理事件
                wev->handler(wev);
            }
        }
    }

    return NGX_OK;
}
```

可以看到，上述函数向内核收集了当前已就绪的所有事件，对于不需要延后处理的事件，立刻调用其回调函数。这里相当于实现了 **事件分发** 的工作。因此每一个回调函数不能导致进程休眠或占用太多事件，否则将不能及时处理其它事件。

什么是过期事件？比如在处理第一个事件时，由于做了一些操作导致第三个事件无效了。为了处理这个问题，需要通过 `instance` 标志位来标记是否过期。当连接每次从连接池中取出时，`instance` 标志位将会被取反。

`ngx_epoll_done()` 函数在 Nginx 退出服务时会被调用。主要工作：

1. 关闭 EPOLL 描述符 (回收内核中的 EPOLL 内存)
2. 释放 `event_list` 数组

以上，是 `ngx_epoll_module` 模块的事件模块上下文中的所有的函数实现。由此，整个模块的定义就显而易见了：

```c
ngx_module_t  ngx_epoll_module = {
    NGX_MODULE_V1,
    &ngx_epoll_module_ctx,               /* module context */
    ngx_epoll_commands,                  /* module directives */
    NGX_EVENT_MODULE,                    /* module type */
    NULL,                                /* init master */
    NULL,                                /* init module */
    NULL,                                /* init process */
    NULL,                                /* init thread */
    NULL,                                /* exit thread */
    NULL,                                /* exit process */
    NULL,                                /* exit master */
    NGX_MODULE_V1_PADDING
};
```
