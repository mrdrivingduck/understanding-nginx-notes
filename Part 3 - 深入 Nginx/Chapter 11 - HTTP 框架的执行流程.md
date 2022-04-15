# Chapter 11 - HTTP 框架的执行流程

Created by : Mr Dk.

2020 / 07 / 31 14:04

Nanjing, Jiangsu, China

---

HTTP 框架如何集成各个 HTTP 模块来共同处理 HTTP 请求？

1. HTTP 框架需要基于 TCP 事件框架解决 HTTP 传输、解析、组装的问题
2. HTTP 框架需要为 HTTP 模块屏蔽事件驱动架构，使其不需关心网络事件处理，又能灵活介入各阶段

HTTP 框架负责与客户端建立 TCP 连接，接收 HTTP request line、HTTP header 并解析，根据配置文件中找到的 HTTP 模块依次合作处理请求。另外，还提供了接收 HTTP body、发送 HTTP response 和派生子请求的工具函数。

## 11.2 新连接建立时的行为

当 Nginx 中的 TCP 连接成功建立后，HTTP 框架就会介入请求处理。在负责 TCP 连接建立的 `ngx_event_accept()` 函数的最后，会调用 `ngx_listening_t` 监听结构体的 `handler` 函数。这个函数在 HTTP 框架初始化时就会被设置为 `ngx_http_init_connection()` 函数，传入的参数为新建立的 `ngx_connection_t` 连接结构体：

```c
void
ngx_http_init_connection(ngx_connection_t *c)
{
    ngx_uint_t              i;
    ngx_event_t            *rev;
    struct sockaddr_in     *sin;
    ngx_http_port_t        *port;
    ngx_http_in_addr_t     *addr;
    ngx_http_log_ctx_t     *ctx;
    ngx_http_connection_t  *hc;
#if (NGX_HAVE_INET6)
    struct sockaddr_in6    *sin6;
    ngx_http_in6_addr_t    *addr6;
#endif

    hc = ngx_pcalloc(c->pool, sizeof(ngx_http_connection_t));
    if (hc == NULL) {
        ngx_http_close_connection(c);
        return;
    }

    c->data = hc;

    /* find the server configuration for the address:port */

    port = c->listening->servers;

    if (port->naddrs > 1) {

        /*
         * there are several addresses on this port and one of them
         * is an "*:port" wildcard so getsockname() in ngx_http_server_addr()
         * is required to determine a server address
         */

        if (ngx_connection_local_sockaddr(c, NULL, 0) != NGX_OK) {
            ngx_http_close_connection(c);
            return;
        }

        switch (c->local_sockaddr->sa_family) {

#if (NGX_HAVE_INET6)
        case AF_INET6:
            sin6 = (struct sockaddr_in6 *) c->local_sockaddr;

            addr6 = port->addrs;

            /* the last address is "*" */

            for (i = 0; i < port->naddrs - 1; i++) {
                if (ngx_memcmp(&addr6[i].addr6, &sin6->sin6_addr, 16) == 0) {
                    break;
                }
            }

            hc->addr_conf = &addr6[i].conf;

            break;
#endif

        default: /* AF_INET */
            sin = (struct sockaddr_in *) c->local_sockaddr;

            addr = port->addrs;

            /* the last address is "*" */

            for (i = 0; i < port->naddrs - 1; i++) {
                if (addr[i].addr == sin->sin_addr.s_addr) {
                    break;
                }
            }

            hc->addr_conf = &addr[i].conf;

            break;
        }

    } else {

        switch (c->local_sockaddr->sa_family) {

#if (NGX_HAVE_INET6)
        case AF_INET6:
            addr6 = port->addrs;
            hc->addr_conf = &addr6[0].conf;
            break;
#endif

        default: /* AF_INET */
            addr = port->addrs;
            hc->addr_conf = &addr[0].conf;
            break;
        }
    }

    /* the default server configuration for the address:port */
    hc->conf_ctx = hc->addr_conf->default_server->ctx;

    ctx = ngx_palloc(c->pool, sizeof(ngx_http_log_ctx_t));
    if (ctx == NULL) {
        ngx_http_close_connection(c);
        return;
    }

    ctx->connection = c;
    ctx->request = NULL;
    ctx->current_request = NULL;

    c->log->connection = c->number;
    c->log->handler = ngx_http_log_error;
    c->log->data = ctx;
    c->log->action = "waiting for request";

    c->log_error = NGX_ERROR_INFO;

    rev = c->read;
    // 设置新连接的可读事件处理函数为 ngx_http_wait_request_handler
    // TCP 连接上有数据到达后，将调用这个函数初始化 HTTP request
    rev->handler = ngx_http_wait_request_handler;
    // 设置新连接的可写事件处理函数为 ngx_http_empty_handler (不做任何工作)
    c->write->handler = ngx_http_empty_handler;

#if (NGX_HTTP_V2)
    if (hc->addr_conf->http2) {
        rev->handler = ngx_http_v2_init;
    }
#endif

#if (NGX_HTTP_SSL)
    {
    ngx_http_ssl_srv_conf_t  *sscf;

    sscf = ngx_http_get_module_srv_conf(hc->conf_ctx, ngx_http_ssl_module);

    if (sscf->enable || hc->addr_conf->ssl) {
        hc->ssl = 1;
        c->log->action = "SSL handshaking";
        rev->handler = ngx_http_ssl_handshake;
    }
    }
#endif

    if (hc->addr_conf->proxy_protocol) {
        hc->proxy_protocol = 1;
        c->log->action = "reading PROXY protocol";
    }

    // 如果新连接上已经有用户数据到达了
    // 那么直接调用刚才设置的 ngx_http_init_request()
    if (rev->ready) {
        /* the deferred accept(), iocp */

        if (ngx_use_accept_mutex) {
            ngx_post_event(rev, &ngx_posted_events);
            return;
        }

        rev->handler(rev);
        return;
    }

    // 将读事件添加到定时器中
    // 如果超时后还没有用户事件发来，那么由定时器来触发调用 ngx_http_init_request()，可能会关闭连接
    ngx_add_timer(rev, c->listening->post_accept_timeout);
    ngx_reusable_connection(c, 1);

    // 将连接的可读事件添加到 EPOLL 事件驱动模块中
    if (ngx_handle_read_event(rev, 0) != NGX_OK) {
        ngx_http_close_connection(c);
        return;
    }
}
```

## 11.3 第一次可读事件的处理

HTTP 框架不会在 TCP 连接建立成功后就立刻初始化 HTTP 请求，而是在 TCP 读缓冲区中第一次接收到用户数据时才进行。这个设计体现了 Nginx 追求高性能的考虑，降低了一个请求占用内存资源的时间。所以 Nginx 将新连接的可读事件回调函数设置为 `ngx_http_init_request()`，并添加到了定时器和事件驱动模块中；另外，可能在新连接建立的同时，用户的数据也已经发到 TCP 缓冲区中了。综上，有三种情况会触发 `ngx_http_init_request()`：

1. 新连接建立后，用户数据已经发送到了 TCP 缓冲区中，那么 HTTP 框架会直接调用回调函数
2. 一段时间后，用户数据发送到了 TCP 缓冲区中，由 EPOLL 等事件驱动模块通知 HTTP 框架调用回调函数
3. 超时时间后，由定时器触发这个回调函数

在这个回调函数中，主要完成三个工作：

1. 构造代表 HTTP 请求的 `ngx_http_request_t` 结构体并初始化部分参数
2. 由于 HTTP 请求已经开始处理流程，需要修改读事件回调函数为 `ngx_http_process_request_line`
3. 调用回调函数，准备开始处理 HTTP request line

```c
static void
ngx_http_wait_request_handler(ngx_event_t *rev)
{
    u_char                    *p;
    size_t                     size;
    ssize_t                    n;
    ngx_buf_t                 *b;
    ngx_connection_t          *c;
    ngx_http_connection_t     *hc;
    ngx_http_core_srv_conf_t  *cscf;

    c = rev->data;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0, "http wait request handler");

    // 首先检查这个函数的调用是否是因为超时，即超时时间内都没有收到用户数据
    // 如果超时，那么不再处理这个请求，关闭请求
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        ngx_http_close_connection(c);
        return;
    }

    if (c->close) {
        ngx_http_close_connection(c);
        return;
    }

    // 获取连接，及其对应的 server{} 块配置项
    hc = c->data;
    cscf = ngx_http_get_module_srv_conf(hc->conf_ctx, ngx_http_core_module);

    // 下面准备在 TCP 连接的内存池中分配一块 buffer，作为接收数据的用户态缓冲区
    // 缓冲区大小由配置文件指定
    // 在 TCP 连接的内存池中分配，便于之后复用
    size = cscf->client_header_buffer_size;

    b = c->buffer;

    if (b == NULL) {
        // 为 TCP 连接分配内存池
        b = ngx_create_temp_buf(c->pool, size);
        if (b == NULL) {
            ngx_http_close_connection(c);
            return;
        }

        c->buffer = b;

    } else if (b->start == NULL) {
        // 为 HTTP 分配的内存池
        b->start = ngx_palloc(c->pool, size);
        if (b->start == NULL) {
            ngx_http_close_connection(c);
            return;
        }

        b->pos = b->start;
        b->last = b->start;
        b->end = b->last + size;
    }

    // 这里干啥了？
    n = c->recv(c, b->last, size);

    if (n == NGX_AGAIN) {

        if (!rev->timer_set) {
            ngx_add_timer(rev, c->listening->post_accept_timeout);
            ngx_reusable_connection(c, 1);
        }

        if (ngx_handle_read_event(rev, 0) != NGX_OK) {
            ngx_http_close_connection(c);
            return;
        }

        /*
         * We are trying to not hold c->buffer's memory for an idle connection.
         */
        // 只回收 HTTP 连接的内存池
        if (ngx_pfree(c->pool, b->start) == NGX_OK) {
            b->start = NULL;
        }

        return;
    }

    if (n == NGX_ERROR) {
        ngx_http_close_connection(c);
        return;
    }

    if (n == 0) {
        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                      "client closed connection");
        ngx_http_close_connection(c);
        return;
    }

    b->last += n;

    if (hc->proxy_protocol) {
        hc->proxy_protocol = 0;

        p = ngx_proxy_protocol_read(c, b->pos, b->last);

        if (p == NULL) {
            ngx_http_close_connection(c);
            return;
        }

        b->pos = p;

        if (b->pos == b->last) {
            c->log->action = "waiting for request";
            b->pos = b->start;
            b->last = b->start;
            ngx_post_event(rev, &ngx_posted_events);
            return;
        }
    }

    c->log->action = "reading client request line";

    ngx_reusable_connection(c, 0);

    // 初始化一个 HTTP request 结构体，并设置到连接结构体中
    // 在细节上，这里面做了很多工作
    // 接下来，随着 HTTP 解析工作的进行，将不断填充这个结构体中的 field
    c->data = ngx_http_create_request(c);
    if (c->data == NULL) {
        ngx_http_close_connection(c);
        return;
    }

    // 接下来，将要开始处理 HTTP request line，因此重设回调函数并调用
    rev->handler = ngx_http_process_request_line;
    ngx_http_process_request_line(rev);
}
```

## 11.4 接收 HTTP 请求行

HTTP request line 的格式如下：

```http
GET /uri HTTP/1.1
```

因此，请求行的长度是不定的，与 URI 的长度相关。这意味着内核中 TCP 读缓冲区的大小不一定能够一次接收到完整的 HTTP request line，由此，调用一次 `ngx_http_process_request_line()` 也不一定能做完解析 HTTP request line 的工作。在上一步中，这个函数被设置为读事件回调函数，可能会被事件驱动模块多次触发，反复接收 TCP 流，并用状态机来解析。直到确认接收了完整的 HTTP request line，这个阶段才算完成。所以这个函数是 **可重入** 的。

```c
static void
ngx_http_process_request_line(ngx_event_t *rev)
{
    ssize_t              n;
    ngx_int_t            rc, rv;
    ngx_str_t            host;
    ngx_connection_t    *c;
    ngx_http_request_t  *r;

    c = rev->data;
    r = c->data;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, rev->log, 0,
                   "http process request line");

    // 判断读事件是否超时
    // 如果超时，则关闭请求
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        c->timedout = 1;
        ngx_http_close_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }

    rc = NGX_AGAIN;

    for ( ;; ) {

        // 从 Linux 内核的 socket 缓冲区中复制到 header_in 缓冲区中
        if (rc == NGX_AGAIN) {
            n = ngx_http_read_request_header(r);

            // 如果本次没有新数据，则结束函数，等待下次再次被触发
            if (n == NGX_AGAIN || n == NGX_ERROR) {
                break;
            }
        }

        // 用状态机解析已经接收到的 TCP 字符流
        rc = ngx_http_parse_request_line(r, r->header_in);

        // 成功解析 request line
        if (rc == NGX_OK) {
            /* the request line has been parsed successfully */

            // 设置请求结构体中的相关信息
            r->request_line.len = r->request_end - r->request_start;
            r->request_line.data = r->request_start;
            r->request_length = r->header_in->pos - r->request_start;

            ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0,
                           "http request line: \"%V\"", &r->request_line);

            r->method_name.len = r->method_end - r->request_start + 1;
            r->method_name.data = r->request_line.data;

            if (r->http_protocol.data) {
                r->http_protocol.len = r->request_end - r->http_protocol.data;
            }

            if (ngx_http_process_request_uri(r) != NGX_OK) {
                break;
            }

            if (r->schema_end) {
                r->schema.len = r->schema_end - r->schema_start;
                r->schema.data = r->schema_start;
            }

            if (r->host_end) {

                host.len = r->host_end - r->host_start;
                host.data = r->host_start;

                rc = ngx_http_validate_host(&host, r->pool, 0);

                if (rc == NGX_DECLINED) {
                    ngx_log_error(NGX_LOG_INFO, c->log, 0,
                                  "client sent invalid host in request line");
                    ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
                    break;
                }

                if (rc == NGX_ERROR) {
                    ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                    break;
                }

                if (ngx_http_set_virtual_server(r, &host) == NGX_ERROR) {
                    break;
                }

                r->headers_in.server = host;
            }

            // HTTP 版本小于 1.0，那么不接收请求头，直接开始处理请求
            if (r->http_version < NGX_HTTP_VERSION_10) {

                if (r->headers_in.server.len == 0
                    && ngx_http_set_virtual_server(r, &r->headers_in.server)
                       == NGX_ERROR)
                {
                    break;
                }

                ngx_http_process_request(r);
                break;
            }

            // 初始化用于存放 HTTP headers 的链表，为下一步接收 HTTP headers 做准备
            if (ngx_list_init(&r->headers_in.headers, r->pool, 20,
                              sizeof(ngx_table_elt_t))
                != NGX_OK)
            {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                break;
            }

            c->log->action = "reading client request headers";

            // 将读事件回调函数设置为处理 HTTP header，然后调用这个函数
            rev->handler = ngx_http_process_request_headers;
            ngx_http_process_request_headers(rev);

            break;
        }

        // 非法 request line
        if (rc != NGX_AGAIN) {
            /* there was error while a request line parsing */

            ngx_log_error(NGX_LOG_INFO, c->log, 0,
                          ngx_http_client_errors[rc - NGX_HTTP_CLIENT_ERROR]);

            if (rc == NGX_HTTP_PARSE_INVALID_VERSION) {
                ngx_http_finalize_request(r, NGX_HTTP_VERSION_NOT_SUPPORTED);

            } else {
                ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
            }

            break;
        }

        /* NGX_AGAIN: a request line parsing is still incomplete */

        // 还需要更多的字符才能进行解析
        // 这里需要判断用户态缓冲区是否还有足够内存
        if (r->header_in->pos == r->header_in->end) {

            // 内存不够，分配更大的缓冲区
            // 分配大小由配置决定
            rv = ngx_http_alloc_large_header_buffer(r, 1);

            // 分配失败就结束请求
            if (rv == NGX_ERROR) {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                break;
            }

            if (rv == NGX_DECLINED) {
                r->request_line.len = r->header_in->end - r->request_start;
                r->request_line.data = r->request_start;

                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                              "client sent too long URI");
                ngx_http_finalize_request(r, NGX_HTTP_REQUEST_URI_TOO_LARGE);
                break;
            }
        }
    }

    ngx_http_run_posted_requests(c);
}
```

其中，从内核中读取字符流的逻辑实现在 `ngx_http_read_reqeust_header()` 中：

```c
static ssize_t
ngx_http_read_request_header(ngx_http_request_t *r)
{
    ssize_t                    n;
    ngx_event_t               *rev;
    ngx_connection_t          *c;
    ngx_http_core_srv_conf_t  *cscf;

    c = r->connection;
    rev = c->read;

    n = r->header_in->last - r->header_in->pos;

    // 缓冲区中还有未解析的字符，直接返回
    if (n > 0) {
        return n;
    }

    // 从内核的 TCP 读缓冲区中复制新数据
    if (rev->ready) {
        n = c->recv(c, r->header_in->last,
                    r->header_in->end - r->header_in->last);
    } else {
        n = NGX_AGAIN;
    }

    // 本次没有新数据
    if (n == NGX_AGAIN) {
        // 如果当前事件不在定时器中，则加入定时器
        if (!rev->timer_set) {
            cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
            ngx_add_timer(rev, cscf->client_header_timeout);
        }

        // 将当前事件加入事件驱动处理模块
        if (ngx_handle_read_event(rev, 0) != NGX_OK) {
            ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return NGX_ERROR;
        }

        return NGX_AGAIN;
    }

    if (n == 0) {
        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                      "client prematurely closed connection");
    }

    // 如果发生错误，就结束请求
    if (n == 0 || n == NGX_ERROR) {
        c->error = 1;
        c->log->action = "reading client request headers";

        ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
        return NGX_ERROR;
    }

    r->header_in->last += n;

    return n;
}
```

## 11.5 接收 HTTP 头部

HTTP request 的格式：

- Request line
- Request header
- 空行
- Request body

```http
GET /uri HTTP/1.1
cred: xxx
username: ttt
content-length: 4

test
```

这一步的目的是为了完整接收 HTTP headers，由 `ngx_http_process_request_headers()` 实现。在接收较大头部时，可能会被多次调用。因此，这个函数也是可重入的。

HTTP headers 也是可变长的字符串，需要用状态机来解析数据。当然，对于其总长度是有限制的。当最初分配的 `client_header_buffer_size` 内存大小不够用时，Nginx 会再次分配大小为 `large_client_header_buffers` 的缓冲区 (这两个值都在配置中指定)。也就是说，request line 与 request header 的长度总和不能超过 `large_client_header_buffers`，否则 Nginx 会报错。

```c
static void
ngx_http_process_request_headers(ngx_event_t *rev)
{
    u_char                     *p;
    size_t                      len;
    ssize_t                     n;
    ngx_int_t                   rc, rv;
    ngx_table_elt_t            *h;
    ngx_connection_t           *c;
    ngx_http_header_t          *hh;
    ngx_http_request_t         *r;
    ngx_http_core_srv_conf_t   *cscf;
    ngx_http_core_main_conf_t  *cmcf;

    c = rev->data;
    r = c->data;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, rev->log, 0,
                   "http process request header line");

    // 检查读事件是否超时，如果超时则关闭连接
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        c->timedout = 1;
        ngx_http_close_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }

    // 获取核心配置项
    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    rc = NGX_AGAIN;

    for ( ;; ) {

        if (rc == NGX_AGAIN) {

            // 检查 header_in 缓冲区是否耗尽
            if (r->header_in->pos == r->header_in->end) {

                // 重新分配更大的缓冲区
                rv = ngx_http_alloc_large_header_buffer(r, 0);

                if (rv == NGX_ERROR) {
                    ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                    break;
                }

                // 已达到缓冲区上限，无法分配更大的缓冲区
                if (rv == NGX_DECLINED) {
                    p = r->header_name_start;

                    r->lingering_close = 1;

                    // HTTP 494
                    if (p == NULL) {
                        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                                      "client sent too large request");
                        ngx_http_finalize_request(r,
                                            NGX_HTTP_REQUEST_HEADER_TOO_LARGE);
                        break;
                    }

                    len = r->header_in->end - p;

                    if (len > NGX_MAX_ERROR_STR - 300) {
                        len = NGX_MAX_ERROR_STR - 300;
                    }

                    ngx_log_error(NGX_LOG_INFO, c->log, 0,
                                "client sent too long header line: \"%*s...\"",
                                len, r->header_name_start);

                    ngx_http_finalize_request(r,
                                            NGX_HTTP_REQUEST_HEADER_TOO_LARGE);
                    break;
                }
            }

            // 从内核 TCP 读缓冲区中复制数据
            n = ngx_http_read_request_header(r);

            if (n == NGX_AGAIN || n == NGX_ERROR) {
                break;
            }
        }

        /* the host header could change the server configuration context */
        cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);

        // 解析 TCP 字符流
        rc = ngx_http_parse_header_line(r, r->header_in,
                                        cscf->underscores_in_headers);

        // 成功解析 HTTP headers
        if (rc == NGX_OK) {

            r->request_length += r->header_in->pos - r->header_name_start;

            if (r->invalid_header && cscf->ignore_invalid_headers) {

                /* there was error while a header line parsing */

                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                              "client sent invalid header line: \"%*s\"",
                              r->header_end - r->header_name_start,
                              r->header_name_start);
                continue;
            }

            /* a header line has been parsed successfully */

            // 成功解析了一个 HTTP header
            h = ngx_list_push(&r->headers_in.headers);
            if (h == NULL) {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                break;
            }

            h->hash = r->header_hash;

            h->key.len = r->header_name_end - r->header_name_start;
            h->key.data = r->header_name_start;
            h->key.data[h->key.len] = '\0';

            h->value.len = r->header_end - r->header_start;
            h->value.data = r->header_start;
            h->value.data[h->value.len] = '\0';

            h->lowcase_key = ngx_pnalloc(r->pool, h->key.len);
            if (h->lowcase_key == NULL) {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                break;
            }

            if (h->key.len == r->lowcase_index) {
                ngx_memcpy(h->lowcase_key, r->lowcase_header, h->key.len);

            } else {
                ngx_strlow(h->lowcase_key, h->key.data, h->key.len);
            }

            hh = ngx_hash_find(&cmcf->headers_in_hash, h->hash,
                               h->lowcase_key, h->key.len);

            if (hh && hh->handler(r, h, hh->offset) != NGX_OK) {
                break;
            }

            ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "http header: \"%V: %V\"",
                           &h->key, &h->value);

            continue;
        }

        // 完整的 header 解析完毕
        if (rc == NGX_HTTP_PARSE_HEADER_DONE) {

            /* a whole header has been parsed successfully */

            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "http header done");

            r->request_length += r->header_in->pos - r->header_name_start;

            r->http_state = NGX_HTTP_PROCESS_REQUEST_STATE;

            rc = ngx_http_process_request_header(r);

            if (rc != NGX_OK) {
                break;
            }

            // 开始处理 HTTP 请求
            ngx_http_process_request(r);

            break;
        }

        if (rc == NGX_AGAIN) {

            /* a header line parsing is still not complete */

            continue;
        }

        /* rc == NGX_HTTP_PARSE_INVALID_HEADER */

        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                      "client sent invalid header line");

        ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
        break;
    }

    // 最终执行 POST 请求
    ngx_http_run_posted_requests(c);
}
```

## 11.6 处理 HTTP 请求

接收到完整的 HTTP headers 之后，已经有足够多必要的信息开始在业务上处理 HTTP 请求了。从上面的代码可以看到，最终都是通过调用 `ngx_http_process_request()` 处理请求。

> 只有 HTTP 请求被第一次处理时，调用的是 `ngx_http_process_request()`。如果一次没有处理完毕，当 HTTP 框架再次处理同一个请求时，将会调用 `ngx_http_request_handler()`。

```c
void
ngx_http_process_request(ngx_http_request_t *r)
{
    ngx_connection_t  *c;

    c = r->connection;

#if (NGX_HTTP_SSL)

    if (r->http_connection->ssl) {
        long                      rc;
        X509                     *cert;
        const char               *s;
        ngx_http_ssl_srv_conf_t  *sscf;

        if (c->ssl == NULL) {
            ngx_log_error(NGX_LOG_INFO, c->log, 0,
                          "client sent plain HTTP request to HTTPS port");
            ngx_http_finalize_request(r, NGX_HTTP_TO_HTTPS);
            return;
        }

        sscf = ngx_http_get_module_srv_conf(r, ngx_http_ssl_module);

        if (sscf->verify) {
            rc = SSL_get_verify_result(c->ssl->connection);

            if (rc != X509_V_OK
                && (sscf->verify != 3 || !ngx_ssl_verify_error_optional(rc)))
            {
                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                              "client SSL certificate verify error: (%l:%s)",
                              rc, X509_verify_cert_error_string(rc));

                ngx_ssl_remove_cached_session(c->ssl->session_ctx,
                                       (SSL_get0_session(c->ssl->connection)));

                ngx_http_finalize_request(r, NGX_HTTPS_CERT_ERROR);
                return;
            }

            if (sscf->verify == 1) {
                cert = SSL_get_peer_certificate(c->ssl->connection);

                if (cert == NULL) {
                    ngx_log_error(NGX_LOG_INFO, c->log, 0,
                                  "client sent no required SSL certificate");

                    ngx_ssl_remove_cached_session(c->ssl->session_ctx,
                                       (SSL_get0_session(c->ssl->connection)));

                    ngx_http_finalize_request(r, NGX_HTTPS_NO_CERT);
                    return;
                }

                X509_free(cert);
            }

            if (ngx_ssl_ocsp_get_status(c, &s) != NGX_OK) {
                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                              "client SSL certificate verify error: %s", s);

                ngx_ssl_remove_cached_session(c->ssl->session_ctx,
                                       (SSL_get0_session(c->ssl->connection)));

                ngx_http_finalize_request(r, NGX_HTTPS_CERT_ERROR);
                return;
            }
        }
    }

#endif

    // 将当前连接的读事件从定时器中移除
    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }

#if (NGX_STAT_STUB)
    (void) ngx_atomic_fetch_add(ngx_stat_reading, -1);
    r->stat_reading = 0;
    (void) ngx_atomic_fetch_add(ngx_stat_writing, 1);
    r->stat_writing = 1;
#endif

    // 现在开始不再需要接收 HTTP request line 或 HTTP header
    // 因此重新设置读写事件的回调函数
    c->read->handler = ngx_http_request_handler;
    c->write->handler = ngx_http_request_handler;
    // 为请求的读事件设置一个不处理任何事的回调函数
    r->read_event_handler = ngx_http_block_reading;

    ngx_http_handler(r);
}
```

准备工作结束后，接下来开始处理 HTTP 请求：

```c
void
ngx_http_handler(ngx_http_request_t *r)
{
    ngx_http_core_main_conf_t  *cmcf;

    r->connection->log->action = NULL;

    // 判断请求是否要做内部跳转
    if (!r->internal) {
        switch (r->headers_in.connection_type) {
        case 0:
            r->keepalive = (r->http_version > NGX_HTTP_VERSION_10);
            break;

        case NGX_HTTP_CONNECTION_CLOSE:
            r->keepalive = 0;
            break;

        case NGX_HTTP_CONNECTION_KEEP_ALIVE:
            r->keepalive = 1;
            break;
        }

        r->lingering_close = (r->headers_in.content_length_n > 0
                              || r->headers_in.chunked);
        // 不做内部跳转，则从第一个回调函数开始
        r->phase_handler = 0;

    } else {
        // 否则，跳转到 NGX_HTTP_SERVER_REWRITE_PHASE 阶段处理
        cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
        r->phase_handler = cmcf->phase_engine.server_rewrite_index;
    }

    r->valid_location = 1;
#if (NGX_HTTP_GZIP)
    r->gzip_tested = 0;
    r->gzip_ok = 0;
    r->gzip_vary = 0;
#endif

    // 为请求的写事件设置回调，并直接开始执行
    r->write_event_handler = ngx_http_core_run_phases;
    ngx_http_core_run_phases(r);
}
```

在 `ngx_http_core_run_phases()` 中，开始调用各 HTTP 模块共同处理请求：

```c
void
ngx_http_core_run_phases(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_phase_handler_t   *ph;
    ngx_http_core_main_conf_t  *cmcf;

    // 全局核心配置项
    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    // 核心配置中的阶段处理引擎
    ph = cmcf->phase_engine.handlers;

    // 依次执行每一个阶段的 checker()
    // 在 checker() 中，各个 handler() 会被调用
    while (ph[r->phase_handler].checker) {

        rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

        if (rc == NGX_OK) {
            return;
        }
    }
}
```

其中，`phase_handler` 决定了当前执行到哪一个阶段。这个值可以被 HTTP 框架重置，以实现阶段之间的跳转。很可能一次无法处理完所有的 HTTP 请求，那么控制权会交还为 HTTP 框架。再下一次触发时，将会由 `ngx_http_request_handler()` 来处理 (注意这里换了回调函数)。这个函数已经在之前在 `ngx_http_process_request()` 中 被设置为 TCP 连接读写事件的回调函数：

```c
static void
ngx_http_request_handler(ngx_event_t *ev)
{
    ngx_connection_t    *c;
    ngx_http_request_t  *r;

    c = ev->data;
    r = c->data;

    ngx_http_set_log_request(c->log, r);

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http run request: \"%V?%V\"", &r->uri, &r->args);

    if (c->close) {
        r->main->count++;
        ngx_http_terminate_request(r, 0);
        ngx_http_run_posted_requests(c);
        return;
    }

    if (ev->delayed && ev->timedout) {
        ev->delayed = 0;
        ev->timedout = 0;
    }

    // HTTP 请求可写，则调用写事件回调 ngx_http_core_run_phases()
    // 继续按阶段调用各 HTTP 模块的处理函数
    if (ev->write) {
        r->write_event_handler(r);

    } else {
        // 读事件回调不做任何事
        r->read_event_handler(r);
    }

    // 执行 POST 请求
    ngx_http_run_posted_requests(c);
}
```

综上，HTTP 请求被处理的方式是共同的：

1. 先按阶段调用各个 HTTP 模块处理请求
2. 再处理 POST 请求

而按阶段处理请求的具体方式是依次调用每一个阶段的 `checker()`。`checker()` 函数的主要任务是执行该阶段每个 HTTP 模块实现的回调函数，并根据返回值决定：

1. 当前阶段是否结束 (之后的回调还需要被执行吗)
2. 下次要执行的回调是哪一个 (下一个，还是下阶段第一个)
3. 立刻执行下一个回调函数，还是先将控制权交还给事件驱动模块

对于 11 个阶段，实现了不同的 `checker()` 函数。这些函数的具体代码已经在上一节中分析过。
