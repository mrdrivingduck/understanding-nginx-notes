# Chapter 12.6 - 12.9 - 转发响应并结束请求

Created by : Mr Dk.

2020 / 08 / 02 15:34

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

### 12.8.1 `ngx_event_pipe_read_upstream()` 函数

该函数的主要任务是，把接收到的上游响应存放到内存或者磁盘文件中，然后用 `ngx_buf_t` 缓冲区指向这些响应，最终用 `in` 和 `out` 缓冲区链表把这些缓冲区管理起来。

```c
static ngx_int_t
ngx_event_pipe_read_upstream(ngx_event_pipe_t *p)
{
    off_t         limit;
    ssize_t       n, size;
    ngx_int_t     rc;
    ngx_buf_t    *b;
    ngx_msec_t    delay;
    ngx_chain_t  *chain, *cl, *ln;

    // 检查上游连接是否结束
    if (p->upstream_eof || p->upstream_error || p->upstream_done) {
        return NGX_OK;
    }

#if (NGX_THREADS)

    if (p->aio) {
        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                       "pipe read upstream: aio");
        return NGX_AGAIN;
    }

    if (p->writing) {
        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                       "pipe read upstream: writing");

        rc = ngx_event_pipe_write_chain_to_temp_file(p);

        if (rc != NGX_OK) {
            return rc;
        }
    }

#endif

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, p->log, 0,
                   "pipe read upstream: %d", p->upstream->read->ready);

    // 开始接收上游响应
    for ( ;; ) {

        // 检查上游连接是否结束
        if (p->upstream_eof || p->upstream_error || p->upstream_done) {
            break;
        }

        // 预读缓冲区为空，或上游没有响应可以接收
        if (p->preread_bufs == NULL && !p->upstream->read->ready) {
            break;
        }

        // 检查预读缓冲区，优先处理这里面的 body
        if (p->preread_bufs) {

            /* use the pre-read bufs if they exist */

            chain = p->preread_bufs;
            p->preread_bufs = NULL;
            n = p->preread_size;

            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, p->log, 0,
                           "pipe preread: %z", n);

            if (n) {
                p->read = 1;
            }

        } else {

            if (p->limit_rate) {
                if (p->upstream->read->delayed) {
                    break;
                }

                limit = (off_t) p->limit_rate * (ngx_time() - p->start_sec + 1)
                        - p->read_length;

                if (limit <= 0) {
                    p->upstream->read->delayed = 1;
                    delay = (ngx_msec_t) (- limit * 1000 / p->limit_rate + 1);
                    ngx_add_timer(p->upstream->read, delay);
                    break;
                }

            } else {
                limit = 0;
            }

            // 如果预读缓冲区为空，则检查 free_raw_bufs 缓冲区链表
            // 如果不为空，那么直接使用
            if (p->free_raw_bufs) {

                /* use the free bufs if they exist */

                chain = p->free_raw_bufs;
                if (p->single_buf) {
                    p->free_raw_bufs = p->free_raw_bufs->next;
                    chain->next = NULL;
                } else {
                    p->free_raw_bufs = NULL;
                }

            } else if (p->allocated < p->bufs.num) {
                // 还能够从内存池中分配到一块新的缓冲区，那么分配一块新缓冲区

                /* allocate a new buf if it's still allowed */

                b = ngx_create_temp_buf(p->pool, p->bufs.size);
                if (b == NULL) {
                    return NGX_ABORT;
                }

                p->allocated++;

                chain = ngx_alloc_chain_link(p->pool);
                if (chain == NULL) {
                    return NGX_ABORT;
                }

                chain->buf = b;
                chain->next = NULL;

            } else if (!p->cacheable
                       && p->downstream->data == p->output_ctx
                       && p->downstream->write->ready
                       && !p->downstream->write->delayed)
            {
                // 下游连接已经就绪，那么向下游发送响应，释放出一些缓冲区

                /*
                 * if the bufs are not needed to be saved in a cache and
                 * a downstream is ready then write the bufs to a downstream
                 */

                p->upstream_blocked = 1;

                ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                               "pipe downstream ready");

                break;

            } else if (p->cacheable
                       || p->temp_file->offset < p->max_temp_file_size)
            {
                // 检查临时文件中已写入的响应内容长度
                // 将 in 缓冲区中的内容写入临时文件，再将 ngx_buf_t 缓冲区从 in 中移入 out 中

                /*
                 * if it is allowed, then save some bufs from p->in
                 * to a temporary file, and add them to a p->out chain
                 */

                rc = ngx_event_pipe_write_chain_to_temp_file(p);

                ngx_log_debug1(NGX_LOG_DEBUG_EVENT, p->log, 0,
                               "pipe temp offset: %O", p->temp_file->offset);

                if (rc == NGX_BUSY) {
                    break;
                }

                if (rc != NGX_OK) {
                    return rc;
                }

                chain = p->free_raw_bufs;
                if (p->single_buf) {
                    p->free_raw_bufs = p->free_raw_bufs->next;
                    chain->next = NULL;
                } else {
                    p->free_raw_bufs = NULL;
                }

            } else {
                // 没有空间用于缓存上游数据了，暂时不再接收上游响应

                /* there are no bufs to read in */

                ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                               "no pipe bufs to read in");

                break;
            }

            // 接收上游响应
            n = p->upstream->recv_chain(p->upstream, chain, limit);

            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, p->log, 0,
                           "pipe recv chain: %z", n);

            // 将带有新数据的缓冲区放置到 free_raw_bufs 链表的头部
            if (p->free_raw_bufs) {
                chain->next = p->free_raw_bufs;
            }
            p->free_raw_bufs = chain;

            // 接收上游数据失败
            if (n == NGX_ERROR) {
                p->upstream_error = 1;
                break;
            }

            if (n == NGX_AGAIN) {
                if (p->single_buf) {
                    ngx_event_pipe_remove_shadow_links(chain->buf);
                }

                break;
            }

            // 置待处理标志位
            p->read = 1;

            if (n == 0) {
                p->upstream_eof = 1;
                break;
            }
        }

        delay = p->limit_rate ? (ngx_msec_t) n * 1000 / p->limit_rate : 0;

        // 开始依次处理 free_raw_bufs 中的每一个缓冲块
        // free_raw_bufs 被清空
        p->read_length += n;
        cl = chain;
        p->free_raw_bufs = NULL;

        // 处理每一个缓冲块
        while (cl && n > 0) {

            // 将缓冲块的 shadows 释放，因为必然不存在多次引用的情况
            ngx_event_pipe_remove_shadow_links(cl->buf);

            // 缓冲块中的剩余长度
            size = cl->buf->end - cl->buf->last;

            // 本次 body 的长度多于缓冲区剩余空间
            if (n >= size) {
                // 缓冲区满了
                cl->buf->last = cl->buf->end;

                /* STUB */ cl->buf->num = p->num++;

                // 调用 input_filter() 函数处理 body
                // 默认的行为是，将这个缓冲区加入到 in 链表中
                if (p->input_filter(p, cl->buf) == NGX_ERROR) {
                    return NGX_ABORT;
                }

                // 当前缓冲块可以被销毁
                // 接着处理下一个缓冲块
                n -= size;
                ln = cl;
                cl = cl->next;
                ngx_free_chain(p->pool, ln);

            } else {
                // 这个缓冲区未满，还可以再次接收响应
                cl->buf->last += n;
                n = 0;
            }
        }

        // 将本次未满的缓冲区放回 free_raw_bufs 中
        if (cl) {
            for (ln = cl; ln->next; ln = ln->next) { /* void */ }

            ln->next = p->free_raw_bufs;
            p->free_raw_bufs = cl;
        }

        if (delay > 0) {
            p->upstream->read->delayed = 1;
            ngx_add_timer(p->upstream->read, delay);
            break;
        }
    }


    if (p->free_raw_bufs && p->length != -1) {
        cl = p->free_raw_bufs;

        if (cl->buf->last - cl->buf->pos >= p->length) {

            p->free_raw_bufs = cl->next;

            /* STUB */ cl->buf->num = p->num++;

            if (p->input_filter(p, cl->buf) == NGX_ERROR) {
                return NGX_ABORT;
            }

            ngx_free_chain(p->pool, cl);
        }
    }

    if (p->length == 0) {
        p->upstream_done = 1;
        p->read = 1;
    }

    // 上游连接已结束
    // 如果 free_raw_bufs 缓冲区链表不为空，则处理其中的数据
    if ((p->upstream_eof || p->upstream_error) && p->free_raw_bufs) {

        /* STUB */ p->free_raw_bufs->buf->num = p->num++;

        // 再次调用 input_filter() 处理最后一个剩余的缓冲区
        if (p->input_filter(p, p->free_raw_bufs->buf) == NGX_ERROR) {
            return NGX_ABORT;
        }

        p->free_raw_bufs = p->free_raw_bufs->next;

        // 需要尽快释放缓冲区中用到的内存
        // 释放 shadow 为空的缓冲区
        if (p->free_bufs && p->buf_to_file == NULL) {
            for (cl = p->free_raw_bufs; cl; cl = cl->next) {
                if (cl->buf->shadow == NULL) {
                    ngx_pfree(p->pool, cl->buf->start);
                }
            }
        }
    }

    if (p->cacheable && (p->in || p->buf_to_file)) {

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                       "pipe write chain");

        rc = ngx_event_pipe_write_chain_to_temp_file(p);

        if (rc != NGX_OK) {
            return rc;
        }
    }

    return NGX_OK;
}
```

### 12.8.2 `ngx_event_pipe_write_to_downstream()` 函数

该函数主要负责把 `in` 和 `out` 两个缓冲区链表中的数据发送给下游客户端。由于 `out` 链表中的缓冲区数据在响应中的位置比 `in` 链表更靠前，所以要被优先发送给下游。当下游的连接处于可写状态时，会尽可能地循环发送 `out` 和 `in` 中的缓冲区。

```c
static ngx_int_t
ngx_event_pipe_write_to_downstream(ngx_event_pipe_t *p)
{
    u_char            *prev;
    size_t             bsize;
    ngx_int_t          rc;
    ngx_uint_t         flush, flushed, prev_last_shadow;
    ngx_chain_t       *out, **ll, *cl;
    ngx_connection_t  *downstream;

    downstream = p->downstream;

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, p->log, 0,
                   "pipe write downstream: %d", downstream->write->ready);

#if (NGX_THREADS)

    if (p->writing) {
        rc = ngx_event_pipe_write_chain_to_temp_file(p);

        if (rc == NGX_ABORT) {
            return NGX_ABORT;
        }
    }

#endif

    flushed = 0;

    for ( ;; ) {
        if (p->downstream_error) {
            return ngx_event_pipe_drain_chains(p);
        }

        // 检查上游连接是否结束
        if (p->upstream_eof || p->upstream_error || p->upstream_done) {

            /* pass the p->out and p->in chains to the output filter */

            for (cl = p->busy; cl; cl = cl->next) {
                cl->buf->recycled = 0;
            }

            // 把 out 链表中的缓冲区发送给下游
            if (p->out) {
                ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                               "pipe write downstream flush out");

                for (cl = p->out; cl; cl = cl->next) {
                    cl->buf->recycled = 0;
                }

                rc = p->output_filter(p->output_ctx, p->out);

                if (rc == NGX_ERROR) {
                    p->downstream_error = 1;
                    return ngx_event_pipe_drain_chains(p);
                }

                p->out = NULL;
            }

            if (p->writing) {
                break;
            }

            // 把 in 链表中的缓冲区发送给下游
            if (p->in) {
                ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                               "pipe write downstream flush in");

                for (cl = p->in; cl; cl = cl->next) {
                    cl->buf->recycled = 0;
                }

                rc = p->output_filter(p->output_ctx, p->in);

                if (rc == NGX_ERROR) {
                    p->downstream_error = 1;
                    return ngx_event_pipe_drain_chains(p);
                }

                p->in = NULL;
            }

            ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                           "pipe write downstream done");

            /* TODO: free unused bufs */

            p->downstream_done = 1;
            break;
        }

        if (downstream->data != p->output_ctx
            || !downstream->write->ready
            || downstream->write->delayed)
        {
            break;
        }

        /* bsize is the size of the busy recycled bufs */

        prev = NULL;
        bsize = 0;

        // 统计 busy 缓冲区中待发送的响应长度，检查是否超过 busy_size 配置
        for (cl = p->busy; cl; cl = cl->next) {

            if (cl->buf->recycled) {
                if (prev == cl->buf->start) {
                    continue;
                }

                bsize += cl->buf->end - cl->buf->start;
                prev = cl->buf->start;
            }
        }

        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, p->log, 0,
                       "pipe write busy: %uz", bsize);

        out = NULL;

        // 如果 ngx_http_request_t 中 out 缓冲区的长度已经超过了 busy_size
        // 那么不再发送 out 和 in 链表中的内容
        if (bsize >= (size_t) p->busy_size) {
            flush = 1;
            goto flush;
        }

        flush = 0;
        ll = NULL;
        prev_last_shadow = 1;

        for ( ;; ) {
            // out 链表中有内容
            if (p->out) {
                cl = p->out;

                if (cl->buf->recycled) {
                    ngx_log_error(NGX_LOG_ALERT, p->log, 0,
                                  "recycled buffer in pipe out chain");
                }

                p->out = p->out->next;

            } else if (!p->cacheable && !p->writing && p->in) {
                // in 链表中有内容
                cl = p->in;

                ngx_log_debug3(NGX_LOG_DEBUG_EVENT, p->log, 0,
                               "pipe write buf ls:%d %p %z",
                               cl->buf->last_shadow,
                               cl->buf->pos,
                               cl->buf->last - cl->buf->pos);

                if (cl->buf->recycled && prev_last_shadow) {
                    // 检查待发送的长度加上当前缓冲块是否超过 busy_size
                    if (bsize + cl->buf->end - cl->buf->start > p->busy_size) {
                        flush = 1;
                        break;
                    }

                    bsize += cl->buf->end - cl->buf->start;
                }

                prev_last_shadow = cl->buf->last_shadow;

                p->in = p->in->next;

            } else {
                break;
            }

            cl->next = NULL;

            if (out) {
                *ll = cl;
            } else {
                out = cl;
            }
            ll = &cl->next;
        }

    flush:

        ngx_log_debug2(NGX_LOG_DEBUG_EVENT, p->log, 0,
                       "pipe write: out:%p, f:%ui", out, flush);

        if (out == NULL) {

            if (!flush) {
                break;
            }

            /* a workaround for AIO */
            if (flushed++ > 10) {
                return NGX_BUSY;
            }
        }

        // 向下游发送缓冲区
        rc = p->output_filter(p->output_ctx, out);

        // 更新 free、busy、out 缓冲区
        ngx_chain_update_chains(p->pool, &p->free, &p->busy, &out, p->tag);

        if (rc == NGX_ERROR) {
            p->downstream_error = 1;
            return ngx_event_pipe_drain_chains(p);
        }

        // 遍历 free 链表，释放其中的 shadow，可以用于接收新的响应
        for (cl = p->free; cl; cl = cl->next) {

            if (cl->buf->temp_file) {
                if (p->cacheable || !p->cyclic_temp_file) {
                    continue;
                }

                /* reset p->temp_offset if all bufs had been sent */

                if (cl->buf->file_last == p->temp_file->offset) {
                    p->temp_file->offset = 0;
                }
            }

            /* TODO: free buf if p->free_bufs && upstream done */

            /* add the free shadow raw buf to p->free_raw_bufs */

            if (cl->buf->last_shadow) {
                if (ngx_event_pipe_add_free_buf(p, cl->buf->shadow) != NGX_OK) {
                    return NGX_ABORT;
                }

                cl->buf->last_shadow = 0;
            }

            cl->buf->shadow = NULL;
        }
    }

    return NGX_OK;
}
```

---

## 12.9 结束 upstream 请求

当 Nginx 与 **上游服务器** 交互出错，或正常处理完毕时，需要结束请求。不能直接使用 `ngx_http_finalize_request()` 来结束请求，因为这个函数用于结束 Nginx 与下游客户端的连接，而 Nginx 与上游服务器的连接将会无法释放。Upstream 机制提供了三种可以结束请求的方式：

- 直接调用 `ngx_http_upstream_finalize_request()` 函数
- 调用 `ngx_http_upstream_cleanup()`
- 调用 `ngx_http_upstream_next()`

在启动 upstream 机制时，`ngx_http_upstream_cleanup()` 函数会注册到请求的 `cleanup` 链表中。这样，HTTP 请求在结束时就会调用 `ngx_http_upstream_cleanup()`。该函数中，实际上还是通过调用 `ngx_http_upstream_finalize_request()` 来结束请求。

```c
static void
ngx_http_upstream_cleanup(void *data)
{
    ngx_http_request_t *r = data;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "cleanup http upstream request: \"%V\"", &r->uri);

    ngx_http_upstream_finalize_request(r, r->upstream, NGX_DONE);
}
```

当处理请求出现错误时，则一般调用 `ngx_http_upstream_next()` 函数。Upstream 机制在这个函数中提供了一个较为灵活的功能：与上游服务器交互发生错误时，Nginx 可以多给上游服务器一些机会，重新向这台或另一台上游服务器发起连接。在该函数中，结束请求前，会检查 `ngx_peer_connection_t` 结构体中的 `tries` 成员 (每个连接的最大重试次数) - 每次出错就减 1，当 `tries` 减到 0 时，才真正调用 `ngx_http_upstream_finalize_request()` 函数结束请求；否则就调用 `ngx_http_upstream_connect()` 函数重新向上游服务器发起请求：

```c
static void
ngx_http_upstream_next(ngx_http_request_t *r, ngx_http_upstream_t *u,
    ngx_uint_t ft_type)
{
    ngx_msec_t  timeout;
    ngx_uint_t  status, state;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http next upstream, %xi", ft_type);

    if (u->peer.sockaddr) {

        if (u->peer.connection) {
            u->state->bytes_sent = u->peer.connection->sent;
        }

        if (ft_type == NGX_HTTP_UPSTREAM_FT_HTTP_403
            || ft_type == NGX_HTTP_UPSTREAM_FT_HTTP_404)
        {
            state = NGX_PEER_NEXT;

        } else {
            state = NGX_PEER_FAILED;
        }

        u->peer.free(&u->peer, u->peer.data, state);
        u->peer.sockaddr = NULL;
    }

    if (ft_type == NGX_HTTP_UPSTREAM_FT_TIMEOUT) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, NGX_ETIMEDOUT,
                      "upstream timed out");
    }

    // ？
    if (u->peer.cached && ft_type == NGX_HTTP_UPSTREAM_FT_ERROR) {
        /* TODO: inform balancer instead */
        u->peer.tries++;
    }

    switch (ft_type) {

    case NGX_HTTP_UPSTREAM_FT_TIMEOUT:
    case NGX_HTTP_UPSTREAM_FT_HTTP_504:
        status = NGX_HTTP_GATEWAY_TIME_OUT;
        break;

    case NGX_HTTP_UPSTREAM_FT_HTTP_500:
        status = NGX_HTTP_INTERNAL_SERVER_ERROR;
        break;

    case NGX_HTTP_UPSTREAM_FT_HTTP_503:
        status = NGX_HTTP_SERVICE_UNAVAILABLE;
        break;

    case NGX_HTTP_UPSTREAM_FT_HTTP_403:
        status = NGX_HTTP_FORBIDDEN;
        break;

    case NGX_HTTP_UPSTREAM_FT_HTTP_404:
        status = NGX_HTTP_NOT_FOUND;
        break;

    case NGX_HTTP_UPSTREAM_FT_HTTP_429:
        status = NGX_HTTP_TOO_MANY_REQUESTS;
        break;

    /*
     * NGX_HTTP_UPSTREAM_FT_BUSY_LOCK and NGX_HTTP_UPSTREAM_FT_MAX_WAITING
     * never reach here
     */

    default:
        status = NGX_HTTP_BAD_GATEWAY;
    }

    if (r->connection->error) {
        ngx_http_upstream_finalize_request(r, u,
                                           NGX_HTTP_CLIENT_CLOSED_REQUEST);
        return;
    }

    u->state->status = status;

    timeout = u->conf->next_upstream_timeout;

    if (u->request_sent
        && (r->method & (NGX_HTTP_POST|NGX_HTTP_LOCK|NGX_HTTP_PATCH)))
    {
        ft_type |= NGX_HTTP_UPSTREAM_FT_NON_IDEMPOTENT;
    }

    // 如果重试次数已经减至 0
    if (u->peer.tries == 0
        || ((u->conf->next_upstream & ft_type) != ft_type)
        || (u->request_sent && r->request_body_no_buffering)
        || (timeout && ngx_current_msec - u->peer.start_time >= timeout))
    {
        // 结束请求
        ngx_http_upstream_finalize_request(r, u, status);
        return;
    }

    if (u->peer.connection) {
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "close http upstream connection: %d",
                       u->peer.connection->fd);

        if (u->peer.connection->pool) {
            ngx_destroy_pool(u->peer.connection->pool);
        }

        ngx_close_connection(u->peer.connection);
        u->peer.connection = NULL;
    }

    // 如果还能允许重试，那么重新发起连接
    ngx_http_upstream_connect(r, u);
}
```

前两个函数到最终都还是调用 `ngx_http_upstream_finalize_request()` 来结束请求。这个函数到最终会调用 HTTP 框架提供的 `ngx_http_finalize_request()` 结束请求，但在这之前需要释放与上游服务器交互时分配的资源。

```c
static void
ngx_http_upstream_finalize_request(ngx_http_request_t *r,
    ngx_http_upstream_t *u, ngx_int_t rc)
{
    ngx_uint_t  flush;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "finalize http upstream request: %i", rc);

    // 请求已经被关闭
    if (u->cleanup == NULL) {
        /* the request was already finalized */
        ngx_http_finalize_request(r, NGX_DONE);
        return;
    }

    // 资源清理回调函数置为 NULL
    *u->cleanup = NULL;
    u->cleanup = NULL;

    // 释放解析主机域名时分配的资源
    if (u->resolved && u->resolved->ctx) {
        ngx_resolve_name_done(u->resolved->ctx);
        u->resolved->ctx = NULL;
    }

    // 设置响应结束时间为当前时间
    if (u->state && u->state->response_time == (ngx_msec_t) -1) {
        u->state->response_time = ngx_current_msec - u->start_time;

        if (u->pipe && u->pipe->read_length) {
            u->state->bytes_received += u->pipe->read_length
                                        - u->pipe->preread_size;
            u->state->response_length = u->pipe->read_length;
        }

        if (u->peer.connection) {
            u->state->bytes_sent = u->peer.connection->sent;
        }
    }

    // 调用 HTTP 模块实现的 upstream 结束时的回调函数
    u->finalize_request(r, rc);

    // TCP 连接池？？
    if (u->peer.free && u->peer.sockaddr) {
        u->peer.free(&u->peer, u->peer.data, 0);
        u->peer.sockaddr = NULL;
    }

    // 如果与上游的 TCP 连接还存在，则关闭这个连接
    if (u->peer.connection) {

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "close http upstream connection: %d",
                       u->peer.connection->fd);

        if (u->peer.connection->pool) {
            ngx_destroy_pool(u->peer.connection->pool);
        }

        ngx_close_connection(u->peer.connection);
    }

    u->peer.connection = NULL;

    if (u->pipe && u->pipe->temp_file) {
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http upstream temp fd: %d",
                       u->pipe->temp_file->file.fd);
    }

    // 如果使用了磁盘文件作为缓存向下游转发文件，那么删除这个临时文件
    if (u->store && u->pipe && u->pipe->temp_file
        && u->pipe->temp_file->file.fd != NGX_INVALID_FILE)
    {
        if (ngx_delete_file(u->pipe->temp_file->file.name.data)
            == NGX_FILE_ERROR)
        {
            ngx_log_error(NGX_LOG_CRIT, r->connection->log, ngx_errno,
                          ngx_delete_file_n " \"%s\" failed",
                          u->pipe->temp_file->file.name.data);
        }
    }

    // 不再响应客户端请求
    r->read_event_handler = ngx_http_block_reading;

    if (rc == NGX_DECLINED) {
        return;
    }

    r->connection->log->action = "sending to client";

    if (!u->header_sent
        || rc == NGX_HTTP_REQUEST_TIME_OUT
        || rc == NGX_HTTP_CLIENT_CLOSED_REQUEST)
    {
        ngx_http_finalize_request(r, rc);
        return;
    }

    flush = 0;

    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        rc = NGX_ERROR;
        flush = 1;
    }

    if (r->header_only
        || (u->pipe && u->pipe->downstream_error))
    {
        ngx_http_finalize_request(r, rc);
        return;
    }

    if (rc == 0) {

        if (ngx_http_upstream_process_trailers(r, u) != NGX_OK) {
            ngx_http_finalize_request(r, NGX_ERROR);
            return;
        }

        rc = ngx_http_send_special(r, NGX_HTTP_LAST);

    } else if (flush) {
        r->keepalive = 0;
        rc = ngx_http_send_special(r, NGX_HTTP_FLUSH);
    }

    // 最终调用 HTTP 框架提供的函数结束请求
    ngx_http_finalize_request(r, rc);
}
```

与缓冲区相关的部分并没有看懂 😭
