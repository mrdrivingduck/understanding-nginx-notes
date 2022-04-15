# Chapter 8.3-8.6 - Nginx 框架核心结构体

Created by : Mr Dk.

2020 / 07 / 20 13:59

Nanjing, Jiangsu, China

---

## 8.3 Nginx 框架中的核心结构体 `ngx_cycle_t`

Nginx 核心的框架代码围绕着这个结构体展开。每个进程 (master / worker / cache manager / cache loader) 都会拥有一个唯一的 `ngx_cyclt_t` 结构体。Nginx 的启动初始化、正常运行都以这个结构体为中心。

### 8.3.1 `ngx_listening_t` 结构体

Nginx 作为一个 Web 服务器，需要监听端口以处理其中的网络事件。在 Nginx 中，每个 `ngx_listening_t` 结构体都代表着一个监听端口。这样的结构体组成的数组会被维护在 `ngx_cycle_t` 核心结构体中：

```c
typedef struct ngx_listening_s ngx_listening_t;

struct ngx_listening_s {
    ngx_socket_t fd; // socket 句柄
    struct sockaddr *sockaddr; // 监听地址
    socklen_t socklen; // 地址长度

    size_t addr_text_max_len; // IP 地址字符串最大长度
    ngx_str_t addr_text; // 字符串形式的 IP 地址

    int type; // socket 类型

    int backlog; // backlog 队列
    int rcvbuf; // 内核 socket 接收缓冲区
    int sndbuf; // 内核 socket 发送缓冲区

    ngx_connection_handler_pt handler; // 新的 TCP 连接建立成功后的处理方法

    void *servers; // 保存当前监听端口对应的所有主机名

    // 日志对象指针
    ngx_log_t log;
    ngx_log_t *logp;

    size_t pool_size; // 为新的 TCP 连接创建的内存池大小
    ngx_msec_t post_accept_timeout; // 连接建立后，指定时间内没有收到用户数据，则丢弃连接

    ngx_listening_t *previous; // 多个结构体由该指针串成单链表
    ngx_connection_t *connection; // 指向对应的连接结构体

    unsigned open:1; // 当前监听句柄是否有效的标志位
    unsigned remain:1; // 初始化新的 ngx_cycle_t 结构体 (运行时更新 Nginx) 时，保留原先打开的监听端口
    unsigned ignore:1; // 跳过设置当前结构体中的 socket
    unsigned bound:1; // 未被使用
    unsigned inherited:1; // 是否继承自前一个进程
    unsigned nonblocking_accept:1; // 未使用
    unsigned listen:1; // 当前结构体对应的 socket 是否已经在监听
    unsigned nonblocking:1; // socket 是否阻塞
    unsigned shared:1; // 无意义
    unsigned addr_ntop:1; // 将网络地址转换为字符串形式的地址
};
```

其中，`handler` 成员在这个监听端口上成功建立 TCP 连接后就会被回调：

```c
typedef void (*ngx_connection_handler_pt) (ngx_connection_t *c);
```

### 8.3.2 ngx_cycle_t 结构体

该结构体中保存着 Nginx 配置文件的路径。因此，在初始化时，Nginx 框架就要通过读取配置文件来填充数据结构。

```c
typedef struct ngx_cycle_s ngx_cycle_t;

struct ngx_cycle_s {
    void ****conf_ctx; // 保存所有模块配置项结构体的指针

    ngx_pool_t *pool; // 内存池

    ngx_log_t *log;
    ngx_log_t new_log;

    ngx_uint_t files_n;
    ngx_connection_t **files; // 所有连接
    ngx_connection_t *free_connections; // 可用连接池
    ngx_uint_t free_connection_n;

    ngx_queue_t resuable_connections_queue; // 可重复使用的 ngx_connection_t 连接队列
    ngx_array_t listening; // 监听端口 ngx_listening_t 的动态数组
    ngx_array_t pathes; // Nginx 要操作的所有目录，如果不存在则试图创建，如果不成功则启动失败
    ngx_list_t open_files; // Nginx 已经打开的所有文件
    ngx_list_t shared_memory; // 共享内存链表

    ngx_uint_t connection_n; // 当前进程的连接总数
    ngx_connection_t *connections; // 当前进程的所有连接对象

    ngx_event_t *read_events; // 当前进程的所有读事件对象
    ngx_event_t *write_events; // 当前进程的所有写事件对象

    ngx_cycle_t *old_cycle; // 用于引用前一个 ngx_cycle_t 对象的成员

    ngx_str_t conf_file; // 配置文件相对于安装目录的路径
    ngx_str_t conf_param; // 需要特殊处理的命令行参数
    ngx_str_t conf_prefix; // 配置文件所在的目录路径
    ngx_str_t prefix; // Nginx 安装目录
    ngx_str_t lock_file; // 用于进程同步的文件锁名称
    ngx_str_t hostname; // gethostname() 系统调用得到的主机名
};
```

在构造这个结构体的函数中，大部分的属性都会被初始化。

## 8.4 Nginx 启动时框架的处理流程

1. 解析命令行参数，确定配置文件路径，创建一个临时的 `ngx_cycle_t` 变量存放配置文件路径
2. (平滑升级)，旧的 master 进程通过 `fork()` 和 `execve()` 启动新版本的 master 进程，并通过环境变量传递升级信息
3. 调用所有核心模块的 `create_conf()` 函数，构造各模块用于存储配置项的数据结构
4. 调用配置模块提供的配置项解析函数，遍历配置文件中的所有配置项 (获取要监听的端口)
5. 调用所有模块的 `init_conf()` 函数，在配置项解析完毕后可以做一些处理工作
6. 将 `ngx_cycle_t` 结构体中保存的目录或文件打开，初始化用于进程通信的共享内存
7. 根据配置中解析的待监听端口 (保存在 `ngx_listening_t` 中)，设置 socket 句柄并监听端口
8. 调用所有模块的 `init_module()` 函数

接下来，如果 Nginx 被配置为单进程工作模式，则调用 `ngx_single_process_cycle()` 函数进入单进程工作模式，调用所有模块的 `init_process()` 函数，启动完毕。如果 Nginx 被配置为 master / worker 模式，则还要按照配置文件中指定的数目启动子进程，再调用所有模块的 `init_process()` 函数；另外，还要按情况启动 cache manager 子进程和 cache loader 子进程。

## 8.5 Worker 进程是如何工作的

主要关注 worker 进程的循环 `worker_process_cycle()` 函数。在这个循环中，程序不断对四个全局标志位进行检测，并对标志位执行相应的函数。Nginx 采用信号来设置这些标志位 - `ngx_signal_handler()` 函数根据接收到的信号，设置这些标志位：

| 信号           | 标志位                       | 意义             |
| -------------- | ---------------------------- | ---------------- |
| `QUIT`         | `sig_atomic_t ngx_quit`      | 优雅地关闭进程   |
| `TERM` / `INT` | `sig_atomic_t ngx_terminate` | 强制关闭进程     |
| `USR1`         | `sig_atomic_t ngx_reopen`    | 重新打开所有文件 |

1. 首先检测 `ngx_existing` 标志位是否为 `1`。如果为 `1`，则开始准备关闭 worker 进程 (这个标志由 `ngx_quit` 设置)
   - 对于正在处理的连接，调用关闭连接的函数
   - 检查 `ngx_event_timer_rbtree` 是否还有定时事件要执行
   - 调用所有模块的 `exit_process()` 函数，销毁内存池，退出 worker 进程
2. 调用 `ngx_process_events_and_timers()` 函数处理定时事件 (事件模块的核心函数)
3. 检查 `ngx_terminate` 标志位，如果置位，则立刻调用所有模块的 `exit_process()` 函数，并销毁内存池，退出 worker 进程
4. 检查 `ngx_quit` 标志位，如果置位，则优雅地关闭连接 - 首先将进程的名称修改，然后关闭所有监听的端口，并设置 `ngx_exiting` 标志为 `1`，继续向下执行
5. 检查 `ngx_reopen` 标志位，如果置位，则需要重新打开所有文件
6. 回到 1，进入下一个循环

## 8.6 Master 进程是如何工作的

Master 进程不处理网络事件，只会管理 worker 子进程来实现重启服务、平滑升级、更换日志文件、更换配置文件等功能。Master 进程的主循环也是通过检查由信号设置的标志位来实现的，同样，`ngx_signal_handler()` 函数根据接收到的信号设置这些标志位。

| 信号           | 标志位              | 意义                                       |
| -------------- | ------------------- | ------------------------------------------ |
| `QUIT`         | `ngx_quit`          | 优雅地关闭服务                             |
| `TERM` / `INT` | `ngx_terminate`     | 强制关闭服务                               |
| `USR1`         | `ngx_reopen`        | 重新打开服务中的所有文件                   |
| `WINCH`        | `ngx_noaccept`      | 所有子进程不再处理新的连接                 |
| `USR2`         | `ngx_change_binary` | 平滑升级到新版本的 Nginx                   |
| `HUP`          | `ngx_reconfigure`   | 重新读取配置文件，并使服务对新配置项生效   |
| `CHLD`         | `ngx_reap`          | 有子进程意外结束，此时需要监控所有的子进程 |
|                | `ngx_restart`       |                                            |

Master 进程的所有子进程信息都被维护在一个数组中，用于管理子进程：

```c
#define NGX_MAX_PROCESSES 1024 // 最多 1024 个子进程

ngx_int_t ngx_process_slot; // 当前操作的进程下标

ngx_int_t ngx_last_process; // 有意义进程的最大下标

ngx_process_t ngx_processes[NGX_MAX_PROCESS]; // 子进程数组

typedef struct {
    ngx_pid_t pid; // 进程 ID
    int status; // 有 waitpid() 获取到的进程状态
    ngx_socket_t channel[2]; // 用于 master 与 worker 之间的通信

    ngx_spawn_proc_pt proc; // 子进程的循环执行函数
    void *data; // 函数指针

    char *name; // 进程名称

    unsigned respawn:1; // 标志位，重新生成子进程
    unsigned just_spawn:1; // 标志位，正在生成子进程
    unsigned detached:1; // 标志位，正在进行父子进程分离
    unsigned exiting:1; // 标志位，进程正在退出
    unsigned exited:1; // 标志位，进程已经退出
} ngx_process_t;
```

Master 进程如何启动子进程？首先通过 `fork()` 系统调用产生子进程，并在 `ngx_processes` 数组中选择一个还未使用的槽来存储这个子进程的相关信息。在构造子进程时，需要将子进程将要执行的工作循环函数指针作为参数。因此 worker 进程、cache manager 进程、cache loader 进程的工作循环的定义方式都是一致的：

```c
// 产生子进程调用的函数
ngx_pid_t ngx_spawn_process(ngx_cycle_t *cycle, ngx_spawn_proc_pt proc, void *data, char *name, ngx_int_t respawn);

typedef void (*ngx_spawn_proc_pt) (ngx_cycle_t *cycle, void *data); // 子进程工作循环函数指针定义

static void ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data); // Worker 进程工作循环
static void ngx_cache_manager_process_cycle(ngx_cycle_t *cycle, void *data); // Cache 进程工作循环
```

当子进程意外退出时，master 进程会收到来自内核的 `CHLD` 信号。通过遍历 `ngx_processes` 数组，维护子进程的状态。在 master 进程的工作循环中，对上述八个标志位进行检测，如果标志位被置位，则进入相应的分支进行处理：

1. (由内核 `CHLD` 信号激活) `ngx_reap` 标志置位，则遍历 `ngx_processes` 数组并检查每个子进程的状态，重新拉起非正常退出的子进程，返回 `live` 标志
2. 如果 `live` 标志为 0 (所有子进程正常退出) 且 `ngx_terminate` / `ngx_quit` 标志位为 1 时，开始退出 master 进程，删除存储 pid 的文件
3. 调用所有模块的 `exit_master()` 函数
4. 调用 `ngx_master_process_exit()` 关闭进程打开的监听端口
5. 销毁内存池，退出 master 进程
6. 如果 `ngx_terminate` 标志置位，则向所有子进程发送 `TERM` 信号 (强制关闭)，然后挂起进程等待内核信号在第 1 步激活进程
7. 如果 `ngx_quit` 标志置位，则要开始着手优雅地退出服务 - 向所有子进程发送 `QUIT` 信号 (优雅关闭)
8. 接上步，关闭所有监听端口，挂起 master 进程并等待内核信号在第 1 步激活进程
9. 如果 `ngx_reconfigure` 标志置位，则需要重新读取配置文件 - Nginx 会重新初始化 `ngx_cycle_t` 结构体
10. 接上步，用新的 `ngx_cycle_t` 拉起新的 worker 进程
11. 接上步，根据缓存模块的加载情况，决定是否拉起 cache manager 进程或 cache loader 进程，把 `live` 标志置位
12. 接上步，向使用原有配置的 worker 进程发送 `QUIT` 信号，要求它们优雅地结束服务
13. 如果 `ngx_restart` 标志置位，那么就要拉起 worker 进程
14. 接上步，根据缓存模块情况选择是否启动 cache manager 进程或 cache loader 进程
15. 如果 `ngx_reopen` 标志置位，则重新打开所有文件
16. 接上步，向所有子进程发送 `USR1` 信号，要求子进程重新打开所有文件
17. 如果 `ngx_change_binary` 标志置位，则平滑升级 Nginx，使用新的子进程启动新版本的 Nginx 程序
18. 如果 `ngx_noaccept` 标志置位，则向所有子进程发送 `QUIT`，并设置 `ngx_noaccepting` 表示停止接受新的连接，然后返回第 1 步进入下一个循环
