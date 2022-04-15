# Chapter 14 - 进程间通信机制

Created by : Mr Dk.

2020 / 08 / 06 22:31

Nanjing, Jiangsu, China

---

## 14.1 概述

Nginx 框架使用了三种 **传递消息** 的方式：

- 共享内存
- 套接字
- 信号

在多个进程访问共享资源时，还需要通过 **进程同步** 使各进程安全地访问资源：

- 原子操作
- 信号量
- 文件锁

由于 Nginx 地每个 worker 进程都会同时处理千万个请求，所以处理任意一个请求时，都不应该阻塞当前进程。所以应当尽可能避免 worker 进程进入睡眠状态。

---

## 14.2 共享内存

当多个进程使用同一块共享内存时，在任何一个进程修改了共享内存中的内容后，其它进程通过访问这段共享内存，能够立刻得到修改后的内容。

Nginx 定义了结构体 `ngx_shm_t` 结构体描述一块共享内存：

```c
typedef struct {
    u_char      *addr; // 指向共享内存的起始地址
    size_t       size; // 共享内存的长度
    ngx_str_t    name; // 这块共享内存的名称
    ngx_log_t   *log; // 记录日志的对象
    ngx_uint_t   exists;   /* unsigned  exists:1;  */ // 这块内存是否已经被分配过
} ngx_shm_t;
```

操作这个结构体的行为有两个：分配与释放。

Linux 中提供 `mmap` 系统调用，可以将磁盘文件映射到内存中。用户可以直接操作内存，Linux 内核负责将内存同步到磁盘文件中。而 Nginx 中并没有用到映射文件的功能，仅使用分配内存的功能：

```c
ngx_int_t
ngx_shm_alloc(ngx_shm_t *shm)
{
    // 开辟一块 shm->size 大小的可读写共享内存
    // 内存首地址将会存放在 shm->addr 中
    shm->addr = (u_char *) mmap(NULL, shm->size,
                                PROT_READ|PROT_WRITE,
                                MAP_ANON|MAP_SHARED, -1, 0);

    // 分配失败
    if (shm->addr == MAP_FAILED) {
        ngx_log_error(NGX_LOG_ALERT, shm->log, ngx_errno,
                      "mmap(MAP_ANON|MAP_SHARED, %uz) failed", shm->size);
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

而当释放内存时，需要使用与 `mmap` 相对应的 `munmap` 系统调用：

```c
void
ngx_shm_free(ngx_shm_t *shm)
{
    if (munmap((void *) shm->addr, shm->size) == -1) {
        ngx_log_error(NGX_LOG_ALERT, shm->log, ngx_errno,
                      "munmap(%p, %uz) failed", shm->addr, shm->size);
    }
}
```

出于 Nginx 的跨平台设计，共享内存有三种实现：

- 不映射文件，使用 `mmap` 分配共享内存
- 以 `/dev/zero` 文件使用 `mmap` 分配共享内存
- 使用 `shmget` 分配共享内存

Nginx 中，各进程间共享数据的主要方式就是使用共享内存。一般由 master 进程创建共享内存，在 master 进程 fork 出 worker 子进程后，所有进程就开始使用这块共享内存了。作为一个 Web 服务器，Nginx 需要统计整个服务器中 HTTP 连接状况的总体功能 (各 worker 子进程的连接状况总和)。Nginx 定义了一些原子变量，用于统计连接状况：

```c
static ngx_atomic_t   ngx_stat_accepted0; // 已经成功建立过的 TCP 连接数
ngx_atomic_t         *ngx_stat_accepted = &ngx_stat_accepted0;
static ngx_atomic_t   ngx_stat_handled0; // 已经被 handle 的连接数
ngx_atomic_t         *ngx_stat_handled = &ngx_stat_handled0;
static ngx_atomic_t   ngx_stat_requests0; // 已经被 HTTP 模块处理过的连接数
ngx_atomic_t         *ngx_stat_requests = &ngx_stat_requests0;
static ngx_atomic_t   ngx_stat_active0; // 活跃连接数
ngx_atomic_t         *ngx_stat_active = &ngx_stat_active0;
static ngx_atomic_t   ngx_stat_reading0; // 正在接收 TCP 流的连接数
ngx_atomic_t         *ngx_stat_reading = &ngx_stat_reading0;
static ngx_atomic_t   ngx_stat_writing0; // 正在发送 TCP 流的连接数
ngx_atomic_t         *ngx_stat_writing = &ngx_stat_writing0;
static ngx_atomic_t   ngx_stat_waiting0; // ?
ngx_atomic_t         *ngx_stat_waiting = &ngx_stat_waiting0;
```

在 Nginx 中，显然需要根据这些变量所占的内存，创建一块能包含这些变量的共享内存。在事件模块的初始化过程中：

```c
static ngx_int_t
ngx_event_module_init(ngx_cycle_t *cycle)
{
    void              ***cf;
    u_char              *shared;
    size_t               size, cl;
    ngx_shm_t            shm;
    ngx_time_t          *tp;
    ngx_core_conf_t     *ccf;
    ngx_event_conf_t    *ecf;

    cf = ngx_get_conf(cycle->conf_ctx, ngx_events_module);
    ecf = (*cf)[ngx_event_core_module.ctx_index];

    if (!ngx_test_config && ngx_process <= NGX_PROCESS_MASTER) {
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                      "using the \"%s\" event method", ecf->name);
    }

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    ngx_timer_resolution = ccf->timer_resolution;

#if !(NGX_WIN32)
    {
    ngx_int_t      limit;
    struct rlimit  rlmt;

    if (getrlimit(RLIMIT_NOFILE, &rlmt) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "getrlimit(RLIMIT_NOFILE) failed, ignored");

    } else {
        if (ecf->connections > (ngx_uint_t) rlmt.rlim_cur
            && (ccf->rlimit_nofile == NGX_CONF_UNSET
                || ecf->connections > (ngx_uint_t) ccf->rlimit_nofile))
        {
            limit = (ccf->rlimit_nofile == NGX_CONF_UNSET) ?
                         (ngx_int_t) rlmt.rlim_cur : ccf->rlimit_nofile;

            ngx_log_error(NGX_LOG_WARN, cycle->log, 0,
                          "%ui worker_connections exceed "
                          "open file resource limit: %i",
                          ecf->connections, limit);
        }
    }
    }
#endif /* !(NGX_WIN32) */


    if (ccf->master == 0) {
        return NGX_OK;
    }

    if (ngx_accept_mutex_ptr) {
        return NGX_OK;
    }


    /* cl should be equal to or greater than cache line size */

    // 缓存行大小
    // 这里每个原子变量都独占一个缓存行
    // 应该是为了消除缓存的伪共享性，防止共享内存中的值需要反复刷新到进程栈中
    cl = 128;

    // 每个原子变量占一个缓存行大小
    size = cl            /* ngx_accept_mutex */
           + cl          /* ngx_connection_counter */
           + cl;         /* ngx_temp_number */

#if (NGX_STAT_STUB)

    size += cl           /* ngx_stat_accepted */
           + cl          /* ngx_stat_handled */
           + cl          /* ngx_stat_requests */
           + cl          /* ngx_stat_active */
           + cl          /* ngx_stat_reading */
           + cl          /* ngx_stat_writing */
           + cl;         /* ngx_stat_waiting */

#endif

    // 初始化 ngx_shm_t 结构体
    shm.size = size;
    ngx_str_set(&shm.name, "nginx_shared_zone");
    shm.log = cycle->log;

    // 分配共享内存
    if (ngx_shm_alloc(&shm) != NGX_OK) {
        return NGX_ERROR;
    }

    shared = shm.addr;

    // 负载均衡锁占共享内存中的第一块
    ngx_accept_mutex_ptr = (ngx_atomic_t *) shared;
    ngx_accept_mutex.spin = (ngx_uint_t) -1;

    // 初始化负载均衡锁
    if (ngx_shmtx_create(&ngx_accept_mutex, (ngx_shmtx_sh_t *) shared,
                         cycle->lock_file.data)
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    // 共享内存的第二块用于统计建立过的连接数
    ngx_connection_counter = (ngx_atomic_t *) (shared + 1 * cl);

    // CAS 设置连接数的值
    (void) ngx_atomic_cmp_set(ngx_connection_counter, 0, 1);

    ngx_log_debug2(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "counter: %p, %uA",
                   ngx_connection_counter, *ngx_connection_counter);

    ngx_temp_number = (ngx_atomic_t *) (shared + 2 * cl);

    tp = ngx_timeofday();

    ngx_random_number = (tp->msec << 16) + ngx_pid;

#if (NGX_STAT_STUB)

    // 初始化其余共享变量
    ngx_stat_accepted = (ngx_atomic_t *) (shared + 3 * cl);
    ngx_stat_handled = (ngx_atomic_t *) (shared + 4 * cl);
    ngx_stat_requests = (ngx_atomic_t *) (shared + 5 * cl);
    ngx_stat_active = (ngx_atomic_t *) (shared + 6 * cl);
    ngx_stat_reading = (ngx_atomic_t *) (shared + 7 * cl);
    ngx_stat_writing = (ngx_atomic_t *) (shared + 8 * cl);
    ngx_stat_waiting = (ngx_atomic_t *) (shared + 9 * cl);

#endif

    return NGX_OK;
}
```

这些统计变量全部位于共享内存中，由 worker 进程共同修改。对于整个 Nginx 服务来说全局有效。

---

## 14.3 原子操作

能够进行原子操作的变量只有整型 (应当是与支持原子操作的指令定义有关)：

- `ngx_atomic_uint_t`
- `ngx_atomic_t`

这两种类型都由 `volatile` 关键字修饰，使编译器不会优化它。

对原子变量进行修改，需要使用特定的函数：

- `ngx_atomic_cmp_set()` - 将旧值设置为新值
- `ngx_atomic_fetch_add()` - 原子加法

从函数名上来看，显然都使用了 CAS 的思想。Nginx 为了尽可能不让 worker 进程休眠，在同步上一定是尽可能自旋。这两个函数是 Nginx 封装的，因此 Nginx 在实现中还需要考虑相应的 OS 是否支持原子操作。

在 x86 架构下，Nginx 需要通过内联汇编直接进行操作，同时还要指定 `volatile` 关键字防止编译器优化。在汇编中，最终使用 `cmpxchgl` 指令实现 CAS：

- 首先使用 `lock` 指令锁住总线，防止多核并行执行
- 判断旧值与参数值是否相等
- 如果旧值与参数值相等，那么将旧值替换为新值

另外，使用 `xaddl` 指令可以实现原子加法。

### 14.3.3 自旋锁

Nginx 基于上述的原子变量实现了 **自旋锁**。自旋锁是一种 **非睡眠锁**，如果进程试图获取锁失败，将不会进入睡眠状态，而是在获取锁的代码处自旋。每当内核调度到这个进程时，进程就能持续检查是否可以获取到锁。

自旋锁保护的场景为，进程使用锁的时间非常短 - 甚至比进程进入睡眠再被唤醒 + 上下文切换的时间还要短。在这种场景下，自旋损失的 CPU 资源是极其少的。

Nginx 基于一个原子变量实现了自旋锁。原子变量为 0 时，表示锁被释放。通过 `ngx_atomic_cmp_set()` 函数将原子变量设置为 0 就能够释放锁，而获得锁的过程要复杂些：

```c
void
ngx_spinlock(ngx_atomic_t *lock, ngx_atomic_int_t value, ngx_uint_t spin)
{

#if (NGX_HAVE_ATOMIC_OPS)

    ngx_uint_t  i, n;

    for ( ;; ) {

        // 锁没有被持有
        // 试图将锁通过 CAS 设置为 value，企图占有锁
        if (*lock == 0 && ngx_atomic_cmp_set(lock, 0, value)) {
            return;
        }

        // 多 CPU 场景
        // 最好不要立刻让出 CPU，而是等一等，看看其它 core 上的进程是否会释放锁
        // 减少上下文切换的次数
        if (ngx_ncpu > 1) {

            for (n = 1; n < spin; n <<= 1) {

                // 随着等待的次数越来越多，进入下一个分支的频率会越来越低
                // 检查 lock 值消耗相对较多的 CPU
                for (i = 0; i < n; i++) {
                    // 该函数是许多架构中专门为自旋锁提供的指令
                    // CPU 会将自身置于节能状态，降低等待时的功耗
                    // 执行这个指令并不会让出 CPU
                    ngx_cpu_pause();
                }

                // 检查锁是否被释放，如果被释放，则试图通过 CAS 占有锁
                if (*lock == 0 && ngx_atomic_cmp_set(lock, 0, value)) {
                    return;
                }
            }
        }

        // 让出 CPU，进程依旧是可执行状态
        // 相当于让内核调用 schedule() 调度其它可执行的进程
        ngx_sched_yield();
    }

#else

#if (NGX_THREADS)

#error ngx_spinlock() or ngx_atomic_cmp_set() are not defined !

#endif

#endif

}
```

从实现中可以看出，Nginx 中的自旋锁充分考虑了单 CPU 与多 CPU 的场景，对持有锁时间非常短的场景很有效率。

---

## 14.4 Nginx 频道

Nginx 的 master 与 worker 进程之间的通信是由 `ngx_channel_t` 频道实现的，使用本机套接字实现。通过 `socketpair()` 函数，可以创建一对本机 socket `sv[2]`。向一个描述符中写入，可以从另一个描述符中读取。从一个进程 fork 出另一个进程后，子进程也会继承这个 socket。由此，两个进程可以通过 socket 中的四个描述符进行双工通信。目前，Nginx 仅使用从 master 到 worker 方向的频道，用于 master 进程对 worker 进程的管理。在每次派生 worker 子进程之前，先通过 `socketpair()` 函数产生描述符，然后再派生子进程。在每个进程的数据结构成员中，保存着描述符：

```c
typedef struct {
    // ...
    ngx_socket_t channel[2];
    // ...
} ngx_process_t;
```

> 话说，这不就是 Linux 的管道吗......

Nginx 定义了 master 与 worker 进程之间的消息格式：

```c
typedef struct {
    ngx_uint_t  command; // 命令
    ngx_pid_t   pid; // 发送命令方的 pid
    ngx_int_t   slot; // 发送命令方在进程数组中的 index
    ngx_fd_t    fd; // 通信的 socket 句柄
} ngx_channel_t;
```

而命令定义如下：

```c
#define NGX_CMD_OPEN_CHANNEL   1
#define NGX_CMD_CLOSE_CHANNEL  2
#define NGX_CMD_QUIT           3
#define NGX_CMD_TERMINATE      4
#define NGX_CMD_REOPEN         5
```

对 master 进程来说，会使用 `channel[0]` 发送消息；对 worker 进程来说，会使用 `channel[1]` 接收消息。Worker 子进程会把接收消息的事件与回调函数注册到事件驱动模块中，在接收到 master 进程的消息后，回调函数将会被调用。

---

## 14.5 信号

信号用于在 Linux 进程之间传递消息。Nginx 通过重定义了一些信号的处理函数来使用信号。Nginx 定义了 `ngx_signal_t` 结构体用于描述接收到信号时的行为：

```c
typedef struct {
    int     signo; // 需要处理的信号
    char   *signame; // 信号对应的名称
    char   *name; // 信号对应的 Nginx 命令
    void  (*handler)(int signo, siginfo_t *siginfo, void *ucontext); // 信号回调函数
} ngx_signal_t;
```

其次，Nginx 定义了一个数组，里面包含了 Nginx 中将会处理的所有信号：

```c
ngx_signal_t  signals[] = {
    { ngx_signal_value(NGX_RECONFIGURE_SIGNAL),
      "SIG" ngx_value(NGX_RECONFIGURE_SIGNAL),
      "reload",
      ngx_signal_handler },

    { ngx_signal_value(NGX_REOPEN_SIGNAL),
      "SIG" ngx_value(NGX_REOPEN_SIGNAL),
      "reopen",
      ngx_signal_handler },

    { ngx_signal_value(NGX_NOACCEPT_SIGNAL),
      "SIG" ngx_value(NGX_NOACCEPT_SIGNAL),
      "",
      ngx_signal_handler },

    { ngx_signal_value(NGX_TERMINATE_SIGNAL),
      "SIG" ngx_value(NGX_TERMINATE_SIGNAL),
      "stop",
      ngx_signal_handler },

    { ngx_signal_value(NGX_SHUTDOWN_SIGNAL),
      "SIG" ngx_value(NGX_SHUTDOWN_SIGNAL),
      "quit",
      ngx_signal_handler },

    { ngx_signal_value(NGX_CHANGEBIN_SIGNAL),
      "SIG" ngx_value(NGX_CHANGEBIN_SIGNAL),
      "",
      ngx_signal_handler },

    { SIGALRM, "SIGALRM", "", ngx_signal_handler },

    { SIGINT, "SIGINT", "", ngx_signal_handler },

    { SIGIO, "SIGIO", "", ngx_signal_handler },

    { SIGCHLD, "SIGCHLD", "", ngx_signal_handler },

    { SIGSYS, "SIGSYS, SIG_IGN", "", NULL },

    { SIGPIPE, "SIGPIPE, SIG_IGN", "", NULL },

    { 0, NULL, "", NULL }
};
```

比如，在执行 `nginx -s reload` 时，会新拉起一个进程，向所有进程发送信号。这样，运行中的进程将调用信号处理函数来处理信号 (比如重新打开文件)。

上述信号数组会被用于 `ngx_init_signals()` 函数中的信号初始化，向 Linux 注册所用到的信号的回调函数。

```c
ngx_int_t
ngx_init_signals(ngx_log_t *log)
{
    ngx_signal_t      *sig;
    struct sigaction   sa;

    // 信号数组中的每一个信号
    for (sig = signals; sig->signo != 0; sig++) {
        ngx_memzero(&sa, sizeof(struct sigaction));

        // 设置信号处理函数
        if (sig->handler) {
            sa.sa_sigaction = sig->handler;
            sa.sa_flags = SA_SIGINFO;

        } else {
            sa.sa_handler = SIG_IGN;
        }

        sigemptyset(&sa.sa_mask);
        // 向 Linux 注册信号的回调函数
        if (sigaction(sig->signo, &sa, NULL) == -1) {
#if (NGX_VALGRIND)
            ngx_log_error(NGX_LOG_ALERT, log, ngx_errno,
                          "sigaction(%s) failed, ignored", sig->signame);
#else
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                          "sigaction(%s) failed", sig->signame);
            return NGX_ERROR;
#endif
        }
    }

    return NGX_OK;
}
```

---

## 14.6 信号量

保证两个或多个代码段不被并发访问。信号量 **可能导致进程睡眠**，因此要谨慎使用。Nginx 仅把信号量作为简单的互斥锁来使用。最初的信号量的值为 0，调用 `sem_post()` 函数会把信号量的值 + 1，这个操作不会有任何阻塞。调用 `sem_wait()` 函数会将信号量的值 - 1。如果信号量的值已经为 0，那么进程将在这里阻塞，进入睡眠。直到其它进程释放信号量后，该进程才会被唤醒，从而继续执行。

---

## 14.7 文件锁

Linux 内核提供了基于文件的互斥锁：

```c
int fcntl(int fd, int cmd, struct flock *lock);
```

其中：

- `fd` 是一个已经打开的文件句柄
- `cmd` 表示执行的锁操作
- `lock` 描述了锁信息

`cmd` 参数在 Nginx 中只会有两个值：

- `F_SETLK` - 在争夺锁时，如果失败，则立刻返回
- `F_SETLKW` - 在争夺锁时，如果失败，则当前进程阻塞在该函数上，进程转为睡眠状态

而 `lock` 参数用于描述锁信息：

```c
struct flock
{
    short l_type; // 锁类型
    short l_whence; // 锁区域起始地址
    long l_start; // 锁区域相对于起始地址的偏移量
    long l_len; // 锁区域的长度
    long l_pid; // 拥有锁的 pid
};
```

这个结构体可以支持锁住文件中的部分内容。但是 Nginx 封装的文件锁仅用于保护代码的顺序执行，使用方式较为简单 - 一个文件对应一个全局互斥锁。因此大部分取值都是固定的，而 `l_type` 取决于用户想实现阻塞睡眠的锁还是非阻塞的锁。

Nginx 为文件锁封装了三个函数：

- `ngx_trylock_fd()` - 不会阻塞进程的互斥锁
- `ngx_lock_fd()` - 会阻塞进程执行，导致 worker 进程进入睡眠
- `ngx_unlock_fd()` - 释放锁

---

## 14.8 互斥锁

基于原子操作、信号量、文件锁，Nginx 在更高层次上封装了一个互斥锁。

```c
typedef struct {
// 如果支持原子变量
#if (NGX_HAVE_ATOMIC_OPS)
    ngx_atomic_t  *lock; // 原子变量
// 如果支持信号量
#if (NGX_HAVE_POSIX_SEM)
    ngx_atomic_t  *wait; // 原子变量
    ngx_uint_t     semaphore; // 信号量值
    sem_t          sem; // 信号量锁
#endif
#else
// 文件锁
    ngx_fd_t       fd; // 文件句柄
    u_char        *name; // 文件名
#endif
    ngx_uint_t     spin; // 自旋次数 (自旋状态下，等待其它 CPU core 释放锁的时间)
} ngx_shmtx_t;
```

不支持原子操作时，就用文件锁来实现互斥锁；支不支持信号量只会影响获得锁的方式 - 如果不支持信号量，那么互斥锁就相当于自旋锁；如果支持信号量，那么在自旋等待超时后，进程将进入睡眠。

### 14.8.1 文件锁实现的 `ngx_shmtx_t`

首先创建 `ngx_shmtx_t` 结构体，然后调用 `ngx_shmtx_create()` 函数创建锁。这个函数的主要目的是确保 fd 可用。

```c
ngx_int_t
ngx_shmtx_create(ngx_shmtx_t *mtx, ngx_shmtx_sh_t *addr, u_char *name)
{
    // mtx 中已经有值
    if (mtx->name) {

        // mtx 已经被相同的文件初始化过了
        if (ngx_strcmp(name, mtx->name) == 0) {
            mtx->name = name;
            return NGX_OK;
        }

        // 文件名不一致，说明使用了新文件锁，因此销毁原文件锁
        ngx_shmtx_destroy(mtx);
    }

    // 按照指定的文件名创建并打开文件
    mtx->fd = ngx_open_file(name, NGX_FILE_RDWR, NGX_FILE_CREATE_OR_OPEN,
                            NGX_FILE_DEFAULT_ACCESS);

    // 文件无法打开
    if (mtx->fd == NGX_INVALID_FILE) {
        ngx_log_error(NGX_LOG_EMERG, ngx_cycle->log, ngx_errno,
                      ngx_open_file_n " \"%s\" failed", name);
        return NGX_ERROR;
    }

    // 只需要这个文件的 inode 信息，因此可以把文件删除
    // 只需要 fd 可用
    if (ngx_delete_file(name) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                      ngx_delete_file_n " \"%s\" failed", name);
    }

    mtx->name = name;

    return NGX_OK;
}
```

相应的销毁函数就是关闭已经打开的 fd 句柄：

```c
void
ngx_shmtx_destroy(ngx_shmtx_t *mtx)
{
    if (ngx_close_file(mtx->fd) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                      ngx_close_file_n " \"%s\" failed", mtx->name);
    }
}
```

以非阻塞的方式获取锁：

```c
ngx_uint_t
ngx_shmtx_trylock(ngx_shmtx_t *mtx)
{
    ngx_err_t  err;

    err = ngx_trylock_fd(mtx->fd);

    if (err == 0) {
        return 1;
    }

    // 已被其它进程持有
    if (err == NGX_EAGAIN) {
        return 0;
    }

#if __osf__ /* Tru64 UNIX */

    if (err == NGX_EACCES) {
        return 0;
    }

#endif

    ngx_log_abort(err, ngx_trylock_fd_n " %s failed", mtx->name);

    return 0;
}
```

阻塞方式获取锁 - 将使进程睡眠，等待其它进程释放锁后由内核唤醒。这个函数没有返回值，因为它的返回就意味着已经成功获得了锁 - 代码将会继续向下执行。

```c
void
ngx_shmtx_lock(ngx_shmtx_t *mtx)
{
    ngx_err_t  err;

    err = ngx_lock_fd(mtx->fd);

    if (err == 0) {
        return;
    }

    ngx_log_abort(err, ngx_lock_fd_n " %s failed", mtx->name);
}
```

### 14.8.2 原子变量实现的 `ngx_shmtx_t`

原子变量 `lock` 用于表示当前锁的状态。当 `lock` 的值为 0 时，说明没有进程持有该锁；当 `lock` 的值为负数时，表示有进程正持有锁 - Nginx 通过判断 `lock` 变量的符号位快速判断锁是否已被持有。

初始化锁：

```c
ngx_int_t
ngx_shmtx_create(ngx_shmtx_t *mtx, ngx_shmtx_sh_t *addr, u_char *name)
{
    mtx->lock = &addr->lock;

    if (mtx->spin == (ngx_uint_t) -1) {
        return NGX_OK;
    }

    // 自旋等待时间默认为 2048
    mtx->spin = 2048;

// 同时使用信号量
#if (NGX_HAVE_POSIX_SEM)

    mtx->wait = &addr->wait;

    // 将信号量的值初始化为 0
    if (sem_init(&mtx->sem, 1, 0) == -1) {
        ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                      "sem_init() failed");
    } else {
        // 信号量被成功初始化，标志位设为 1
        mtx->semaphore = 1;
    }

#endif

    return NGX_OK;
}
```

释放锁：

```c
void
ngx_shmtx_destroy(ngx_shmtx_t *mtx)
{
#if (NGX_HAVE_POSIX_SEM)

    // 把向内核申请的信号量给释放
    if (mtx->semaphore) {
        if (sem_destroy(&mtx->sem) == -1) {
            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                          "sem_destroy() failed");
        }
    }

#endif
}
```

非阻塞获取锁 (代码已经重构为与书上说法不一致了 - 没有看到所谓的判断 `lock` 是否为负数)：

```c
ngx_uint_t
ngx_shmtx_trylock(ngx_shmtx_t *mtx)
{
    // 先判断锁是否被持有，然后通过 CAS 操作将锁的状态置为已被持有
    return (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid));
}
```

阻塞获取锁：

```c
void
ngx_shmtx_lock(ngx_shmtx_t *mtx)
{
    ngx_uint_t         i, n;

    ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0, "shmtx lock");

    // 在没拿到锁之间不会退出循环
    for ( ;; ) {

        // 试图获得锁
        if (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid)) {
            return;
        }

        // 如果是多核 CPU
        if (ngx_ncpu > 1) {

            // 尝试失败越多次，降频频率越低
            for (n = 1; n < mtx->spin; n <<= 1) {

                for (i = 0; i < n; i++) {
                    // 降低 CPU 功耗，不让出 CPU
                    ngx_cpu_pause();
                }

                // 试图获得锁
                if (*mtx->lock == 0
                    && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid))
                {
                    return;
                }
            }
        }

#if (NGX_HAVE_POSIX_SEM)

        // 如果支持信号量，且使用了信号量
        if (mtx->semaphore) {
            // ?
            (void) ngx_atomic_fetch_add(mtx->wait, 1);

            // ?
            if (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid)) {
                (void) ngx_atomic_fetch_add(mtx->wait, -1);
                return;
            }

            ngx_log_debug1(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                           "shmtx wait %uA", *mtx->wait);

            // 检查信号量的值是否为正数
            // 如果不是正数，那么当前进程进入睡眠状态
            while (sem_wait(&mtx->sem) == -1) {
                ngx_err_t  err;

                err = ngx_errno;

                if (err != NGX_EINTR) {
                    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, err,
                                  "sem_wait() failed while waiting on shmtx");
                    break;
                }
            }

            ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                           "shmtx awoke");

            continue;
        }

#endif

        // 让出 CPU
        ngx_sched_yield();
    }
}
```

当不使用信号量时，这里实际上就是一个自旋锁。使用信号量后，进程将可能进入睡眠 - 所以不建议在 Nginx worker 进程中使用带信号量的取锁函数。

释放锁的过程不会阻塞进程，由 CAS 操作实现：

```c
void
ngx_shmtx_unlock(ngx_shmtx_t *mtx)
{
    if (mtx->spin != (ngx_uint_t) -1) {
        ngx_log_debug0(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0, "shmtx unlock");
    }

    // 通过 CAS 操作获得锁
    if (ngx_atomic_cmp_set(mtx->lock, ngx_pid, 0)) {
        // 如果使用了信号量，那么唤醒睡眠等待的进程
        ngx_shmtx_wakeup(mtx);
    }
}

static void
ngx_shmtx_wakeup(ngx_shmtx_t *mtx)
{
// 仅在支持信号量的情况下执行
#if (NGX_HAVE_POSIX_SEM)
    ngx_atomic_uint_t  wait;

    // 没有使用信号量，那么直接结束
    if (!mtx->semaphore) {
        return;
    }

    // 循环判断 wait 变量的值是否为正数
    // 如果是正数，那么将其的值 - 1
    for ( ;; ) {

        wait = *mtx->wait;

        if ((ngx_atomic_int_t) wait <= 0) {
            return;
        }

        if (ngx_atomic_cmp_set(mtx->wait, wait, wait - 1)) {
            break;
        }
    }

    ngx_log_debug1(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                   "shmtx wake %uA", wait);

    // 信号量的值 + 1，表示释放了信号量互斥锁
    if (sem_post(&mtx->sem) == -1) {
        ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, ngx_errno,
                      "sem_post() failed while wake shmtx");
    }

#endif
}
```
