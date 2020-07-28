# Chapter 9.9-9.10 - 文件的异步 I/O 与 TCP

Created by : Mr Dk.

2020 / 07 / 28 10:08

Nanjing, Jiangsu, China

---

## 9.9 文件的异步 I/O

Linux kernel 2.6.2x 之后开始支持文件异步 I/O (不是 glibc 库中基于多线程实现的异步 I/O)。这种 I/O 由内核实现，只有内核成功完成了磁盘操作，内核才会通知进程。Nginx 把读取文件的操作 **异步地** 提交给内核后，内核会通知 I/O 设备独立工作，此时 Nginx 进程又可以继续充分使用 CPU。另外，当大量读事件堆积到 I/O 设备上时，能够充分发挥 *电梯调度算法* 的优势，降低读取磁盘扇区的成本。

> Linux 内核级别的文件异步 I/O 不支持 **高速缓冲** - 即使需要读取的磁盘块已经位于 buffer 中，内核也还是会到磁盘上读取。从单个请求的角度看，有可能降低处理速度；但是从阻塞 worker 进程的角度来说还是有所改进的。
>
> 考虑使用场景。如果大部分用户请求对文件的操作都会落到缓存中，那么最好不使用异步 I/O。

### 9.9.1 Linux 内核提供的文件异步 I/O

个人认为，异步 I/O 的 API 与事件机制与 EPOLL 高度类似。首先使用 `io_setup()` 获得一个异步 I/O 上下文的描述符 (对应 `epoll_create()`)，然后使用 `io_submit()` 和 `io_cancel()` 向内核中添加或删除事件 (对应 `epoll_ctl()`)。最后调用 `io_getevents()` 获取已经就绪的事件 (对应 `epoll_wait()`)。

Nginx 中只使用了异步 I/O 中的 **读取**，而不支持异步写入。因为文件的异步 I/O 不使用 buffer，而文件的写入通常是落入 buffer 的，再由内核统一将 dirty page 写入磁盘。

在 `epoll_create()` 执行后，如果开启了文件异步 I/O，那么就获取一个异步 I/O 上下文的描述符句柄，并为其设置一个回调函数后，加入 EPOLL 的通知列表中：

```c
static void
ngx_epoll_aio_init(ngx_cycle_t *cycle, ngx_epoll_conf_t *epcf)
{
    int                 n;
    struct epoll_event  ee;

    // 获取异步 I/O 上下文描述符
#if (NGX_HAVE_SYS_EVENTFD_H)
    ngx_eventfd = eventfd(0, 0);
#else
    ngx_eventfd = syscall(SYS_eventfd, 0);
#endif

    if (ngx_eventfd == -1) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                      "eventfd() failed");
        ngx_file_aio = 0;
        return;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "eventfd: %d", ngx_eventfd);

    n = 1;

    if (ioctl(ngx_eventfd, FIONBIO, &n) == -1) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                      "ioctl(eventfd, FIONBIO) failed");
        goto failed;
    }

    // 初始化上下文容量
    if (io_setup(epcf->aio_requests, &ngx_aio_ctx) == -1) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                      "io_setup() failed");
        goto failed;
    }

    // 设置 fd 与被 EPOLL 激活时的回调函数
    ngx_eventfd_event.data = &ngx_eventfd_conn;
    ngx_eventfd_event.handler = ngx_epoll_eventfd_handler;
    ngx_eventfd_event.log = cycle->log;
    ngx_eventfd_event.active = 1;
    ngx_eventfd_conn.fd = ngx_eventfd;
    ngx_eventfd_conn.read = &ngx_eventfd_event;
    ngx_eventfd_conn.log = cycle->log;

    ee.events = EPOLLIN|EPOLLET;
    ee.data.ptr = &ngx_eventfd_conn;

    // 加入 EPOLL 的查看列表中
    if (epoll_ctl(ep, EPOLL_CTL_ADD, ngx_eventfd, &ee) != -1) {
        return;
    }

    ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                  "epoll_ctl(EPOLL_CTL_ADD, eventfd) failed");

    if (io_destroy(ngx_aio_ctx) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "io_destroy() failed");
    }

failed:

    if (close(ngx_eventfd) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "eventfd close() failed");
    }

    ngx_eventfd = -1;
    ngx_aio_ctx = 0;
    ngx_file_aio = 0;
}
```

在 `epoll_wait()` 中检测到异步 I/O 事件就绪后，调用回调函数 `ngx_epoll_eventfd_handler()`，获取所有已经就绪的 I/O 事件，并放入延后队列中：

```c
static void
ngx_epoll_eventfd_handler(ngx_event_t *ev)
{
    int               n, events;
    long              i;
    uint64_t          ready;
    ngx_err_t         err;
    ngx_event_t      *e;
    ngx_event_aio_t  *aio;
    struct io_event   event[64];
    struct timespec   ts;

    ngx_log_debug0(NGX_LOG_DEBUG_EVENT, ev->log, 0, "eventfd handler");

    // 获取本次已经就绪的异步 I/O 事件数量，保存在 ready 中
    n = read(ngx_eventfd, &ready, 8);

    err = ngx_errno;

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, ev->log, 0, "eventfd: %d", n);

    if (n != 8) {
        if (n == -1) {
            if (err == NGX_EAGAIN) {
                return;
            }

            ngx_log_error(NGX_LOG_ALERT, ev->log, err, "read(eventfd) failed");
            return;
        }

        ngx_log_error(NGX_LOG_ALERT, ev->log, 0,
                      "read(eventfd) returned only %d bytes", n);
        return;
    }

    ts.tv_sec = 0;
    ts.tv_nsec = 0;

    // 直到所有就绪事件处理完
    while (ready) {

        // 将就绪事件复制到 events 数组中 (如果一次处理不完，还要多循环几次)
        events = io_getevents(ngx_aio_ctx, 1, 64, event, &ts);

        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                       "io_getevents: %d", events);

        if (events > 0) {
            // 减去已经从内核取出的事件数
            ready -= events;

            // 开始依次处理每一个事件
            for (i = 0; i < events; i++) {

                ngx_log_debug4(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                               "io_event: %XL %XL %L %L",
                                event[i].data, event[i].obj,
                                event[i].res, event[i].res2);

                e = (ngx_event_t *) (uintptr_t) event[i].data;

                e->complete = 1;
                e->active = 0;
                e->ready = 1;

                aio = e->data;
                aio->res = event[i].res;

                // 放入延后处理队列中
                ngx_post_event(e, &ngx_posted_events);
            }

            continue;
        }

        if (events == 0) {
            return;
        }

        /* events == -1 */
        ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_errno,
                      "io_getevents() failed");
        return;
    }
}
```

在延后队列中的事件被执行时，再调用每个事件对应的回调函数，处理业务：

```c
static void
ngx_file_aio_event_handler(ngx_event_t *ev)
{
    ngx_event_aio_t  *aio;

    aio = ev->data;

    ngx_log_debug2(NGX_LOG_DEBUG_CORE, ev->log, 0,
                   "aio event handler fd:%d %V", aio->fd, &aio->file->name);

    if (ngx_file_aio_result(aio->file, aio, ev) != NGX_AGAIN) {
        // 调用回调函数，处理业务
        aio->handler(ev);
    }
}
```

---

## 9.10 TCP 协议与 Nginx

TCP 面向连接，基于建立成功的 TCP 连接为通信双方提供可靠的字节流服务。建立连接需要通过 **三次握手**：

1. 客户端向服务器发起连接 (SYN)
2. 服务器确认连接，并向客户端也发起连接 (ACK + SYN)
3. 客户端确认连接 (ACK)

连接建立的过程由 OS 内核完成，Nginx 等应用程序只是从内核中取出已经建立好的 TCP 连接。

内核在调用 `listen()` 函数后，会为监听端口建立以下两个队列：

* SYN 队列 (未完成握手队列)
* ACCEPT 队列 (已完成握手队列)

客户端的 SYN 到达服务器后，内核会把信息放到 SYN 队列中，并向客户端回复 SYN + ACK；客户端再次发来 ACK 时，内核会把连接从 SYN 队列中取出，放入 ACCEPT 队列中。`accept()` 函数实际上就是把连接从 ACCEPT 队列中取出。如果有大量的连接同时到达，而应用程序不能及时调用 `accept()` 函数，将导致两个队列填满，内核将无法接收新的连接。比如某个 Nginx 模块长时间陷入某个函数的执行，没来得及调用 `accept()`。

内核为每个 TCP 连接都分配了内存，分别充当发送/接收缓冲区。TCP 连接建立时，就可以判断出双方网络间最适宜的、不会再被切分的报文大小 - MSS (最大报文段长度)。`send()` 调用的成功并不意味着报文已经成功发送，而是要发送的数据已经成功被内核从用户态拷贝到了内核态的写缓冲区中。接收报文时，由于 Nginx 是事件驱动的，只有通过 EPOLL 通知 worker 进程接收到了报文，`recv()` 才会被调用，将 TCP 读缓冲区中的内容拷贝到用户态内存中：

* receive 队列是允许用户直接读取的，已经接收到的、排好序的 TCP 报文
* out_of_order 队列存放的是乱序的报文

Nginx 使用好 TCP 协议，就要有效率地使用 CPU 和内存 - 只有在必要时才使用 `send()` / `recv()` 函数。只有接收到报文，且接收到足够多的报文时，worker 进程才调用 `recv()` 函数；只有写缓冲区中有空闲空间时，才调用 `send()` 函数。

---

