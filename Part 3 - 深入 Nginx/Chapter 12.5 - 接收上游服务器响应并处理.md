# Chapter 12.5 - 接收上游服务器响应并处理

Created by : Mr Dk.

2020 / 08 / 01 21:21

Nanjing, Jiangsu, China

---

## 12.5 接收上游服务器的响应头部

### 12.5.1 应用层协议的两段划分方式

Nginx 设置了 `ngx_http_upstream_process_header()` 函数处理上游服务器的响应。Upstream 机制支持上游服务器使用任何基于 TCP 的应用层协议 - TCP 其实就是 **有顺序的数据流**。实际上，上游服务器的响应可大可小，如果在内存中保存完整的响应，可能会引发内存不够的问题；如果在磁盘文件中接收响应，又会带来大量 I/O。

为了解决这个问题，应用层协议通常会将请求划分为 header 和 body。Header 中抽象了不同协议报文之间的共同部分，不同协议数据包的 header 都具有相同的格式。服务器必须解析 header，而 body 则不做格式上的要求，也不一定解析。Header 的长度要么是固定的，要么在一个指定的范围内 - 所以在处理 header 时，开辟的内存只需要容纳 header 的长度即可。

Header 和 body 存储什么样的信息取决于应用层协议，upstream 机制并不关心。Upstream 机制已经抽象出了 `process_header()` 函数，由具体的 HTTP 模块实现 header 解析逻辑。如果 HTTP 模块的目的是反向代理，那么可以将解析出的 header 适配到响应给下游客户端的 header 中。

而 body 的内容较为简单，HTTP 模块一般都不解析 body。Upstream 机制抽象了三种形式的 body 处理方式：

1. 不转发响应
2. 下游网速优先的响应 (使用固定的缓冲区存放上游响应，因为上游响应会很快被下游取走)
3. 上游网速优先的响应 (使用较大内存或磁盘文件来缓存上游响应，因为可能会有堆积)

### 12.5.3 接收响应头部的流程

`process_header` 是 `ngx_http_upstream_t` 结构体中定义的函数指针。Upstream 机制不涉及应用层协议，由使用 upstream 机制的模块负责解析应用层协议。HTTP 模块提供的回调函数是 `ngx_http_upstream_process_header()`。这个函数也被设计为 `ngx_http_upstream_t` 结构体中的读事件回调函数，会被反复调用

```c
static void
ngx_http_upstream_process_header(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
    ssize_t            n;
    ngx_int_t          rc;
    ngx_connection_t  *c;

    // 获取 Nginx 与上游服务器的连接
    c = u->peer.connection;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http upstream process header");

    c->log->action = "reading response header from upstream";

    // 检测读事件是否超时
    if (c->read->timedout) {
        ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_TIMEOUT);
        return;
    }

    // 还没有请求发送到上游服务器，就收到了响应，不对劲
    if (!u->request_sent && ngx_http_upstream_test_connect(c) != NGX_OK) {
        ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_ERROR);
        return;
    }

    // 接收响应头的缓冲区还未分配内存
    if (u->buffer.start == NULL) {
        // 分配缓冲区内存
        u->buffer.start = ngx_palloc(r->pool, u->conf->buffer_size);
        if (u->buffer.start == NULL) {
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        u->buffer.pos = u->buffer.start;
        u->buffer.last = u->buffer.start;
        u->buffer.end = u->buffer.start + u->conf->buffer_size;
        u->buffer.temporary = 1;

        u->buffer.tag = u->output.tag;

        if (ngx_list_init(&u->headers_in.headers, r->pool, 8,
                          sizeof(ngx_table_elt_t))
            != NGX_OK)
        {
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        if (ngx_list_init(&u->headers_in.trailers, r->pool, 2,
                          sizeof(ngx_table_elt_t))
            != NGX_OK)
        {
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

#if (NGX_HTTP_CACHE)

        if (r->cache) {
            u->buffer.pos += r->cache->header_start;
            u->buffer.last = u->buffer.pos;
        }
#endif
    }

    for ( ;; ) {

        // 调用 recv() 读取上游响应
        n = c->recv(c, u->buffer.last, u->buffer.end - u->buffer.last);

        // 接下来还需要继续接收响应
        if (n == NGX_AGAIN) {
#if 0
            // 将读事件添加到定时器中
            ngx_add_timer(rev, u->read_timeout);
#endif
            // 将读事件添加到事件驱动模块中
            if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
                ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }

            // 结束函数
            return;
        }

        // 上游服务器关闭连接
        if (n == 0) {
            ngx_log_error(NGX_LOG_ERR, c->log, 0,
                          "upstream prematurely closed connection");
        }
        // 发生错误
        if (n == NGX_ERROR || n == 0) {
            ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_ERROR);
            return;
        }

        // 统计接收到的响应长度
        u->state->bytes_received += n;
        // 缓冲区地址后移
        u->buffer.last += n;

#if 0
        u->valid_header_in = 0;

        u->peer.cached = 0;
#endif
        // 解析 header
        rc = u->process_header(r);

        // Header 还没有接收完整
        if (rc == NGX_AGAIN) {
            // 检查缓冲区是否耗尽
            if (u->buffer.last == u->buffer.end) {
                ngx_log_error(NGX_LOG_ERR, c->log, 0,
                              "upstream sent too big header");

                ngx_http_upstream_next(r, u,
                                       NGX_HTTP_UPSTREAM_FT_INVALID_HEADER);
                return;
            }

            // 等待下一次触发
            continue;
        }

        break;
    }

    // Header 不合法
    if (rc == NGX_HTTP_UPSTREAM_INVALID_HEADER) {
        ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_INVALID_HEADER);
        return;
    }

    // Header 错误
    if (rc == NGX_ERROR) {
        ngx_http_upstream_finalize_request(r, u,
                                           NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    /* rc == NGX_OK */

    u->state->header_time = ngx_current_msec - u->start_time;

    if (u->headers_in.status_n >= NGX_HTTP_SPECIAL_RESPONSE) {

        if (ngx_http_upstream_test_next(r, u) == NGX_OK) {
            return;
        }

        if (ngx_http_upstream_intercept_errors(r, u) == NGX_OK) {
            return;
        }
    }

    // 处理已经解析出的 headers，会将已经解析出的 headers 设置到 request 结构体的 headers_out 中
    if (ngx_http_upstream_process_headers(r, u) != NGX_OK) {
        return;
    }

    // 开始向客户端转发响应
    ngx_http_upstream_send_response(r, u);
}
```

### 12.5.4 响应体的处理

转发响应由 `ngx_http_upstream_send_response()` 进行。首先会把 headers 发送出去。之后在函数中，会根据配置项中的 `buffering` 标志来决定是否打开缓存处理 body - 打开缓存意味着上游网速更快 (比如 Nginx 与上游服务器在同一个内网中)，从而进行是否需要转发响应的判断。

如果不打开缓存：

```c
if (!u->buffering) {

#if (NGX_HTTP_CACHE)

    if (r->cache) {
        ngx_http_file_cache_free(r->cache, u->pipe->temp_file);
    }

#endif

    // 判断 HTTP 模块是否实现了用于处理 body 的 input_filter 函数
    // 如果没有实现，则给出一个默认的实现函数
    // 如果用户想自己处理 body，会实现这个函数
    if (u->input_filter == NULL) {
        // 处理 body 前的初始化工作
        u->input_filter_init = ngx_http_upstream_non_buffered_filter_init;
        // 处理 body 的回调函数
        u->input_filter = ngx_http_upstream_non_buffered_filter;
        // 回调函数的参数，用于被多次调用时保存上下文
        u->input_filter_ctx = r;
    }

    // 设置读事件回调函数 (接收到上游服务器的响应时回调)
    u->read_event_handler = ngx_http_upstream_process_non_buffered_upstream;
    // 设置写事件回调函数 (向下游客户端转发响应时回调)
    r->write_event_handler =
        ngx_http_upstream_process_non_buffered_downstream;

    r->limit_rate = 0;
    r->limit_rate_set = 1;

    // 调用 input_filter 的初始化函数，做好处理 body 前的准备工作
    if (u->input_filter_init(u->input_filter_ctx) == NGX_ERROR) {
        ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
        return;
    }

    if (clcf->tcp_nodelay && ngx_tcp_nodelay(c) != NGX_OK) {
        ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
        return;
    }

    // 解析完 header 后，还剩下的字符
    n = u->buffer.last - u->buffer.pos;

    // 如果还有剩余字符，说明已经收到了部分 body
    if (n) {
        u->buffer.last = u->buffer.pos;

        u->state->response_length += n;

        // 调用 input_filter 处理 body
        if (u->input_filter(u->input_filter_ctx, n) == NGX_ERROR) {
            ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
            return;
        }

        // Body 处理结束，向下游转发 body
        ngx_http_upstream_process_non_buffered_downstream(r);

    } else {
        u->buffer.pos = u->buffer.start;
        u->buffer.last = u->buffer.start;

        if (ngx_http_send_special(r, NGX_HTTP_FLUSH) == NGX_ERROR) {
            ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
            return;
        }

        // 如果与上游的连接就绪，那么调用读回调函数
        if (u->peer.connection->read->ready || u->length == 0) {
            ngx_http_upstream_process_non_buffered_upstream(r, u);
        }
    }

    return;
}
```

如果打开缓存：

```c
// 准备并处理 body
if (u->input_filter_init
    && u->input_filter_init(p->input_ctx) != NGX_OK)
{
    ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
    return;
}

// 设置读事件回调函数
u->read_event_handler = ngx_http_upstream_process_upstream;
// 设置写事件回调函数
r->write_event_handler = ngx_http_upstream_process_downstream;

// 调用读事件回调函数，向上游服务器读取响应
ngx_http_upstream_process_upstream(r, u);
```

可以看到，对响应中 body 的处理，由 `input_filter()` 完成。如果没有实现，那么 Nginx 提供了一个默认实现 (以及一个相对应的初始化函数)。默认的初始化函数为 `ngx_http_upstream_non_buffered_filter_init()`：

```c
ngx_int_t
ngx_http_upstream_non_buffered_filter_init(void *data)
{
    return NGX_OK;
}
```

默认的 body 处理函数为 `ngx_http_upstream_non_buffered_filter()`。从函数名就可以看出来，这个函数不使用 (文件) 缓存。也就是说，将会试图在内存缓冲区中存放完整的 body - 一旦上游服务器发来的响应超过了内存缓冲区的大小，请求将会出错：

```c
ngx_int_t
ngx_http_upstream_non_buffered_filter(void *data, ssize_t bytes)
{
    ngx_http_request_t  *r = data;

    ngx_buf_t            *b;
    ngx_chain_t          *cl, **ll;
    ngx_http_upstream_t  *u;

    u = r->upstream;

    // 找到 out_bufs 的末尾以便添加新的缓冲区
    for (cl = u->out_bufs, ll = &u->out_bufs; cl; cl = cl->next) {
        ll = &cl->next;
    }

    // 获得一个空闲的 ngx_buf_t
    cl = ngx_chain_get_free_buf(r->pool, &u->free_bufs);
    if (cl == NULL) {
        return NGX_ERROR;
    }

    // 新的缓冲块加入链表尾
    *ll = cl;

    cl->buf->flush = 1;
    cl->buf->memory = 1;

    b = &u->buffer;

    // Buffer 指针向后移动，保存 body
    cl->buf->pos = b->last;
    b->last += bytes;
    cl->buf->last = b->last;
    cl->buf->tag = u->output.tag;

    if (u->length == -1) {
        return NGX_OK;
    }

    if (bytes > u->length) {

        ngx_log_error(NGX_LOG_WARN, r->connection->log, 0,
                      "upstream sent more data than specified in "
                      "\"Content-Length\" header");

        cl->buf->last = cl->buf->pos + u->length;
        u->length = 0;

        return NGX_OK;
    }

    // 需要接收的 body 长度减少
    u->length -= bytes;

    return NGX_OK;
}
```

这是处理响应 body 的回调函数。从 Nginx 的视角来看，它的读事件应该是从上游服务器获取响应，写事件应该是向下游客户端转发响应。因此上面函数中设置的读写回调函数才是 `input_filter()` 函数的调用者。根据 Nginx 根据网络环境处理响应的策略，读写回调函数也被分为了两组：

- 不打开缓存 - `ngx_http_upstream_process_non_buffered_upstream()` / `ngx_http_upstream_process_non_buffered_downstream()`
- 打开缓存 - `ngx_http_upstream_process_upstream()` / `ngx_http_upstream_process_downstream()`
