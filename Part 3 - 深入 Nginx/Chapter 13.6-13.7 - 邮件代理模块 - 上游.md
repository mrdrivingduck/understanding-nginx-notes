# Chapter 13.6-13.7 - 邮件代理模块 - 上游

Created by : Mr Dk.

2020 / 08 / 04 12:19

Nanjing, Jiangsu, China

---

## 13.6 与上游邮件服务器间的认证交互

对于 POP3、SMTP、IMAP 协议的邮件服务器来说，与客户端交互的方式都大不相同。Nginx 会先与上游的邮件服务器进行独立的交互，直到邮件服务器认为可以进行到处理阶段，才会开始 **透传** 协议。

### 13.6.1 `ngx_mail_proxy_ctx_t` 结构体

是 `ngx_mail_session_t` 结构体中 `proxy` 成员指向的结构体。其中维护着 Nginx 与上游服务器的连接，以及通信时接收上游消息的缓冲区。

```c
typedef struct {
    ngx_peer_connection_t   upstream; // 与上游服务器的连接
    ngx_buf_t              *buffer; // 接收响应的缓冲区
} ngx_mail_proxy_ctx_t;
```

### 13.6.2 向上游邮件服务器发起连接

`ngx_mail_proxy_init()` 函数可以启动 Nginx 与上游邮件服务器之间的交互：

```c
void
ngx_mail_proxy_init(ngx_mail_session_t *s, ngx_addr_t *peer)
{
    ngx_int_t                  rc;
    ngx_mail_proxy_ctx_t      *p;
    ngx_mail_proxy_conf_t     *pcf;
    ngx_mail_core_srv_conf_t  *cscf;

    s->connection->log->action = "connecting to upstream";

    cscf = ngx_mail_get_module_srv_conf(s, ngx_mail_core_module);

    // 创建 ngx_mail_proxy_ctx_t 结构体
    p = ngx_pcalloc(s->connection->pool, sizeof(ngx_mail_proxy_ctx_t));
    if (p == NULL) {
        ngx_mail_session_internal_server_error(s);
        return;
    }

    // 设置、初始化
    s->proxy = p;

    p->upstream.sockaddr = peer->sockaddr;
    p->upstream.socklen = peer->socklen;
    p->upstream.name = &peer->name;
    p->upstream.get = ngx_event_get_peer;
    p->upstream.log = s->connection->log;
    p->upstream.log_error = NGX_ERROR_ERR;

    // 发起非阻塞的 TCP 连接
    rc = ngx_event_connect_peer(&p->upstream);

    if (rc == NGX_ERROR || rc == NGX_BUSY || rc == NGX_DECLINED) {
        ngx_mail_proxy_internal_server_error(s);
        return;
    }

    // 将连接的读事件添加到定时器中
    ngx_add_timer(p->upstream.connection->read, cscf->timeout);

    p->upstream.connection->data = s;
    p->upstream.connection->pool = s->connection->pool;

    // 下游连接的读事件回调设置为不读取内容 (Nginx 不与下游客户端交互)
    s->connection->read->handler = ngx_mail_proxy_block_read;
    // 上游连接的写事件回调设置为什么事都不做 (不通过事件驱动框架调度)
    p->upstream.connection->write->handler = ngx_mail_proxy_dummy_handler;

    pcf = ngx_mail_get_module_srv_conf(s, ngx_mail_proxy_module);

    // 建立 Nginx 与邮件服务器间的内存缓冲区
    s->proxy->buffer = ngx_create_temp_buf(s->connection->pool,
                                           pcf->buffer_size);
    if (s->proxy->buffer == NULL) {
        ngx_mail_proxy_internal_server_error(s);
        return;
    }

    s->out.len = 0;

    // 根据用户请求的协议设置上游连接的读事件回调函数
    switch (s->protocol) {

    case NGX_MAIL_POP3_PROTOCOL:
        p->upstream.connection->read->handler = ngx_mail_proxy_pop3_handler;
        s->mail_state = ngx_pop3_start;
        break;

    case NGX_MAIL_IMAP_PROTOCOL:
        p->upstream.connection->read->handler = ngx_mail_proxy_imap_handler;
        s->mail_state = ngx_imap_start;
        break;

    default: /* NGX_MAIL_SMTP_PROTOCOL */
        p->upstream.connection->read->handler = ngx_mail_proxy_smtp_handler;
        s->mail_state = ngx_smtp_start;
        break;
    }
}
```

可以看到，与上游服务器的交互过程中，写事件回调 (即发送请求) 不由事件驱动模块触发，读事件回调 (接收响应) 根据具体的邮件协议而不同。为什么发送请求不由事件驱动模块触发呢？因为邮件协议中的交互是一来一回的，所以可以在读事件回调中发送请求。

### 13.6.3 与邮件服务器认证交互

每种邮件协议服务器与 Nginx 的交互内容各不相同，以 POP3 协议为例：

```c
static void
ngx_mail_proxy_pop3_handler(ngx_event_t *rev)
{
    u_char                 *p;
    ngx_int_t               rc;
    ngx_str_t               line;
    ngx_connection_t       *c;
    ngx_mail_session_t     *s;
    ngx_mail_proxy_conf_t  *pcf;

    ngx_log_debug0(NGX_LOG_DEBUG_MAIL, rev->log, 0,
                   "mail proxy pop3 auth handler");

    // Nginx 与上游服务器的连接
    c = rev->data;
    // Nginx 的 ngx_mail_session_t 结构体
    s = c->data;

    // 读取上游邮件服务器响应超时
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT,
                      "upstream timed out");
        c->timedout = 1;
        ngx_mail_proxy_internal_server_error(s);
        return;
    }

    // 读取响应到缓冲区中
    rc = ngx_mail_proxy_read_response(s, 0);

    // 没有读取完整，等待被再次调度
    if (rc == NGX_AGAIN) {
        return;
    }

    // 消息不合法
    if (rc == NGX_ERROR) {
        ngx_mail_proxy_upstream_error(s);
        return;
    }

    // 根据当前所处的状态进行不同处理
    switch (s->mail_state) {

    // 构造用户信息发送给邮件服务器
    case ngx_pop3_start:
        ngx_log_debug0(NGX_LOG_DEBUG_MAIL, rev->log, 0, "mail proxy send user");

        s->connection->log->action = "sending user name to upstream";

        line.len = sizeof("USER ")  - 1 + s->login.len + 2;
        line.data = ngx_pnalloc(c->pool, line.len);
        if (line.data == NULL) {
            ngx_mail_proxy_internal_server_error(s);
            return;
        }

        p = ngx_cpymem(line.data, "USER ", sizeof("USER ") - 1);
        p = ngx_cpymem(p, s->login.data, s->login.len);
        *p++ = CR; *p = LF;

        s->mail_state = ngx_pop3_user;
        break;

    // 构造用户密码发送给邮件服务器
    case ngx_pop3_user:
        ngx_log_debug0(NGX_LOG_DEBUG_MAIL, rev->log, 0, "mail proxy send pass");

        s->connection->log->action = "sending password to upstream";

        line.len = sizeof("PASS ")  - 1 + s->passwd.len + 2;
        line.data = ngx_pnalloc(c->pool, line.len);
        if (line.data == NULL) {
            ngx_mail_proxy_internal_server_error(s);
            return;
        }

        p = ngx_cpymem(line.data, "PASS ", sizeof("PASS ") - 1);
        p = ngx_cpymem(p, s->passwd.data, s->passwd.len);
        *p++ = CR; *p = LF;

        s->mail_state = ngx_pop3_passwd;
        break;

    // 与邮件服务器的用户名、密码认证通过
    case ngx_pop3_passwd:
        // 将 Nginx 与上游、下游连接的读写事件回调全部设置为 ngx_mail_proxy_handler
        s->connection->read->handler = ngx_mail_proxy_handler;
        s->connection->write->handler = ngx_mail_proxy_handler;
        rev->handler = ngx_mail_proxy_handler;
        c->write->handler = ngx_mail_proxy_handler;

        pcf = ngx_mail_get_module_srv_conf(s, ngx_mail_proxy_module);
        ngx_add_timer(s->connection->read, pcf->timeout);
        ngx_del_timer(c->read);

        c->log->action = NULL;
        ngx_log_error(NGX_LOG_INFO, c->log, 0, "client logged in");

        // 对下游连接的写事件调用回调函数，开始透传
        ngx_mail_proxy_handler(s->connection->write);

        return;

    default:
#if (NGX_SUPPRESS_WARN)
        ngx_str_null(&line);
#endif
        break;
    }

    // 向上游邮件服务器发送验证消息
    // 当前函数是 Nginx 与上游服务器连接的读事件回调函数
    // 因此发送验证消息不由事件驱动模块触发，而是由上游连接的读事件触发
    // 因为只有接收到了邮件服务器的消息，才会向邮件服务器发送消息，且发送的消息非常短小
    if (c->send(c, line.data, line.len) < (ssize_t) line.len) {
        /*
         * we treat the incomplete sending as NGX_ERROR
         * because it is very strange here
         */
        ngx_mail_proxy_internal_server_error(s);
        return;
    }

    // 设置 buffer 缓冲区指针 (相当于清空缓冲区)
    s->proxy->buffer->pos = s->proxy->buffer->start;
    s->proxy->buffer->last = s->proxy->buffer->start;
}
```

## 13.7 透传上游邮件服务器与客户端间的流

上面函数中，在开始透传后，会将 Nginx 与上下游 TCP 连接的读写事件回调函数都设置为 `ngx_mail_proxy_handler()`。这个函数将同时处理四个事件，实现透传。这里使用了 **固定大小的缓存** 来实现透传功能。

其中，`ngx_mail_session_t` 中的 `buffer` 缓冲区用于从下游客户端到上游服务器的透传；而 `ngx_mail_proxy_ctx_t` 中的 `buffer` 缓冲区用于从上游服务器透传到下游客户端。在这两个 `ngx_buf_t` 类型的缓冲区中，`pos` 指向待转发消息的起始位置，`last` 指向消息的末尾 - 当 `pos` 与 `last` 相等时，全部缓存消息就发送完了。

```c
static void
ngx_mail_proxy_handler(ngx_event_t *ev)
{
    char                   *action, *recv_action, *send_action;
    size_t                  size;
    ssize_t                 n;
    ngx_buf_t              *b;
    ngx_uint_t              do_write;
    // src 和 dst 指针用于代码重用
    // 不管谁是上游、下游，src 指向转发消息的来源，dst 指向转发消息的目标
    ngx_connection_t       *c, *src, *dst;
    ngx_mail_session_t     *s;
    ngx_mail_proxy_conf_t  *pcf;

    // 连接 (可能是上游也可能是下游)
    c = ev->data;
    // 指向 ngx_mail_session_t 结构体 (无论上游下游)
    s = c->data;

    // 事件超时，终止透传
    if (ev->timedout || c->close) {
        c->log->action = "proxying";

        if (c->close) {
            ngx_log_error(NGX_LOG_INFO, c->log, 0, "shutdown timeout");

        } else if (c == s->connection) {
            ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT,
                          "client timed out");
            c->timedout = 1;

        } else {
            ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT,
                          "upstream timed out");
        }

        ngx_mail_proxy_close_session(s);
        return;
    }

    // s->connection 一定指向 Nginx 与下游的 TCP 连接
    if (c == s->connection) {
        // 收到了下游连接上的事件
        
        if (ev->write) {
            // 下游可写事件
            recv_action = "proxying and reading from upstream";
            send_action = "proxying and sending to client";
            // src 为上游连接
            src = s->proxy->upstream.connection;
            // dst 为下游连接
            dst = c;
            // 设置用于向下游转发的缓冲区
            b = s->proxy->buffer;

        } else {
            // 下游可读事件
            recv_action = "proxying and reading from client";
            send_action = "proxying and sending to upstream";
            // src 为下游连接
            src = c;
            // dst 为上游连接
            dst = s->proxy->upstream.connection;
            // 设置用于向上游转发的缓冲区
            b = s->buffer;
        }

    } else {
        // 收到了上游连接上的事件
        
        if (ev->write) {
            // 上游连接可写
            recv_action = "proxying and reading from client";
            send_action = "proxying and sending to upstream";
            // src 为下游连接
            src = s->connection;
            // dst 为上游连接
            dst = c;
            // 设置向上游转发的缓冲区
            b = s->buffer;

        } else {
            // 上游连接可读
            recv_action = "proxying and reading from upstream";
            send_action = "proxying and sending to client";
            // src 为上游连接
            src = c;
            // dst 为下游连接
            dst = s->connection;
            // 设置向下游转发的缓冲区
            b = s->proxy->buffer;
        }
    }

    // 决定本轮是读还是写
    do_write = ev->write ? 1 : 0;

    ngx_log_debug3(NGX_LOG_DEBUG_MAIL, ev->log, 0,
                   "mail proxy handler: %ui, #%d > #%d",
                   do_write, src->fd, dst->fd);

    // 进入循环
    for ( ;; ) {

        // 本轮是写操作
        if (do_write) {

            // 计算缓冲区中要写的字节数
            size = b->last - b->pos;

            // 如果有字节要写，且写事件就绪
            if (size && dst->write->ready) {
                c->log->action = send_action;

                // 发送字节
                n = dst->send(dst, b->pos, size);

                if (n == NGX_ERROR) {
                    ngx_mail_proxy_close_session(s);
                    return;
                }

                // 如果有字节成功发送
                if (n > 0) {
                    // 更新缓冲区指针
                    b->pos += n;

                    // 缓冲区中的内容被完全发送，则清空缓冲区，复用
                    if (b->pos == b->last) {
                        b->pos = b->start;
                        b->last = b->start;
                    }
                }
            }
        }

        // 计算缓冲区中的剩余空间
        size = b->end - b->last;

        // 如果还有剩余空间，且读事件就绪
        if (size && src->read->ready) {
            c->log->action = recv_action;

            // 从读事件中获取字节
            n = src->recv(src, b->last, size);

            // 没有读取到内容，则跳出
            if (n == NGX_AGAIN || n == 0) {
                break;
            }

            // 如果读取到内容
            if (n > 0) {
                // 试图直接发送读取到的内容
                do_write = 1;
                // 更新缓冲区指针
                b->last += n;

                continue;
            }

            if (n == NGX_ERROR) {
                src->read->eof = 1;
            }
        }

        break;
    }

    c->log->action = "proxying";

    // 如果上下游连接中断，则结束透传
    if ((s->connection->read->eof && s->buffer->pos == s->buffer->last)
        || (s->proxy->upstream.connection->read->eof
            && s->proxy->buffer->pos == s->proxy->buffer->last)
        || (s->connection->read->eof
            && s->proxy->upstream.connection->read->eof))
    {
        action = c->log->action;
        c->log->action = NULL;
        ngx_log_error(NGX_LOG_INFO, c->log, 0, "proxied session done");
        c->log->action = action;

        ngx_mail_proxy_close_session(s);
        return;
    }
    
    // 将四个事件再次加入到事件驱动模块中

    if (ngx_handle_write_event(dst->write, 0) != NGX_OK) {
        ngx_mail_proxy_close_session(s);
        return;
    }

    if (ngx_handle_read_event(dst->read, 0) != NGX_OK) {
        ngx_mail_proxy_close_session(s);
        return;
    }

    if (ngx_handle_write_event(src->write, 0) != NGX_OK) {
        ngx_mail_proxy_close_session(s);
        return;
    }

    if (ngx_handle_read_event(src->read, 0) != NGX_OK) {
        ngx_mail_proxy_close_session(s);
        return;
    }

    // 将下游客户端的读事件添加到定时器中
    // 防止僵死的客户端占用 Nginx 服务器资源
    if (c == s->connection) {
        pcf = ngx_mail_get_module_srv_conf(s, ngx_mail_proxy_module);
        ngx_add_timer(c->read, pcf->timeout);
    }
}
```

---

