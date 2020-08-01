# Chapter 12.6 - 12.9 - 转发响应并结束请求

Created by : Mr Dk.

2020 / 08 / 01 21:21

Nanjing, Jiangsu, China

---

## 12.6 不转发响应时的处理流程

> 没找到代码...

## 12.7 以下游网速优先来转发响应

如果下游网速优先，意味着上游服务器的响应数据能够很快被下游客户端取走。因此 Nginx 只需要开一个固定大小的内存缓冲区，存放上游服务器的响应数据。如果缓冲区满，那么暂停接收上游服务器的响应，等到缓冲区中的响应发送给下游客户端后，缓冲区自然会清空。

这种设计的优势在于，不会使用大量内存，也不会使用到磁盘文件。能够提高并发量，降低服务器负载。

### 12.7.1 转发响应 header

HTTP 模块解析 header 后，将解析出的值设置到 `ngx_http_upstream_t` 的 `header_in` 成员中，之后会将 `header_in` 中的 header 设置到 `header_out` 中。这些 header 最终会被发送给客户端。

### 12.7.2 转发响应 body

在上一节已经总结，对于下游网速优先，将不使用磁盘文件缓存响应。上游服务器的读事件回调函数与下游客户端的写事件回调函数分别为 `ngx_http_upstream_process_non_buffered_upstream()` 和 `ngx_http_upstream_process_non_buffered_downstream()`。

```c
static void
ngx_http_upstream_process_non_buffered_upstream(ngx_http_request_t *r,
    ngx_http_upstream_t *u)
{
    ngx_connection_t  *c;

    // 获取 Nginx 与上游服务器的连接
    c = u->peer.connection;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http upstream process non buffered upstream");

    c->log->action = "reading upstream";

    // 如果读事件已经超时，则结束请求
    if (c->read->timedout) {
        ngx_connection_error(c, NGX_ETIMEDOUT, "upstream timed out");
        ngx_http_upstream_finalize_request(r, u, NGX_HTTP_GATEWAY_TIME_OUT);
        return;
    }

    ngx_http_upstream_process_non_buffered_request(r, 0);
}
```

```c
static void
ngx_http_upstream_process_non_buffered_downstream(ngx_http_request_t *r)
{
    ngx_event_t          *wev;
    ngx_connection_t     *c;
    ngx_http_upstream_t  *u;

    // Nginx 与下游客户端的连接
    c = r->connection;
    // Nginx 与上游服务器的连接
    u = r->upstream;
    wev = c->write;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http upstream process non buffered downstream");

    c->log->action = "sending to client";

    // Nginx 与下游客户端的写事件已超时
    if (wev->timedout) {
        c->timedout = 1;
        ngx_connection_error(c, NGX_ETIMEDOUT, "client timed out");
        ngx_http_upstream_finalize_request(r, u, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }

    ngx_http_upstream_process_non_buffered_request(r, 1);
}
```

可以看到读写回调函数最终都调用了 `ngx_http_upstream_process_non_buffered_request()`，只是第二个参数不同：

```c
static void
ngx_http_upstream_process_non_buffered_request(ngx_http_request_t *r,
    ngx_uint_t do_write)
{
    size_t                     size;
    ssize_t                    n;
    ngx_buf_t                 *b;
    ngx_int_t                  rc;
    ngx_uint_t                 flags;
    ngx_connection_t          *downstream, *upstream;
    ngx_http_upstream_t       *u;
    ngx_http_core_loc_conf_t  *clcf;

    u = r->upstream;
    downstream = r->connection;
    upstream = u->peer.connection;

    b = &u->buffer;

    // do_write 表示本次是否向下游发送响应
    do_write = do_write || u->length == 0;

    // Nginx 与上下游的通信会在这个循环中反复穿插进行
    for ( ;; ) {

        // 向下游发送响应
        if (do_write) {

            // 是否还有未转发给下游的响应数据
            // 直接发送 out_bufs 缓冲区中的内容即可
            if (u->out_bufs || u->busy_bufs || downstream->buffered) {
                // 发送数据
                rc = ngx_http_output_filter(r, u->out_bufs);

                if (rc == NGX_ERROR) {
                    ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
                    return;
                }

                // 更新缓冲区链表
                ngx_chain_update_chains(r->pool, &u->free_bufs, &u->busy_bufs,
                                        &u->out_bufs, u->output.tag);
            }

            if (u->busy_bufs == NULL) {

                if (u->length == 0
                    || (upstream->read->eof && u->length == -1))
                {
                    ngx_http_upstream_finalize_request(r, u, 0);
                    return;
                }

                if (upstream->read->eof) {
                    ngx_log_error(NGX_LOG_ERR, upstream->log, 0,
                                  "upstream prematurely closed connection");

                    ngx_http_upstream_finalize_request(r, u,
                                                       NGX_HTTP_BAD_GATEWAY);
                    return;
                }

                if (upstream->read->error || u->error) {
                    ngx_http_upstream_finalize_request(r, u,
                                                       NGX_HTTP_BAD_GATEWAY);
                    return;
                }

                b->pos = b->start;
                b->last = b->start;
            }
        }

        // 计算缓冲区中的剩余空间
        size = b->end - b->last;

        if (size && upstream->read->ready) {

            // 将上游响应读取到缓冲区中
            n = upstream->recv(upstream, b->last, size);
            // 没读取完，下次继续
            if (n == NGX_AGAIN) {
                break;
            }

            if (n > 0) {
                u->state->bytes_received += n;
                u->state->response_length += n;

                // 调用 input_filter() 处理 body
                if (u->input_filter(u->input_filter_ctx, n) == NGX_ERROR) {
                    ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
                    return;
                }
            }

            // 设置可转发的标志位
            do_write = 1;

            continue;
        }

        break;
    }

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    // 将写事件添加到事件驱动模块中
    if (downstream->data == r) {
        if (ngx_handle_write_event(downstream->write, clcf->send_lowat)
            != NGX_OK)
        {
            ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
            return;
        }
    }

    // 将写事件添加到定时器中
    if (downstream->write->active && !downstream->write->ready) {
        ngx_add_timer(downstream->write, clcf->send_timeout);

    } else if (downstream->write->timer_set) {
        ngx_del_timer(downstream->write);
    }

    if (upstream->read->eof || upstream->read->error) {
        flags = NGX_CLOSE_EVENT;

    } else {
        flags = 0;
    }

    // 将读事件添加到事件驱动模块中
    if (ngx_handle_read_event(upstream->read, flags) != NGX_OK) {
        ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
        return;
    }

    // 将读事件添加到定时器中
    if (upstream->read->active && !upstream->read->ready) {
        ngx_add_timer(upstream->read, u->conf->read_timeout);

    } else if (upstream->read->timer_set) {
        ngx_del_timer(upstream->read);
    }
}
```

## 12.8 以上游网速优先来转发响应

如果要保证上游网速优先，那么 Nginx 所在机器必然会缓存大量的上游服务器响应，因为下游客户端来不及取走。当配置中的 `buffering` 为 `1` 时，先会使用 `bufs.num * bufs.size` 的内存缓冲区，如果不够用时，还会再使用最大不超过 `max_temp_file_size` 字节的临时文件缓存响应。

`ngx_event_pipe_t` 是这种缓存转发方式的核心结构体，这个结构也需要由 HTTP 模块提前创建，维护着上下游之间转发的响应 body。其中的缓冲区管理非常复杂 - 因为 Nginx 追求高效率，因此 **绝不对把相同的内容复制到两块内存中**。同一块内存可能同时用于接收上游响应、向下游转发响应、写入临时文件。说白了，就是一大堆复杂的指针操作，在不多分配内存的条件下，尽最大可能复用一块已有的内存。

比如，在转发 header 时，会初始化 `preread_bufs` 预读缓冲区链表。所谓预读，就是在读取 header 时也顺便读取到了一部分的 body。这个链表的缓冲区并不分配内存，而是直接使用其中的 `ngx_buf_t` 结构体指向已经接收到的响应 body 对应的内存。这就是不重复使用内存的原则。

与下游网速优先的响应转发处理类似，为上游服务器的读事件与下游客户端的写事件分别设置了回调函数 `ngx_http_upstream_process_upstream()` 和 `ngx_http_upstream_process_downstream()`：

```c
static void
ngx_http_upstream_process_upstream(ngx_http_request_t *r,
    ngx_http_upstream_t *u)
{
    ngx_event_t       *rev;
    ngx_event_pipe_t  *p;
    ngx_connection_t  *c;

    // 获得 Nginx 与上游服务器的连接
    c = u->peer.connection;
    p = u->pipe;
    // 获得读事件
    rev = c->read;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http upstream process upstream");

    c->log->action = "reading upstream";

    // 读事件已超时
    if (rev->timedout) {

        p->upstream_error = 1;
        ngx_connection_error(c, NGX_ETIMEDOUT, "upstream timed out");

    } else {

        if (rev->delayed) {

            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                           "http upstream delayed");

            if (ngx_handle_read_event(rev, 0) != NGX_OK) {
                ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
            }

            return;
        }

        // ...
        if (ngx_event_pipe(p, 0) == NGX_ABORT) {
            ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
            return;
        }
    }

    ngx_http_upstream_process_request(r, u);
}
```

```c
static void
ngx_http_upstream_process_downstream(ngx_http_request_t *r)
{
    ngx_event_t          *wev;
    ngx_connection_t     *c;
    ngx_event_pipe_t     *p;
    ngx_http_upstream_t  *u;

    c = r->connection;
    u = r->upstream;
    p = u->pipe;
    wev = c->write;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http upstream process downstream");

    c->log->action = "sending to client";

#if (NGX_THREADS)
    p->aio = r->aio;
#endif

    // 写事件已超时
    if (wev->timedout) {

        p->downstream_error = 1;
        c->timedout = 1;
        ngx_connection_error(c, NGX_ETIMEDOUT, "client timed out");

    } else {

        if (wev->delayed) {

            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                           "http downstream delayed");

            if (ngx_handle_write_event(wev, p->send_lowat) != NGX_OK) {
                ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
            }

            return;
        }

        // ...
        if (ngx_event_pipe(p, 1) == NGX_ABORT) {
            ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
            return;
        }
    }

    ngx_http_upstream_process_request(r, u);
}
```

其中用到了两个公共的函数。`ngx_event_pipe()` 实现了缓存响应的功能，可以看到第二个参数体现了读写的区别：

```c
ngx_int_t
ngx_event_pipe(ngx_event_pipe_t *p, ngx_int_t do_write)
{
    ngx_int_t     rc;
    ngx_uint_t    flags;
    ngx_event_t  *rev, *wev;

    for ( ;; ) {
        // 检测 do_write 标志位，决定了是接收上游响应，还是向下游转发
        if (do_write) {
            p->log->action = "sending to client";

            // 向下游发送响应
            rc = ngx_event_pipe_write_to_downstream(p);

            if (rc == NGX_ABORT) {
                return NGX_ABORT;
            }

            if (rc == NGX_BUSY) {
                return NGX_OK;
            }
            
            // 以上两个返回值，不会再向下执行
        }

        p->read = 0;
        p->upstream_blocked = 0;

        p->log->action = "reading upstream";

        // 读取上游服务器响应
        if (ngx_event_pipe_read_upstream(p) == NGX_ABORT) {
            return NGX_ABORT;
        }

        // 如果这两个标志位有一个为 1，那么就将 do_write 置为 1，继续向下游发送响应
        if (!p->read && !p->upstream_blocked) {
            break;
        }

        do_write = 1;
    }

    if (p->upstream->fd != (ngx_socket_t) -1) {
        rev = p->upstream->read;

        flags = (rev->eof || rev->error) ? NGX_CLOSE_EVENT : 0;

        // 将上游读事件添加到事件驱动模块中
        if (ngx_handle_read_event(rev, flags) != NGX_OK) {
            return NGX_ABORT;
        }

        // 将上游读事件添加到定时器中
        if (!rev->delayed) {
            if (rev->active && !rev->ready) {
                ngx_add_timer(rev, p->read_timeout);

            } else if (rev->timer_set) {
                ngx_del_timer(rev);
            }
        }
    }

    if (p->downstream->fd != (ngx_socket_t) -1
        && p->downstream->data == p->output_ctx)
    {
        wev = p->downstream->write;
        
        // 将下游写事件添加到事件驱动模块中
        if (ngx_handle_write_event(wev, p->send_lowat) != NGX_OK) {
            return NGX_ABORT;
        }

        // 将下游写事件添加到定时器中
        if (!wev->delayed) {
            if (wev->active && !wev->ready) {
                ngx_add_timer(wev, p->send_timeout);

            } else if (wev->timer_set) {
                ngx_del_timer(wev);
            }
        }
    }

    return NGX_OK;
}
```

以上函数，通过调用 `ngx_event_pipe_write_to_downstream()` 和 `ngx_event_pipe_read_upstream()`，屏蔽了缓存细节。