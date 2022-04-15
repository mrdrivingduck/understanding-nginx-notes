# Chapter 12.1-12.4 - Upstream 与上游服务器通信

Created by : Mr Dk.

2020 / 07 / 31 23:45

Nanjing, Jiangsu, China

---

## 12.1 Upstream 机制概述

在 Nginx 中，Nginx 与客户端之间被称为 _下游_，Nginx 与其它服务器之间被称为 _上游_。Nginx 要为下游提供功能，很多时候需要从上游服务器获取素材。所谓的 upstream 机制就是使 HTTP 模块在处理客户端请求时可以访问上游的其它服务器。

Upstream 机制由 `ngx_http_upstream_module` 模块实现，它是一个 HTTP 模块。也就是说，如果使用 upstream 机制，客户端的请求必须基于 HTTP。由不同的模块支持 Nginx 访问不同的上游服务器。

对于每个 `ngx_http_request_t` 请求来说，只能访问一个上有服务器。但是一个客户端请求可以派生出很多子请求，每个子请求都能单独访问一个上游服务器。Upstream 机制不只提供了直接处理上游服务器响应的功能，还具有将上游服务器的响应 **即时转发** 给下游客户端的功能 - 因此每个 `ngx_http_request_t` 只能用来访问一个上游服务器。

在转发响应时，要解决两个问题：

1. 下游协议是 HTTP，上游可以是基于 TCP 的任何协议，需要做协议适配
2. 上下游的网速差别较大时，针对不同场景的具体实现不同

### 12.1.2 `ngx_http_upstream_t` 数据结构的意义

使用 upstream 机制，那么一个请求就必须维护一个从 Nginx 到上游服务器的连接。该连接由以下结构体维护。所以要为一个请求构造这个结构体。一个请求默认是不使用 upstream 机制的，那么这个结构体就不会被构造。

```c
typedef struct ngx_http_upstream_s    ngx_http_upstream_t;

struct ngx_http_upstream_s {
    // 读写事件的回调函数，每个阶段的读写回调都不同
    ngx_http_upstream_handler_pt     read_event_handler;
    ngx_http_upstream_handler_pt     write_event_handler;

    // 向上游服务器发起的连接
    ngx_peer_connection_t            peer;

    // 上游网速快时，使用 pipe 转发响应
    ngx_event_pipe_t                *pipe;

    ngx_chain_t                     *request_bufs;

    ngx_output_chain_ctx_t           output;
    ngx_chain_writer_ctx_t           writer;

    // 使用 upstream 时的各种配置
    ngx_http_upstream_conf_t        *conf;
    ngx_http_upstream_srv_conf_t    *upstream;
#if (NGX_HTTP_CACHE)
    ngx_array_t                     *caches;
#endif

    // Upstream 直接转发响应时，保存解析好的响应头部
    ngx_http_upstream_headers_in_t   headers_in;

    // 解析主机域名
    ngx_http_upstream_resolved_t    *resolved;

    ngx_buf_t                        from_client;

    // 接收上游服务器响应头的缓冲区，以及长度
    ngx_buf_t                        buffer;
    off_t                            length;

    ngx_chain_t                     *out_bufs;
    ngx_chain_t                     *busy_bufs;
    ngx_chain_t                     *free_bufs;

    // 处理 body 前的初始化函数
    ngx_int_t                      (*input_filter_init)(void *data);
    // 处理 body 的函数
    ngx_int_t                      (*input_filter)(void *data, ssize_t bytes);
    // 前两个函数的参数
    void                            *input_filter_ctx;

#if (NGX_HTTP_CACHE)
    ngx_int_t                      (*create_key)(ngx_http_request_t *r);
#endif
    // 构造 upstream 请求
    ngx_int_t                      (*create_request)(ngx_http_request_t *r);
    // 如果通信失败，再次建立连接
    ngx_int_t                      (*reinit_request)(ngx_http_request_t *r);
    // 解析上游服务器的响应 header
    ngx_int_t                      (*process_header)(ngx_http_request_t *r);
    void                           (*abort_request)(ngx_http_request_t *r);
    // 结束请求
    void                           (*finalize_request)(ngx_http_request_t *r,
                                         ngx_int_t rc);
    // 上游响应出现 location/refresh 头部表示重定向时被调用
    ngx_int_t                      (*rewrite_redirect)(ngx_http_request_t *r,
                                         ngx_table_elt_t *h, size_t prefix);
    ngx_int_t                      (*rewrite_cookie)(ngx_http_request_t *r,
                                         ngx_table_elt_t *h);

    ngx_msec_t                       start_time;

    // 上游响应的信息
    ngx_http_upstream_state_t       *state;

    // 记录日志时会用到
    ngx_str_t                        method;
    ngx_str_t                        schema;
    ngx_str_t                        uri;

#if (NGX_HTTP_SSL || NGX_COMPAT)
    ngx_str_t                        ssl_name;
#endif

    ngx_http_cleanup_pt             *cleanup;

    unsigned                         store:1;
    unsigned                         cacheable:1;
    unsigned                         accel:1;
    unsigned                         ssl:1;
#if (NGX_HTTP_CACHE)
    unsigned                         cache_status:3;
#endif

    unsigned                         buffering:1;
    unsigned                         keepalive:1;
    unsigned                         upgrade:1;
    unsigned                         error:1;

    unsigned                         request_sent:1;
    unsigned                         request_body_sent:1;
    unsigned                         request_body_blocked:1;
    unsigned                         header_sent:1;
};
```

### 12.1.3 `ngx_http_upstream_conf_t` 配置结构体

这个结构体制定了 upstream 的运行方式，必须在启动 upstream 机制前设置。

```c
typedef struct {
    // 定义上游服务器的配置，在 ngx_http_upstream_t 结构体没有实现 resolved 时生效
    ngx_http_upstream_srv_conf_t    *upstream;

    // 各种超时时间
    ngx_msec_t                       connect_timeout;
    ngx_msec_t                       send_timeout;
    ngx_msec_t                       read_timeout;
    ngx_msec_t                       next_upstream_timeout;

    size_t                           send_lowat; // 发送缓冲区下限
    size_t                           buffer_size; // 接收响应头的缓冲区大小
    size_t                           limit_rate;

    size_t                           busy_buffers_size;
    size_t                           max_temp_file_size;
    size_t                           temp_file_write_size;

    size_t                           busy_buffers_size_conf;
    size_t                           max_temp_file_size_conf;
    size_t                           temp_file_write_size_conf;

    // 以缓存响应的方式转发上游服务器 body 时所使用的内存大小
    ngx_bufs_t                       bufs;

    ngx_uint_t                       ignore_headers; // Upstream 在转发时跳过对某些 header 的处理
    ngx_uint_t                       next_upstream; // 错误码？
    ngx_uint_t                       store_access; // 保存响应的临时文件的文件目录与权限
    ngx_uint_t                       next_upstream_tries;
    ngx_flag_t                       buffering; // 决定转发响应方式的标志位
    ngx_flag_t                       request_buffering;
    ngx_flag_t                       pass_request_headers;
    ngx_flag_t                       pass_request_body;

    ngx_flag_t                       ignore_client_abort; // 不检查 Nginx 与下游的连接是否断开
    ngx_flag_t                       intercept_errors;
    ngx_flag_t                       cyclic_temp_file;
    ngx_flag_t                       force_ranges;

    ngx_path_t                      *temp_path; // 转发响应时存放临时文件的路径

    ngx_hash_t                       hide_headers_hash;
    ngx_array_t                     *hide_headers; // 不希望转发给下游的 header
    ngx_array_t                     *pass_headers; // 希望转发给下游的 header (针对默认不会被转发给下游的 header)

    ngx_http_upstream_local_t       *local; // 连接上游服务器时的本机地址
    ngx_flag_t                       socket_keepalive;

#if (NGX_HTTP_CACHE)
    ngx_shm_zone_t                  *cache_zone;
    ngx_http_complex_value_t        *cache_value;

    ngx_uint_t                       cache_min_uses;
    ngx_uint_t                       cache_use_stale;
    ngx_uint_t                       cache_methods;

    off_t                            cache_max_range_offset;

    ngx_flag_t                       cache_lock;
    ngx_msec_t                       cache_lock_timeout;
    ngx_msec_t                       cache_lock_age;

    ngx_flag_t                       cache_revalidate;
    ngx_flag_t                       cache_convert_head;
    ngx_flag_t                       cache_background_update;

    ngx_array_t                     *cache_valid;
    ngx_array_t                     *cache_bypass;
    ngx_array_t                     *cache_purge;
    ngx_array_t                     *no_cache;
#endif

    ngx_array_t                     *store_lengths;
    ngx_array_t                     *store_values;

#if (NGX_HTTP_CACHE)
    signed                           cache:2;
#endif
    signed                           store:2;
    unsigned                         intercept_404:1;
    unsigned                         change_buffering:1;
    unsigned                         pass_trailers:1;
    unsigned                         preserve_output:1;

#if (NGX_HTTP_SSL || NGX_COMPAT)
    ngx_ssl_t                       *ssl;
    ngx_flag_t                       ssl_session_reuse;

    ngx_http_complex_value_t        *ssl_name;
    ngx_flag_t                       ssl_server_name;
    ngx_flag_t                       ssl_verify;
#endif

    ngx_str_t                        module;

    NGX_COMPAT_BEGIN(2)
    NGX_COMPAT_END
} ngx_http_upstream_conf_t;
```

## 12.2 启动 upstream

创建 `ngx_http_request_t` 结构体中的 `upstream` 结构体，并正确设置 `upstream->conf` 配置结构体。最后调用 `ngx_http_upstream_init()` 就可以启动 upstream 机制。

```c
void
ngx_http_upstream_init(ngx_http_request_t *r)
{
    ngx_connection_t     *c;

    // Nginx 与客户端的连接 (下游连接)
    c = r->connection;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http init upstream, client timer: %d", c->read->timer_set);

#if (NGX_HTTP_V2)
    if (r->stream) {
        ngx_http_upstream_init_request(r);
        return;
    }
#endif

    // 将下游连接从定时器中移除
    // 一旦启动 upstream，就不应该对客户端读操作带有超时处理
    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }

    if (ngx_event_flags & NGX_USE_CLEAR_EVENT) {

        // 将连接的可写事件加入事件驱动模块
        if (!c->write->active) {
            if (ngx_add_event(c->write, NGX_WRITE_EVENT, NGX_CLEAR_EVENT)
                == NGX_ERROR)
            {
                ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }
        }
    }

    ngx_http_upstream_init_request(r); // 初始化 upstream 请求
}
```

接下来初始化 upstream 请求。主要是填充 `ngx_http_upstream_t` 结构体：

```c
static void
ngx_http_upstream_init_request(ngx_http_request_t *r)
{
    ngx_str_t                      *host;
    ngx_uint_t                      i;
    ngx_resolver_ctx_t             *ctx, temp;
    ngx_http_cleanup_t             *cln;
    ngx_http_upstream_t            *u;
    ngx_http_core_loc_conf_t       *clcf;
    ngx_http_upstream_srv_conf_t   *uscf, **uscfp;
    ngx_http_upstream_main_conf_t  *umcf;

    if (r->aio) {
        return;
    }

    u = r->upstream;

#if (NGX_HTTP_CACHE)

    if (u->conf->cache) {
        ngx_int_t  rc;

        rc = ngx_http_upstream_cache(r, u);

        if (rc == NGX_BUSY) {
            r->write_event_handler = ngx_http_upstream_init_request;
            return;
        }

        r->write_event_handler = ngx_http_request_empty_handler;

        if (rc == NGX_ERROR) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        if (rc == NGX_OK) {
            rc = ngx_http_upstream_cache_send(r, u);

            if (rc == NGX_DONE) {
                return;
            }

            if (rc == NGX_HTTP_UPSTREAM_INVALID_HEADER) {
                rc = NGX_DECLINED;
                r->cached = 0;
                u->buffer.start = NULL;
                u->cache_status = NGX_HTTP_CACHE_MISS;
                u->request_sent = 1;
            }
        }

        if (rc != NGX_DECLINED) {
            ngx_http_finalize_request(r, rc);
            return;
        }
    }

#endif

    u->store = u->conf->store;

    if (!u->store && !r->post_action && !u->conf->ignore_client_abort) {
        r->read_event_handler = ngx_http_upstream_rd_check_broken_connection;
        r->write_event_handler = ngx_http_upstream_wr_check_broken_connection;
    }

    if (r->request_body) {
        u->request_bufs = r->request_body->bufs;
    }

    // 初始化向上游服务器的请求
    if (u->create_request(r) != NGX_OK) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    if (ngx_http_upstream_set_local(r, u, u->conf->local) != NGX_OK) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    if (u->conf->socket_keepalive) {
        u->peer.so_keepalive = 1;
    }

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    u->output.alignment = clcf->directio_alignment;
    u->output.pool = r->pool;
    u->output.bufs.num = 1;
    u->output.bufs.size = clcf->client_body_buffer_size;

    if (u->output.output_filter == NULL) {
        u->output.output_filter = ngx_chain_writer;
        u->output.filter_ctx = &u->writer;
    }

    u->writer.pool = r->pool;

    if (r->upstream_states == NULL) {

        r->upstream_states = ngx_array_create(r->pool, 1,
                                            sizeof(ngx_http_upstream_state_t));
        if (r->upstream_states == NULL) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

    } else {

        u->state = ngx_array_push(r->upstream_states);
        if (u->state == NULL) {
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        ngx_memzero(u->state, sizeof(ngx_http_upstream_state_t));
    }

    // 向 request 指向的 cleanup 链表中添加成员
    cln = ngx_http_cleanup_add(r, 0);
    if (cln == NULL) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    // 设置清理回调函数
    cln->handler = ngx_http_upstream_cleanup;
    cln->data = r;
    u->cleanup = &cln->handler;

    if (u->resolved == NULL) {

        uscf = u->conf->upstream;

    } else {

#if (NGX_HTTP_SSL)
        u->ssl_name = u->resolved->host;
#endif

        host = &u->resolved->host;

        umcf = ngx_http_get_module_main_conf(r, ngx_http_upstream_module);

        uscfp = umcf->upstreams.elts;

        for (i = 0; i < umcf->upstreams.nelts; i++) {

            uscf = uscfp[i];

            if (uscf->host.len == host->len
                && ((uscf->port == 0 && u->resolved->no_port)
                     || uscf->port == u->resolved->port)
                && ngx_strncasecmp(uscf->host.data, host->data, host->len) == 0)
            {
                goto found;
            }
        }

        if (u->resolved->sockaddr) {

            if (u->resolved->port == 0
                && u->resolved->sockaddr->sa_family != AF_UNIX)
            {
                ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                              "no port in upstream \"%V\"", host);
                ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }

            if (ngx_http_upstream_create_round_robin_peer(r, u->resolved)
                != NGX_OK)
            {
                ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }

            ngx_http_upstream_connect(r, u);

            return;
        }

        if (u->resolved->port == 0) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "no port in upstream \"%V\"", host);
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        temp.name = *host;

        ctx = ngx_resolve_start(clcf->resolver, &temp);
        if (ctx == NULL) {
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        if (ctx == NGX_NO_RESOLVER) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "no resolver defined to resolve %V", host);

            ngx_http_upstream_finalize_request(r, u, NGX_HTTP_BAD_GATEWAY);
            return;
        }

        ctx->name = *host;
        ctx->handler = ngx_http_upstream_resolve_handler;
        ctx->data = r;
        ctx->timeout = clcf->resolver_timeout;

        u->resolved->ctx = ctx;

        if (ngx_resolve_name(ctx) != NGX_OK) {
            u->resolved->ctx = NULL;
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        return;
    }

found:

    if (uscf == NULL) {
        ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0,
                      "no upstream configuration");
        ngx_http_upstream_finalize_request(r, u,
                                           NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    u->upstream = uscf;

#if (NGX_HTTP_SSL)
    u->ssl_name = uscf->host;
#endif

    if (uscf->peer.init(r, uscf) != NGX_OK) {
        ngx_http_upstream_finalize_request(r, u,
                                           NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    u->peer.start_time = ngx_current_msec;

    if (u->conf->next_upstream_tries
        && u->peer.tries > u->conf->next_upstream_tries)
    {
        u->peer.tries = u->conf->next_upstream_tries;
    }

    // 向上游服务器发起连接
    ngx_http_upstream_connect(r, u);
}
```

## 12.3 与上游服务器建立连接

建立 TCP 需要三次握手，而三次握手的时间是不可控的。为保证 TCP 连接不阻塞进程，Nginx 使用非阻塞 socket 连接上游服务器。因此，在建立连接发起后，如果没有立刻返回成功，需要在事件驱动模块中监控这个 socket，直到出现 **可写** 事件时，才说明连接已经建立成功。所以需要设置一个写事件回调函数。

```c
static void
ngx_http_upstream_connect(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
    ngx_int_t          rc;
    ngx_connection_t  *c;

    r->connection->log->action = "connecting to upstream";

    if (u->state && u->state->response_time == (ngx_msec_t) -1) {
        u->state->response_time = ngx_current_msec - u->start_time;
    }

    u->state = ngx_array_push(r->upstream_states);
    if (u->state == NULL) {
        ngx_http_upstream_finalize_request(r, u,
                                           NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    ngx_memzero(u->state, sizeof(ngx_http_upstream_state_t));

    u->start_time = ngx_current_msec;

    u->state->response_time = (ngx_msec_t) -1;
    u->state->connect_time = (ngx_msec_t) -1;
    u->state->header_time = (ngx_msec_t) -1;

    // 发起 TCP 连接
    rc = ngx_event_connect_peer(&u->peer);

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http upstream connect: %i", rc);

    if (rc == NGX_ERROR) {
        ngx_http_upstream_finalize_request(r, u,
                                           NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    u->state->peer = u->peer.name;

    // 连接暂未建立成功
    if (rc == NGX_BUSY) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "no live upstreams");
        ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_NOLIVE);
        return;
    }

    if (rc == NGX_DECLINED) {
        ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_ERROR);
        return;
    }

    /* rc == NGX_OK || rc == NGX_AGAIN || rc == NGX_DONE */

    c = u->peer.connection;

    c->requests++;

    c->data = r;

    // 设置连接的读写回调函数
    c->write->handler = ngx_http_upstream_handler;
    c->read->handler = ngx_http_upstream_handler;

    // 向上游服务器发送请求的回调函数
    u->write_event_handler = ngx_http_upstream_send_request_handler;
    // 接收上游服务器响应的回调函数
    u->read_event_handler = ngx_http_upstream_process_header;

    c->sendfile &= r->connection->sendfile;
    u->output.sendfile = c->sendfile;

    if (r->connection->tcp_nopush == NGX_TCP_NOPUSH_DISABLED) {
        c->tcp_nopush = NGX_TCP_NOPUSH_DISABLED;
    }

    if (c->pool == NULL) {

        /* we need separate pool here to be able to cache SSL connections */

        c->pool = ngx_create_pool(128, r->connection->log);
        if (c->pool == NULL) {
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }
    }

    c->log = r->connection->log;
    c->pool->log = c->log;
    c->read->log = c->log;
    c->write->log = c->log;

    /* init or reinit the ngx_output_chain() and ngx_chain_writer() contexts */

    u->writer.out = NULL;
    u->writer.last = &u->writer.out;
    u->writer.connection = c;
    u->writer.limit = 0;

    if (u->request_sent) {
        if (ngx_http_upstream_reinit(r, u) != NGX_OK) {
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }
    }

    if (r->request_body
        && r->request_body->buf
        && r->request_body->temp_file
        && r == r->main)
    {
        /*
         * the r->request_body->buf can be reused for one request only,
         * the subrequests should allocate their own temporary bufs
         */

        u->output.free = ngx_alloc_chain_link(r->pool);
        if (u->output.free == NULL) {
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        u->output.free->buf = r->request_body->buf;
        u->output.free->next = NULL;
        u->output.allocated = 1;

        r->request_body->buf->pos = r->request_body->buf->start;
        r->request_body->buf->last = r->request_body->buf->start;
        r->request_body->buf->tag = u->output.tag;
    }

    u->request_sent = 0;
    u->request_body_sent = 0;
    u->request_body_blocked = 0;

    // 如果连接还未建立，那么把连接的写事件添加到定时器中
    if (rc == NGX_AGAIN) {
        ngx_add_timer(c->write, u->conf->connect_timeout);
        return;
    }

#if (NGX_HTTP_SSL)

    if (u->ssl && c->ssl == NULL) {
        ngx_http_upstream_ssl_init_connection(r, u, c);
        return;
    }

#endif

    // 连接建立成功，向上游服务器发送请求
    ngx_http_upstream_send_request(r, u, 1);
}
```

其中，`ngx_connection_t` 的读写回调函数 `ngx_http_upstream_handler()` 主要工作是调用 `ngx_http_upstream_t` 中的读写回调函数。与 HTTP 框架中的读写回调函数类似，优先调用写回调函数，再调用读回调函数：

```c
static void
ngx_http_upstream_handler(ngx_event_t *ev)
{
    ngx_connection_t     *c;
    ngx_http_request_t   *r;
    ngx_http_upstream_t  *u;

    c = ev->data;
    r = c->data;

    u = r->upstream;
    c = r->connection;

    ngx_http_set_log_request(c->log, r);

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http upstream request: \"%V?%V\"", &r->uri, &r->args);

    if (ev->delayed && ev->timedout) {
        ev->delayed = 0;
        ev->timedout = 0;
    }

    // Nginx 与上游服务器之间的 TCP 连接可写事件被触发，调用写回调函数
    if (ev->write) {
        u->write_event_handler(r, u);

    } else {
        // 可读事件被触发，调用回调函数
        u->read_event_handler(r, u);
    }

    // 处理 POST 请求
    ngx_http_run_posted_requests(c);
}
```

## 12.4 发送请求到上游服务器

`ngx_http_upstream_t` 的读写回调是怎么实现的呢？根据上面的代码，写事件回调是 `ngx_http_upstream_send_request_handler()`，负责向上游服务器发送请求。由于请求大小未知，这个回调函数可能会被事件驱动模块多次触发多次调用。

```c
static void
ngx_http_upstream_send_request_handler(ngx_http_request_t *r,
    ngx_http_upstream_t *u)
{
    ngx_connection_t  *c;

    // 获取 Nginx 与上游服务器的连接结构体
    c = u->peer.connection;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http upstream send request handler");

    // 连接超时，由错误处理策略决定如何处理
    if (c->write->timedout) {
        ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_TIMEOUT);
        return;
    }

#if (NGX_HTTP_SSL)

    if (u->ssl && c->ssl == NULL) {
        ngx_http_upstream_ssl_init_connection(r, u, c);
        return;
    }

#endif

    // 此时说明上游服务器的响应需要直接转发给客户端
    if (u->header_sent && !u->conf->preserve_output) {
        // 此时，上游响应 header 已经解析完毕，不应该继续向上游发送请求
        // 将写事件回调设置为不做任何工作
        u->write_event_handler = ngx_http_upstream_dummy_handler;

        // 将写事件添加到事件驱动模块中
        (void) ngx_handle_write_event(c->write, 0);

        return;
    }

    // 向上游服务器发送请求 (真正的发送请求)
    ngx_http_upstream_send_request(r, u, 1);
}
```

上述函数大多是在检测请求状态，真正发送请求的功能由 `ngx_http_upstream_send_request()` 完成：

```c
static void
ngx_http_upstream_send_request(ngx_http_request_t *r, ngx_http_upstream_t *u,
    ngx_uint_t do_write)
{
    ngx_int_t          rc;
    ngx_connection_t  *c;

    c = u->peer.connection;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http upstream send request");

    // 统计连接时间
    if (u->state->connect_time == (ngx_msec_t) -1) {
        u->state->connect_time = ngx_current_msec - u->start_time;
    }

    // 测试连接
    if (!u->request_sent && ngx_http_upstream_test_connect(c) != NGX_OK) {
        ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_ERROR);
        return;
    }

    c->log->action = "sending request to upstream";

    // 向上游服务器发送请求
    rc = ngx_http_upstream_send_request_body(r, u, do_write);

    // 错误
    if (rc == NGX_ERROR) {
        ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_ERROR);
        return;
    }

    // 特殊响应 (>= 400 ?)
    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        ngx_http_upstream_finalize_request(r, u, rc);
        return;
    }

    // 还有请求尚未被发送
    if (rc == NGX_AGAIN) {
        // 将写事件添加到定时器中，防止超时
        if (!c->write->ready || u->request_body_blocked) {
            ngx_add_timer(c->write, u->conf->send_timeout);

        } else if (c->write->timer_set) {
            // ?
            ngx_del_timer(c->write);
        }

        // 将写事件添加到事件驱动模块中
        if (ngx_handle_write_event(c->write, u->conf->send_lowat) != NGX_OK) {
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        if (c->write->ready && c->tcp_nopush == NGX_TCP_NOPUSH_SET) {
            if (ngx_tcp_push(c->fd) == -1) {
                ngx_log_error(NGX_LOG_CRIT, c->log, ngx_socket_errno,
                              ngx_tcp_push_n " failed");
                ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }

            c->tcp_nopush = NGX_TCP_NOPUSH_UNSET;
        }

        return;
    }

    /* rc == NGX_OK */

    // 请求发送完成，将写事件从定时器中移除
    if (c->write->timer_set) {
        ngx_del_timer(c->write);
    }

    if (c->tcp_nopush == NGX_TCP_NOPUSH_SET) {
        if (ngx_tcp_push(c->fd) == -1) {
            ngx_log_error(NGX_LOG_CRIT, c->log, ngx_socket_errno,
                          ngx_tcp_push_n " failed");
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        c->tcp_nopush = NGX_TCP_NOPUSH_UNSET;
    }

    // 防止写事件被再次触发，因此将写事件的回调函数修改为不做任何事
    if (!u->conf->preserve_output) {
        u->write_event_handler = ngx_http_upstream_dummy_handler;
    }

    // 重新将写事件添加到事件驱动模块中
    if (ngx_handle_write_event(c->write, 0) != NGX_OK) {
        ngx_http_upstream_finalize_request(r, u,
                                           NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    // 准备开始处理上游服务器的响应
    if (!u->request_body_sent) {
        u->request_body_sent = 1;

        if (u->header_sent) {
            return;
        }

        // 将读事件添加到定时器中，防止超时
        ngx_add_timer(c->read, u->conf->read_timeout);

        // 如果已有响应可以读取，那么开始接收服务器的响应
        if (c->read->ready) {
            ngx_http_upstream_process_header(r, u);
            return;
        }
        // 疑问：为什么没有把读事件添加到事件驱动模块中呢？
        // 显然读事件的就绪是异步的，可能不会立刻完成
    }
}
```

而 `ngx_http_upstream_t` 的读事件回调函数 `ngx_http_upstream_process_header()` 负责接收上游服务器的响应。下一节重点讨论。
