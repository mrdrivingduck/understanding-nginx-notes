# Chapter 13.1-13.5 - 邮件代理模块 - 认证服务器

Created by : Mr Dk.

2020 / 08 / 04 10:22

Nanjing, Jiangsu, China

---

## 13.1 邮件代理服务器的功能

邮件代理服务器 **不提供实际的邮件服务器功能**，只负责把客户端的请求代理到上游的邮件服务器中。

- 认证 - 只有经过认证服务器的认证后，Nginx 才会向上游邮件服务器发起通信请求
- 透传

Nginx 与下游客户端、上游邮件服务器之间都是使用邮件协议，而与认证服务器之间使用的是类 HTTP 协议。Nginx 邮件模块的目的是与上游邮件服务器之间透传 TCP 流。在邮件模块的配置中，直属于 `mail{}` 块的配置称为 main 级别配置，直属于 `server{}` 块下的配置则被称为 srv 配置。

## 13.2 邮件模块的处理框架

与 HTTP 处理类似，Nginx 把邮件请求的处理过程分为八个阶段 (把相同代码可能会被多次调用的过程划分为一个阶段)：

1. Nginx 与客户端建立 TCP 连接，回调 `ngx_mail_init_connection()` 初始化将要用到的数据结构
2. Nginx 接收、解析客户端请求
3. Nginx 与认证服务器建立 TCP 连接
4. Nginx 向认证服务器发起类 HTTP 请求
5. Nginx 接收认证服务器的响应并判断是否合法，并获得上游邮件服务器的地址
6. Nginx 向上游邮件服务器发起 TCP 连接
7. Nginx 与邮件服务器使用 POP3 / SMTP / IMAP 协议交互认证
8. Nginx 在客户端与邮件服务器之间透传协议

邮件模块的定义与 HTTP 模块类似。首先，有一个 `ngx_mail_module` 的核心模块，在其中定义了新的模块类型 `NGX_MAIL_MODULE`。这类模块的 `ctx` 成员指向的通用抽象接口为 `ngx_mail_module_t`：

```c
typedef struct {
    // POP3 / SMTP / IMAP 邮件模块提取出的通用接口
    ngx_mail_protocol_t        *protocol;

    // 建立存储 main 级别配置项的结构体
    void                       *(*create_main_conf)(ngx_conf_t *cf);
    // 解析完 main 级别配置项后回调
    char                       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    // 建立存储 srv 级别配置项的结构体
    void                       *(*create_srv_conf)(ngx_conf_t *cf);
    // 合并 main 级别与 srv 级别的配置项
    char                       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev,
                                                  void *conf);
} ngx_mail_module_t;
```

每一个邮件模块都会实现上述接口。而 `ngx_mail_protocol_t` 结构体的定义如下：

```c
typedef struct ngx_mail_protocol_s  ngx_mail_protocol_t;

struct ngx_mail_protocol_s {
    ngx_str_t                   name; // 邮件模块名称
    in_port_t                   port[4]; // 邮件模块最常监听的四个端口
    ngx_uint_t                  type; // 邮件模块类型 (POP3 / SMTP / IMAP)

    // 与客户端建立连接后的初始化函数
    ngx_mail_init_session_pt    init_session;
    // 接收、解析客户端请求的函数
    ngx_mail_init_protocol_pt   init_protocol;
    // 解析客户端邮件协议接口的函数 (由各邮件模块实现)
    ngx_mail_parse_command_pt   parse_command;
    // 认证客户端请求的函数
    ngx_mail_auth_state_pt      auth_state;

    // 发生一些错误时返回客户端的字符串
    ngx_str_t                   internal_server_error;
    ngx_str_t                   cert_error;
    ngx_str_t                   no_cert;
};
```

## 13.3 初始化请求

Nginx 与客户端建立 TCP 连接后，回调 `ngx_mail_init_connection()` 函数，初始化邮件协议 - 这时会建立一个会话结构体 `ngx_mail_session_t`：

```c
typedef struct {
    uint32_t                signature;         /* "MAIL" */

    // 下游客户端与 Nginx 之间的连接
    ngx_connection_t       *connection;

    // 存放需要向下游客户端发送的内容
    ngx_str_t               out;
    // 接收客户端请求的缓冲区
    ngx_buf_t              *buffer;

    // 各个邮件模块的上下文指针
    void                  **ctx;
    // main 级别配置项
    void                  **main_conf;
    // srv 级别配置项
    void                  **srv_conf;

    // 解析主机域名
    ngx_resolver_ctx_t     *resolver_ctx;

    // 经过认证后，proxy 用于代理客户端与邮件服务器之间的通信
    ngx_mail_proxy_ctx_t   *proxy;

    // 与邮件服务器交互时的处理状态
    ngx_uint_t              mail_state;

    // 标志位
    unsigned                protocol:3;
    unsigned                blocked:1;
    unsigned                quit:1;
    unsigned                quoted:1;
    unsigned                backslash:1;
    unsigned                no_sync_literal:1;
    unsigned                starttls:1;
    unsigned                esmtp:1;
    unsigned                auth_method:3;
    unsigned                auth_wait:1;

    // 用于与认证服务器认证的用户名和密码
    ngx_str_t               login;
    ngx_str_t               passwd;

    ngx_str_t               salt;
    ngx_str_t               tag;
    ngx_str_t               tagged_line;
    ngx_str_t               text;

    // 当前连接上对应的 Nginx 服务器地址
    ngx_str_t              *addr_text;
    // 主机地址
    ngx_str_t               host;

    ngx_str_t               smtp_helo;
    ngx_str_t               smtp_from;
    ngx_str_t               smtp_to;

    ngx_str_t               cmd;

    ngx_uint_t              command;

    // 存放来自下游客户端邮件协议的参数
    ngx_array_t             args;

    // 当前请求尝试访问认证服务器的次数
    ngx_uint_t              login_attempt;

    /* used to parse POP3/IMAP/SMTP command */

    // 解析邮件协议的命令行
    ngx_uint_t              state;
    u_char                 *cmd_start;
    u_char                 *arg_start;
    u_char                 *arg_end;
    ngx_uint_t              literal_len;
} ngx_mail_session_t;
```

```c
void
ngx_mail_init_connection(ngx_connection_t *c)
{
    size_t                     len;
    ngx_uint_t                 i;
    ngx_mail_port_t           *port;
    struct sockaddr           *sa;
    struct sockaddr_in        *sin;
    ngx_mail_log_ctx_t        *ctx;
    ngx_mail_in_addr_t        *addr;
    ngx_mail_session_t        *s;
    ngx_mail_addr_conf_t      *addr_conf;
    ngx_mail_core_srv_conf_t  *cscf;
    u_char                     text[NGX_SOCKADDR_STRLEN];
#if (NGX_HAVE_INET6)
    struct sockaddr_in6       *sin6;
    ngx_mail_in6_addr_t       *addr6;
#endif


    /* find the server configuration for the address:port */

    port = c->listening->servers;

    if (port->naddrs > 1) {

        /*
         * There are several addresses on this port and one of them
         * is the "*:port" wildcard so getsockname() is needed to determine
         * the server address.
         *
         * AcceptEx() already gave this address.
         */

        if (ngx_connection_local_sockaddr(c, NULL, 0) != NGX_OK) {
            ngx_mail_close_connection(c);
            return;
        }

        sa = c->local_sockaddr;

        switch (sa->sa_family) {

        default: /* AF_INET */
            sin = (struct sockaddr_in *) sa;

            addr = port->addrs;

            /* the last address is "*" */

            for (i = 0; i < port->naddrs - 1; i++) {
                if (addr[i].addr == sin->sin_addr.s_addr) {
                    break;
                }
            }

            addr_conf = &addr[i].conf;

            break;
        }

    } else {
        switch (c->local_sockaddr->sa_family) {

        default: /* AF_INET */
            addr = port->addrs;
            addr_conf = &addr[0].conf;
            break;
        }
    }

    // 分配 ngx_mail_session_t 结构体
    s = ngx_pcalloc(c->pool, sizeof(ngx_mail_session_t));
    if (s == NULL) {
        ngx_mail_close_connection(c);
        return;
    }

    // 初始化
    s->signature = NGX_MAIL_MODULE;

    s->main_conf = addr_conf->ctx->main_conf;
    s->srv_conf = addr_conf->ctx->srv_conf;

    s->addr_text = &addr_conf->addr_text;

    c->data = s;
    s->connection = c;

    cscf = ngx_mail_get_module_srv_conf(s, ngx_mail_core_module);

    ngx_set_connection_log(c, cscf->error_log);

    len = ngx_sock_ntop(c->sockaddr, c->socklen, text, NGX_SOCKADDR_STRLEN, 1);

    ngx_log_error(NGX_LOG_INFO, c->log, 0, "*%uA client %*s connected to %V",
                  c->number, len, text, s->addr_text);

    ctx = ngx_palloc(c->pool, sizeof(ngx_mail_log_ctx_t));
    if (ctx == NULL) {
        ngx_mail_close_connection(c);
        return;
    }

    ctx->client = &c->addr_text;
    ctx->session = s;

    c->log->connection = c->number;
    c->log->handler = ngx_mail_log_error;
    c->log->data = ctx;
    c->log->action = "sending client greeting line";

    c->log_error = NGX_ERROR_INFO;

#if (NGX_MAIL_SSL)
    {
    ngx_mail_ssl_conf_t  *sslcf;

    sslcf = ngx_mail_get_module_srv_conf(s, ngx_mail_ssl_module);

    if (sslcf->enable || addr_conf->ssl) {
        c->log->action = "SSL handshaking";

        ngx_mail_ssl_init_connection(&sslcf->ssl, c);
        return;
    }

    }
#endif

    // 开始调用各个协议模块实现的解析请求函数
    ngx_mail_init_session(c);
}
```

## 13.4 接收并解析客户端请求

POP3、SMTP 和 IMAP 模块实现的 `init_session()` 中都会调用各自的 `init_protocol()` 函数，接收、解析客户端请求。流程是相似的：

- 反复接收客户端请求，使用状态机解析是否收到了足够的信息
- 接收到完整的信息后，进入邮件认证阶段

```c
static void
ngx_mail_init_session(ngx_connection_t *c)
{
    ngx_mail_session_t        *s;
    ngx_mail_core_srv_conf_t  *cscf;

    s = c->data;

    cscf = ngx_mail_get_module_srv_conf(s, ngx_mail_core_module);

    s->protocol = cscf->protocol->type;

    s->ctx = ngx_pcalloc(c->pool, sizeof(void *) * ngx_mail_max_module);
    if (s->ctx == NULL) {
        ngx_mail_session_internal_server_error(s);
        return;
    }

    c->write->handler = ngx_mail_send;

    // 调用模块的 init_session() 函数
    cscf->protocol->init_session(s, c);
}
```

## 13.5 邮件认证

### 13.5.1 `ngx_mail_auth_http_ctx_t` 结构体

邮件认证服务器的地址在配置文件的 `auth_http` 中进行配置，由 `ngx_mail_auth_http_module` 模块完成这个功能。该模块会在 `ngx_mail_session_t` 结构体的 `ctx` 中拥有一个上下文结构体：

```c
typedef struct ngx_mail_auth_http_ctx_s  ngx_mail_auth_http_ctx_t;

struct ngx_mail_auth_http_ctx_s {
    // 缓冲区保存发往认证服务器的请求
    ngx_buf_t                      *request;
    // 缓冲区保存认证服务器的响应
    ngx_buf_t                      *response;
    // Nginx 与认证服务器之间的连接
    ngx_peer_connection_t           peer;

    // 解析来自认证服务器响应的回调函数
    ngx_mail_auth_http_handler_pt   handler;

    // 解析认证服务器响应时，state 表示解析状态
    ngx_uint_t                      state;

    // 用于解析响应 header
    u_char                         *header_name_start;
    u_char                         *header_name_end;
    u_char                         *header_start;
    u_char                         *header_end;

    ngx_str_t                       addr; // Auth-Server header
    ngx_str_t                       port; // Auth-Port header
    ngx_str_t                       err; // 错误信息
    ngx_str_t                       errmsg;
    ngx_str_t                       errcode;

    // Auth-Wait header 所带的时间戳，是 Nginx 的等待时间
    time_t                          sleep;

    // 用于邮件认证的独立内存池
    ngx_pool_t                     *pool;
};
```

### 13.5.2 与认证服务器建立连接

首先，Nginx 与认证服务器建立 TCP 连接：

```c
void
ngx_mail_auth(ngx_mail_session_t *s, ngx_connection_t *c)
{
    s->args.nelts = 0;

    if (s->buffer->pos == s->buffer->last) {
        s->buffer->pos = s->buffer->start;
        s->buffer->last = s->buffer->start;
    }

    s->state = 0;

    // 将下游连接读事件从定时器中取出 (因为不再需要与下游客户端交互了)
    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }

    s->login_attempt++;

    ngx_mail_auth_http_init(s);
}

void
ngx_mail_auth_http_init(ngx_mail_session_t *s)
{
    ngx_int_t                   rc;
    ngx_pool_t                 *pool;
    ngx_mail_auth_http_ctx_t   *ctx;
    ngx_mail_auth_http_conf_t  *ahcf;

    s->connection->log->action = "in http auth state";

    // 分配内存池
    pool = ngx_create_pool(2048, s->connection->log);
    if (pool == NULL) {
        ngx_mail_session_internal_server_error(s);
        return;
    }

    // 分配上下文结构体
    ctx = ngx_pcalloc(pool, sizeof(ngx_mail_auth_http_ctx_t));
    if (ctx == NULL) {
        ngx_destroy_pool(pool);
        ngx_mail_session_internal_server_error(s);
        return;
    }

    ctx->pool = pool;

    ahcf = ngx_mail_get_module_srv_conf(s, ngx_mail_auth_http_module);

    // 发起连接
    ctx->request = ngx_mail_auth_http_create_request(s, pool, ahcf);
    if (ctx->request == NULL) {
        ngx_destroy_pool(ctx->pool);
        ngx_mail_session_internal_server_error(s);
        return;
    }

    // 设置上下文
    ngx_mail_set_ctx(s, ctx, ngx_mail_auth_http_module);

    ctx->peer.sockaddr = ahcf->peer->sockaddr;
    ctx->peer.socklen = ahcf->peer->socklen;
    ctx->peer.name = &ahcf->peer->name;
    ctx->peer.get = ngx_event_get_peer;
    ctx->peer.log = s->connection->log;
    ctx->peer.log_error = NGX_ERROR_ERR;

    // 连接加入事件驱动模块
    rc = ngx_event_connect_peer(&ctx->peer);

    // 连接失败
    if (rc == NGX_ERROR || rc == NGX_BUSY || rc == NGX_DECLINED) {
        if (ctx->peer.connection) {
            ngx_close_connection(ctx->peer.connection);
        }

        ngx_destroy_pool(ctx->pool);
        ngx_mail_session_internal_server_error(s);
        return;
    }

    ctx->peer.connection->data = s;
    ctx->peer.connection->pool = s->connection->pool;

    // 与下游客户端连接的读事件回调函数设置为 ngx_mail_auth_http_block_read
    // 此函数的唯一工作是将下游连接读事件再次加入事件驱动模块
    // 不读取任何客户端请求，只保持读事件被监控
    s->connection->read->handler = ngx_mail_auth_http_block_read;
    // 设置上游连接的读写事件回调函数
    ctx->peer.connection->read->handler = ngx_mail_auth_http_read_handler; // 解析认证服务器响应
    ctx->peer.connection->write->handler = ngx_mail_auth_http_write_handler; // 向认证服务器发送请求

    // 尝试解析认证服务器响应发来的响应行
    ctx->handler = ngx_mail_auth_http_ignore_status_line;

    // 将上游连接的读写事件加入定时器
    ngx_add_timer(ctx->peer.connection->read, ahcf->timeout);
    ngx_add_timer(ctx->peer.connection->write, ahcf->timeout);

    // 如果连接已经建立成功，那么开始向认证服务器发送请求
    if (rc == NGX_OK) {
        ngx_mail_auth_http_write_handler(ctx->peer.connection->write);
        return;
    }
}
```

### 13.5.3 发送请求到认证服务器

上个函数最后调用的 `ngx_mail_auth_http_write_handler()`，同时也是上游连接可写事件的回调函数，负责向认证服务器发送请求：

```c
static void
ngx_mail_auth_http_write_handler(ngx_event_t *wev)
{
    ssize_t                     n, size;
    ngx_connection_t           *c;
    ngx_mail_session_t         *s;
    ngx_mail_auth_http_ctx_t   *ctx;
    ngx_mail_auth_http_conf_t  *ahcf;

    c = wev->data;
    s = c->data;

    ctx = ngx_mail_get_module_ctx(s, ngx_mail_auth_http_module);

    ngx_log_debug0(NGX_LOG_DEBUG_MAIL, wev->log, 0,
                   "mail auth http write handler");

    // 发送请求超时
    if (wev->timedout) {
        ngx_log_error(NGX_LOG_ERR, wev->log, NGX_ETIMEDOUT,
                      "auth http server %V timed out", ctx->peer.name);
        // 关闭连接、销毁内存池
        ngx_close_connection(c);
        ngx_destroy_pool(ctx->pool);
        ngx_mail_session_internal_server_error(s);
        return;
    }

    // 计算剩余要发送的请求的长度
    size = ctx->request->last - ctx->request->pos;

    // 发送请求
    n = ngx_send(c, ctx->request->pos, size);

    // 发送失败
    if (n == NGX_ERROR) {
        // 关闭连接、销毁内存池
        ngx_close_connection(c);
        ngx_destroy_pool(ctx->pool);
        ngx_mail_session_internal_server_error(s);
        return;
    }

    // 成功发送了一部分请求
    if (n > 0) {
        // 缓冲区头部指针后移
        ctx->request->pos += n;

        // 所有请求发送完毕
        if (n == size) {
            // 将写事件设置为什么都不做
            wev->handler = ngx_mail_auth_http_dummy_handler;

            // 不再需要发送请求，因此不再需要监控写事件是否超时
            // 从定时器中移除写事件
            if (wev->timer_set) {
                ngx_del_timer(wev);
            }

            // 为什么写事件还要添加到事件驱动模块中呢？
            if (ngx_handle_write_event(wev, 0) != NGX_OK) {
                ngx_close_connection(c);
                ngx_destroy_pool(ctx->pool);
                ngx_mail_session_internal_server_error(s);
            }

            return;
        }
    }

    // 请求还没有发送完
    // 如果定时器中没有写事件，那么添加到定时器中
    if (!wev->timer_set) {
        ahcf = ngx_mail_get_module_srv_conf(s, ngx_mail_auth_http_module);
        ngx_add_timer(wev, ahcf->timeout);
    }
}
```

### 13.5.4 接收并解析响应

也就是使用 `ngx_mail_auth_http_read_handler()` 同时负责解析响应行和头部。这两个部分都不是一次就能接收完毕的，当没有收到足够的字节时，都会希望事件驱动模块能够再次调用该函数。在响应被完全解析后，可以得知认证是否通过 - 如果请求合法，那么可以从响应中得到上游邮件服务器的地址，然后调用 `ngx_mail_proxy_init()` 函数进入与邮件服务器的交互阶段。

```c
static void
ngx_mail_auth_http_read_handler(ngx_event_t *rev)
{
    ssize_t                     n, size;
    ngx_connection_t          *c;
    ngx_mail_session_t        *s;
    ngx_mail_auth_http_ctx_t  *ctx;

    c = rev->data;
    s = c->data;

    ngx_log_debug0(NGX_LOG_DEBUG_MAIL, rev->log, 0,
                   "mail auth http read handler");

    ctx = ngx_mail_get_module_ctx(s, ngx_mail_auth_http_module);

    // 读事件已经超时
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_ERR, rev->log, NGX_ETIMEDOUT,
                      "auth http server %V timed out", ctx->peer.name);
        // 关闭连接，销毁内存池
        ngx_close_connection(c);
        ngx_destroy_pool(ctx->pool);
        ngx_mail_session_internal_server_error(s);
        return;
    }

    // 分配接收响应的缓冲区
    if (ctx->response == NULL) {
        ctx->response = ngx_create_temp_buf(ctx->pool, 1024);
        if (ctx->response == NULL) {
            ngx_close_connection(c);
            ngx_destroy_pool(ctx->pool);
            ngx_mail_session_internal_server_error(s);
            return;
        }
    }

    // 缓冲区剩余长度
    size = ctx->response->end - ctx->response->last;

    // 接收响应
    n = ngx_recv(c, ctx->response->pos, size);

    // 接收到了部分响应
    if (n > 0) {
        // 更新缓冲区中的可用范围
        ctx->response->last += n;

        // 处理响应
        ctx->handler(s, ctx);
        return;
    }

    // 等待再次被调度
    if (n == NGX_AGAIN) {
        return;
    }

    ngx_close_connection(c);
    ngx_destroy_pool(ctx->pool);
    ngx_mail_session_internal_server_error(s);
}
```
