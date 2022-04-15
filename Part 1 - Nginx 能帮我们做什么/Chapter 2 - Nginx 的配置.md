# Chapter 2 - Nginx 的配置

Created by : Mr Dk.

2020 / 07 / 16 23:07

Nanjing, Jiangsu, China

---

## 2.1 Nginx 进程之间的关系

Nginx 使用一个 master 进程来管理多个 worker 进程。一般情况下，worker 进程与 CPU 核心数相等。每个 worker 进程用于提供互联网服务，master 进程只负责监控管理 worker 进程。Worker 进程之间通过 **共享内存**、**原子操作** 等进程间通信机制来实现 **负载均衡** 的功能。Master-worker 方式配置的好处：

1. Master 进程不服务用户，仅用于管理 worker 进程，因此可以是唯一的，运行在较高的权限上 (通常用 root 启动 master 进程)，且当 worker coredump 时，master 可以立刻启动新的 worker 进程继续服务
2. 多个 worker 进程可以提高服务的 **健壮性** (一个挂了其它还能工作)，并且充分利用了 SMP 多核架构，实现真正的多核并行

Nginx 把 worker 进程数设置为与 CPU 核心数相同的原因：Apache 每个进程对应一个连接，并发量提高后，就有了大量的进程，上下文切换代价高；Nginx 的一个 worker 进程同时可以处理的请求数 **理论上仅受限于内存大小**，worker 进程通常不会进入睡眠状态。因此，worker 进程数与 CPU core 数量相等时 (且每个进程绑定一个 core)，上下文切换代价最小。

---

## 2.2 Nginx 配置项语法

### 2.2.1 块配置项

```nginx
events {

}

http {
    upstream backend {
        server 127.0.0.1:8080;
    }
    gzip on;
    server {
        ...
    }
}
```

### 2.2.2 配置项

```nginx
name var var ...;
```

### 2.2.3 注释

```nginx
# pid logs/nginx.pid;
```

### 2.2.4 单位

- 当指定空间大小时 - `k` / `K` 和 `m` / `M` 分别对应 KB 和 MB
- 当指定时间大小时 - `ms` / `s` / `m` / `h` / `d` / `w` / `M` / `y`

---

## 2.3 Nginx 服务的基本配置

Nginx 运行时必须加载几个核心模块和一个事件类模块。这些模块运行时所使用的配置项称为基本配置项。主要分为四类：

- 用于调试、定位问题的配置项
- 正常运行的必备配置项
- 优化性能的配置项
- 事件类配置项

### 2.3.1 调试、问题定位的配置项

- `daemon on | off;` - 默认以守护进程模式运行，进程不会被任何终端产生的信号打断
- `master_process on | off;` - master 进程是否 fork 出 worker 进程来处理请求
- `error_log /path/file level;` - error 日志的位置和输出级别
- `debug_points [stop | abort]` - 是否处理特殊调试点
- `debug_connection [IP | CIDR]` - 仅对指定客户端输出 debug 级别日志
- `worker_rlimit_core size;` - 限制 coredump 核心转储文件大小
- `working_directory path;` - worker 进程的工作目录，也是 coredump 文件生成的位置

### 2.3.2 正常运行的配置项

- 定义环境变量
- Import 其它配置文件
- Pid 文件路径 (保存 master 进程 pid 的文件路径)
- Worker 进程运行的用户及用户组
- Worker 进程可以打开的最大句柄描述符个数

### 2.3.3. 优化性能的配置项

- Worker 进程个数
- 绑定 worker 进程到 CPU core
- SSL 硬件加速 (如果有)
- Worker 进程优先级 (`nice` 值)

### 2.3.4 事件类的配置项

- `accept_mutex [on | off];` - 是否打开负载均衡锁，可以使多个 worker 进程轮流、序列化地与新的客户端建立连接
- `lock_file path/file;` - 在负载均衡锁开启之后，且 OS 不支持原子锁，那么使用该文件实现负载均衡锁
- `accept_mutex_delay Nms;` - 负载均衡锁的获取是非阻塞的，如果取不到则立刻返回，至少等一段时间以后才能再次获得锁
- `multi_accept [on | off];` - 批量建立新连接
- `use [kqueue | rtsig | epoll | /dev/poll | select | poll | eventport];` - 默认选择性能最高的事件模型
- `worker_connections number;` - 每个 worker 进程可以同时处理的最大连接数

## 2.4 用 HTTP 核心模块配置一个静态 Web 服务器

静态 Web 服务器由 `ngx_http_core_module` 模块实现。

### 2.4.1 虚拟主机与请求的分发

IP 地址数量有限，多个 hostname 可能会映射到同一个 IP 地址。那么可以使用 `server` 块来定义 **虚拟主机**，每个虚拟主机只处理与之相对应的 hostname 请求。这样 Nginx 可以以不同的方式处理访问不同 hostname 的 HTTP 请求了。

- `listen address:port [...];` - 监听端口
- `server_name name [...];` - Nginx 会从 HTTP header 中取出 host，与配置中的 `server_name` 进行匹配
- `server_names_hash_bucket_size` - 为快速寻找 `server_name`，Nginx 用 hash map 存储 `server_name`，此配置用于配置桶大小
- `server_names_hash_max_size` - 影响 hash map 的冲突率与内存占用 (此消彼长)
- `server_name_in_redirect on | off;` - 配合 `server_name` 使用，重定向请求使用其中的第一个 hostname 代替原请求中的 Host header
- `location [] /uri {...}` - 根据用户请求中的 URI 来匹配 (多种匹配方式)，并使用 `{}` 中的逻辑处理请求

### 2.4.2 文件路径的定义

- `root path;` - 定义资源文件相对于 HTTP 请求的根目录
- `alias path;` - 同上，但不包含 `location` 中已配置的路径
- `index file ...;` - 访问首页，按文件依次查看可用性
- `error_page code [] uri | @named_location;` - 根据 HTTP 响应码重定向页面
- `recursive_error_pages [on | off];` - 是否允许递归使用 `error_page`
- `try_files path1 [path2] uri;` - 依次访问文件，直到能够读取一个有效的文件

### 2.4.3 内存及磁盘资源的分配

- `client_body_in_file_only on | clean | off;` - HTTP 报文是否储存在文件中 (请求结束后是否清除)
- `client_body_in_single_buffer on | off;` - HTTP 报文一律存储到内存 buffer 中
- 存储 HTTP header 的内存 buffer 大小
- 存储超大 HTTP header 的内存 buffer 大小
- 存储 HTTP body 的内存 buffer 大小
- HTTP 报文的临时存放目录
- `connection_pool_size` - Nginx 为每个建立成功的 TCP 连接预先分配的内存池的初始大小 (减少内核对小块内存的分配次数)，TCP 连接关闭后销毁
- `request_pool_size` - Nginx 为每个 HTTP 请求分配的内存池初始大小 (请求结束时销毁)

### 2.4.4 网络连接的设置

- 读取 HTTP header 的超时时间
- 读取 HTTP body 的超时时间
- 发送响应的超时时间
- 连接超时后向客户端发送 `RST` 来重置连接 (不使用正常的 TCP 四次握手关闭连接)
- Nginx 关闭用户连接的方式
- Keep-alive 相关

### 2.4.5 MIME 类型的设置

### 2.4.6 对客户端请求的限制

- `limit_except method ... {...}` - 按 HTTP method 来限制用户请求
- HTTP body 的最大值
- 对请求的限速

### 2.4.7 文件操作的优化

- `sendfile on | off;` - 启用 Linux 上的 `sendfile` 系统调用发送文件，减少内核态与用户态之间的两次内存复制 (磁盘直接从内核态到达网卡)
- `aio on | off;` - 是否开启异步文件 I/O (与 `sendfile` 互斥)
- directio (?)
- 打开文件缓存，在内存中存储以下信息，减少对磁盘的操作：
  - 文件句柄、文件大小、上次修改时间
  - 已经打开过的目录
  - 没有找到或没有权限操作的文件信息
- 是否在文件缓存中缓存打开文件错误的信息

### 2.4.8 对客户端请求的特殊处理

- 不合法的 HTTP header
- DNS
- ...

---

## 2.5 用 HTTP Proxy Module 配置一个反向代理服务器

反向代理 (Reverse Proxy) 指用 **代理服务器** 来接收 Internet 上的请求，然后转发给 **内部网络** 中的 **上游服务器**，并将上游服务器的结果返回给 Internet 上的客户端。由于 Nginx 能够处理大量并发请求，因此经常被充当为反向代理。

Nginx 的高并发能力使得其可以作为前端服务器直接向客户端提供 **静态文件服务**。有一些复杂、多变的业务不适合放到 Nginx 上，需要 Apache、Tomcat 等服务器来处理。Nginx 通常会被配置为既是静态服务器又是反向代理服务器。不适合 Nginx 处理的请求会被转发到上游服务器上。

与其它反向代理服务器 _Squid_ 的区别：

- Nginx 会将客户端的 HTTP 请求完整接收到服务器的内存或磁盘中后，再向上游服务器发起连接
- Squid 一边接收客户端请求，一边转发到上游服务器

Nginx 显然会延长请求的处理时间，增加了服务器上的内存与磁盘空间的消耗量；但是降低了上游服务器的负载，背负了尽可能多的压力。因为客户端到 Nginx 通常走公网，速度慢；而 Nginx 到上游服务器一般走内网，速度快。如果类似 Squid，一收到请求就立刻开始向上游服务器转发，那么上游服务器需要始终维持这个连接，对上游服务器的并发能力提出了挑战。而对 Nginx 来说，转发过程会占用上游服务器尽可能短的时间，降低了上游服务器的并发压力。

### 2.5.1 负载均衡的基本配置

由 `upstream` 块定义一个上游服务器集群，由 `proxy_pass` 使用：

```nginx
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

另外，有时想要使一个用户的请求始终落到同一台上游服务器上，防止同一个用户的数据在多台上游服务器上存在，造成资源的浪费。那么可以使用 `ip_hash` 为用户的 IP 地址进行 hash，然后对上游集群取模，映射到某一台上游服务器上。

### 2.5.2 反向代理的基本配置

- `proxy_pass URL;` - 直接指定 Nginx 代理的上游服务器 URL，或一个 `upstream` 块
- `proxy_method method;` - 根据 HTTP method 来进行转发
- `proxy_hide_header header;` - 指定哪些 HTTP header 不被 Nginx 转发
- `proxy_pass_header header;` - 与上一项相反
- `proxy_pass_request_body on | off` / `proxy_pass_request_headers on | off` - 是否转发 HTTP body / header
- `proxy_redirect [default | off | redirect];` - 重设上游服务器给出的重定向或刷新请求中 HTTP header 中的 `location` 或 `refresh`
- `proxy_next_upstream [error | timeout | ...];` - 如果转发到上游服务器的过程中发生指定错误，那么换一台上游服务器处理请求
