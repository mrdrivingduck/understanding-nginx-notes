# Chapter 9.7-9.8 - 定时器事件与事件驱动框架处理流程

Created by : Mr Dk.

2020 / 07 / 27 20:19

Nanjing, Jiangsu, China

---

## 9.7 定时器事件

网络事件是由内核触发的，而定时器事件完全由 Nginx 自身实现，与内核无关。

### 9.7.1 缓存事件的管理

Nginx 中每个进程都会单独地管理当前事件，其使用的事件会被缓存在内存中。具体缓存的形式如下 (精确到毫秒级)：

```c
typedef struct {
    time_t sec; // 1970.01.01 00:00:00 到现在的秒数
    ngx_uint_t msec; // 当前事件相对于 sec 的毫秒偏移量
    ngx_int_t gmtoff; // 时区
} ngx_time_t;
```

作为 Web 服务器，Nginx 还定义了一些全局变量，用于得到可读性较强的规范时间字符串：

```c
volatile ngx_msec_t      ngx_current_msec; // 毫秒级别的 UNIX 时间戳
volatile ngx_time_t     *ngx_cached_time; // 当前时间的结构体指针
volatile ngx_str_t       ngx_cached_err_log_time; // 用于记录 error log 的当前时间字符串
volatile ngx_str_t       ngx_cached_http_time; // 用于 HTTP 相关的当前时间字符串
volatile ngx_str_t       ngx_cached_http_log_time; // 用于记录 HTTP 日志的当前时间字符串
volatile ngx_str_t       ngx_cached_http_log_iso8601; // ISO 8601 格式下的字符串形式的标准时间
volatile ngx_str_t       ngx_cached_syslog_time; // 用于系统日志的时间字符串
```

缓存时间的更新时机：在 Nginx 启动时更新一次时间，之后更新时间的操作只能由 `ngx_epoll_process_events()` 执行。当标志位参数中有 `NGX_UPDATE_TIME` 标志时，就会调用 `ngx_time_update()` 更新缓存时间。

### 9.7.2 缓存时间的精度

根据缓存时间的更新策略，缓存时间的精度与 `ngx_epoll_process_events()` 函数的调用频率有关。

Nginx 自身还另外提供了设置更新缓存时间频率的功能。`ngx_event_core_module` 模块在初始化时，会向内核注册定时器，使内核定期调用一次 `ngx_timer_signal_handler()` 函数 - 在该函数中，会将 `ngx_event_timer_alarm` 标志位设置为 `1`，由此之后一旦调用了 `ngx_epoll_process_events()` 函数，一定会更新缓存时间。

如果 `ngx_epoll_process_events()` 很长时间不会被执行呢？在 EPOLL 模块下，会使得 `epoll_wait()` 的调用立刻返回，而不是等待一段时间，由此来控制时间精度。

### 9.7.3 定时器的实现

定时器由 **红黑树** 实现，树结点关键字是事件的超时时间，整棵树按超时时间的大小组织。需要找出最有可能超时的事件，只需要把树中最左边的结点取出来，并与当前事件对比即可。

---

## 9.8 事件驱动框架的处理流程

Nginx 会将监听连接的读事件回调函数设置为 `ngx_event_accept()`，并把读事件添加到事件驱动模块中。当执行 `ngx_epoll_process_events()` 函数时，就会调用回调函数建立新连接。Nginx 为了充分发挥多核 CPU 架构性能的考虑，使用了多个 worker 子进程监听相同的端口，由此会引发子进程在 `accept()` 连接时发生 _惊群_ (争抢) 问题。另外，建立连接时话要考虑负载均衡问题，使得各个子进程之前的负载尽量均衡，尽量独占一个 CPU 核心。

Nginx 的 post 事件处理机制用于解决这两个问题。Nginx 中设计了两个队列：

- `ngx_posted_accept_events` 队列存放监听连接的读事件
- `ngx_posted_events` 队列存放普通读写事件

由 `epoll_wait()` 查询得到的所有事件按类型分发到这两个队列中，然后优先执行监听连接的读时间，再处理普通读写事件。

### 9.8.1 如何建立新连接

建立新连接的回调函数 `ngx_event_accept()`：

1. 调用 `accept()` 函数试图建立连接，如果没有准备好新连接，则整个函数直接返回
2. 设置负载均衡阈值
3. 调用 `ngx_get_connection()` 函数从连接池中获取一个 `ngx_connection_t` 连接对象
4. 为 `ngx_connection_t` 中的 `pool` 指针建立内存池
5. 设置 socket 属性 (为非阻塞)
6. 将新连接的读事件添加到 EPOLL 等事件驱动模块中
7. 调用 `ngx_listening_t` 监听对象的 `handler` 回调函数

### 9.8.2 如何解决 _惊群_ 问题

某一时刻，恰好所有的 worker 子进程都在休眠并等待 `epoll_wait()` 时，一个客户端与服务器建立了新连接。此时，内核会激活所有休眠的 worker 子进程，而实际上最终只有一个子进程能够成功建立连接，其余进程的唤醒都是没有必要的，白白切换了上下文。很多 OS 已经从内核的层面上解决这一问题。但作为一个可移植的 Web 服务器，Nginx 从自身应用的层面上解决了问题：规定同一时刻只能有唯一一个 worker 子进程监听端口。那么如何限制某一时刻只能有一个进程监听端口呢？

只有在试图获得 `accept_mutex` 锁后，当前 worker 进程才会去监听端口。获取锁的过程不会阻塞，如果没有获取到锁，会立刻返回：

```c
ngx_int_t
ngx_trylock_accept_mutex(ngx_cycle_t *cycle)
{
    // 试图得到锁，ngx_accept_mutex 是进程间的同步锁
    if (ngx_shmtx_trylock(&ngx_accept_mutex)) {

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "accept mutex locked");

        // 进程已经获取到锁了，直接返回
        if (ngx_accept_mutex_held && ngx_accept_events == 0) {
            return NGX_OK;
        }

        // 将所有监听连接的读事件加入事件驱动模块中
        // 如果失败，那么就释放锁
        if (ngx_enable_accept_events(cycle) == NGX_ERROR) {
            ngx_shmtx_unlock(&ngx_accept_mutex);
            return NGX_ERROR;
        }

        // 设置已经获得锁的标志，通知本进程其它模块
        ngx_accept_events = 0;
        ngx_accept_mutex_held = 1;

        return NGX_OK;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "accept mutex lock failed: %ui", ngx_accept_mutex_held);

    if (ngx_accept_mutex_held) {
        if (ngx_disable_accept_events(cycle, 0) == NGX_ERROR) {
            return NGX_ERROR;
        }

        ngx_accept_mutex_held = 0;
    }

    return NGX_OK;
}
```

接下来，如果进程成功获取到了锁，那么就既要处理已有连接的读写事件，又要处理监听端口上的新连接事件；如果进程没有获得锁，那么就只处理已有连接的读写事件。那么 **何时释放** 这个锁呢？如果该进程有很多活跃连接，在处理这些连接时占用了很长时间，即很长时间不释放 `ngx_accept_mutex` 锁，那么其它 worker 进程就没有机会处理新连接，不利于负载均衡。

Nginx 使用两个延后处理队列解决这个问题。如果当前进程获得了锁，那么该进程的 `NGX_POST_EVENTS` 标志就会被设置。在事件的分发处理中，如果该标志被设置，那么就将事件放入延后处理队列中，回调函数不会立刻执行：

```c
if (flags & NGX_POST_EVENTS) {
    queue = rev->accept ? &ngx_posted_accept_events
                        : &ngx_posted_events;

    ngx_post_event(rev, queue);

} else {
    rev->handler(rev);
}
```

这里也相当于顺便给这些事件分了个类。将新连接事件全部放到 `ngx_posted_accept_events` 队列中，将普通事件放到 `ngx_posted_events` 队列中。接下来首先处理所有新连接事件，处理完后就 **立刻释放** `ngx_accept_mutex` 锁，再接着处理 `ngx_posted_events` 队列中的事件。

### 9.8.3 如何实现负载均衡

与 _惊群问题_ 类似，只有打开了 `accept_mutex` 锁才能实现进程间的负载均衡。每个 worker 进程中都有一个整型全局变量 `ngx_accept_disabled`，是负载均衡机制实现的关键阈值。这个值与进程连接池中连接的使用情况密切相关：

```c
ngx_accept_disabled = ngx_cycle->connection_n / 8
                              - ngx_cycle->free_connection_n;
```

可以看到，这个值被初始化为一个负数，绝对值为总连接数的 7/8。当这个阈值是负数时，不会触发负载均衡操作，因为进程的连接池还很充裕；而当 `ngx_accept_disabled` 是正数时，就会触发负载均衡操作：

- 当前进程不再处理新连接事件
- 将 `ngx_accept_disable` 的值减 1

也就是说，当进程的连接数达到连接池的 7/8 时，就不再处理新连接了，而是逐步减小阈值，直到 `ngx_accept_disable` 降到总连接数的 7/8 以下时，才会调用 `ngx_trylock_accept_mutex()` 试图处理新连接事件。总结来看，只有在某个 worker 子进程达到连接池使用达到 7/8 时，才会触发负载均衡，是这个 worker 进程减少处理新连接的机会。避免了某个 worker 进程因连接池耗尽而拒绝服务，而其它进程可处理的连接远未到达上限的问题。

### 9.8.4 POST 事件队列

```c
ngx_queue_t  ngx_posted_accept_events;
ngx_queue_t  ngx_posted_next_events;
ngx_queue_t  ngx_posted_events;
```

### 9.8.5 `ngx_process_events_and_timers` 流程

每个 worker 进程都在 `ngx_worker_process_cycle()` 函数中循环处理事件。其中，处理分发事件是通过 `ngx_process_events_and_timers()` 函数实现的 - 循环调用这个函数就是在循环处理事件。这个函数中，既处理网络事件，又处理定时器事件。总结为：

1. 调用事件驱动模块的 `process_events()` 函数处理网络事件
2. 处理两个延后处理队列中的事件
3. 处理定时器事件

其中，在第一步的 `process_events()` 中，参数会传入 `timer` 和 `flags`，决定了时间精度和事件是否被延后处理。

1. 如果配置文件使用了 `timer_resolution` 配置，那么 `timer` 参数设置为 `-1`，在检测事件 (`epoll_wait()`) 时不等待直接返回
2. 如果没有使用 `timer_resolution`，那么调用 `ngx_event_find_timer()` 获取最近将要触发的事件距离现在的时间，并赋值给 `timer`，使得检测事件时最多只等待这么多时间 (保证定时器事件准时被处理)；设置 `flag` 中的 `NGX_UPDATE_TIME` 以更新缓存的时间
3. 如果配置文件中开启了 `accept_mutex`，那么检测负载均衡阈值 `ngx_accept_disabled` - 如果是正数，则减 1
4. 如果负载均衡阈值是负数，则没有触发负载均衡，试图获取 `accept_mutex` 锁
5. 如果锁获取成功，那么将 `flag` 中加上 `NGX_POST_EVENTS`
6. 如果没有获取到锁，(则等待一会儿再抢锁？)
7. 调用 `ngx_process_events()` 函数分发处理事件，并计算这个函数执行时消耗的时间
8. 如果 `ngx_posted_accept_events` 队列不为空，则依次调用队列中每个事件的回调，建立新连接
9. 如果当前进程持有 `accept_mutex` 锁，则释放锁
10. 如果 `ngx_process_events()` 的执行时间差大于 0，那么可能有新的定时器事件被触发，调用 `ngx_event_expire_timers()` 函数处理所有到期的定时器事件
11. 如果 `ngx_posted_events` 队列不为空，则调用每个事件的回调，处理读写事件
