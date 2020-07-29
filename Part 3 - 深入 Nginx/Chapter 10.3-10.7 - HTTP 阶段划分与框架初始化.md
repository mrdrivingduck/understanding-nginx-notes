# Chapter 10.3-10.7 - HTTP 阶段划分与框架初始化

Created by : Mr Dk.

2020 / 07 / 29 20:57

Nanjing, Jiangsu, China

---

## 10.3 监听端口的管理

监听端口是由 `server{}` 块下的 `listen` 配置项决定的。每监听一个 TCP 端口，都将使用一个独立的 `ngx_http_conf_port_t` 结构体来表示，这个结构体由全局配置 `ngx_http_core_main_conf_t` 引用：

```c
typedef struct {
    ngx_array_t                servers;         /* ngx_http_core_srv_conf_t */

    ngx_http_phase_engine_t    phase_engine;

    ngx_hash_t                 headers_in_hash;

    ngx_hash_t                 variables_hash;

    ngx_array_t                variables;         /* ngx_http_variable_t */
    ngx_array_t                prefix_variables;  /* ngx_http_variable_t */
    ngx_uint_t                 ncaptures;

    ngx_uint_t                 server_names_hash_max_size;
    ngx_uint_t                 server_names_hash_bucket_size;

    ngx_uint_t                 variables_hash_max_size;
    ngx_uint_t                 variables_hash_bucket_size;

    ngx_hash_keys_arrays_t    *variables_keys;

    ngx_array_t               *ports; // 所有的监听端口！！！

    ngx_http_phase_t           phases[NGX_HTTP_LOG_PHASE + 1];
} ngx_http_core_main_conf_t;
```

其中，`ports` 数组表示服务器监听的所有端口。每个端口的信息形式如下：

```c
typedef struct {
    ngx_int_t                  family; // socket 地址家族
    in_port_t                  port; // 监听的端口号
    ngx_array_t                addrs;     /* array of ngx_http_conf_addr_t */
} ngx_http_conf_port_t;
```

其中的 `addrs` 动态数组指向了监听的这个端口的所有 **地址**。因为在 Nginx 中支持一台物理主机有多个 IP 地址。由此，一个 `ngx_http_conf_port_t` 将会对应多个 `ngx_http_conf_addr_t`：

```c
typedef struct {
    ngx_http_listen_opt_t      opt; // 监听 socket 的属性

    // 索引到 server{} 块下的配置，决定使用哪个虚拟主机的配置
    // key 是 server name，value 是 ngx_http_core_srv_conf_t 结构体
    ngx_hash_t                 hash; // 完全匹配 server name 的 hash 表
    ngx_hash_wildcard_t       *wc_head; // 通配符前置的 hash 表
    ngx_hash_wildcard_t       *wc_tail; // 通配符后置的 hash 表

#if (NGX_PCRE)
    ngx_uint_t                 nregex;
    ngx_http_server_name_t    *regex; // 静态数组，正则表达式及其匹配的 server{} 虚拟主机
#endif

    /* the default server configuration for this address:port */
    ngx_http_core_srv_conf_t  *default_server; // 该监听端口下默认的 server{} 虚拟主机
    // 所有 server{} 虚拟主机对应的配置项
    ngx_array_t                servers;  /* array of ngx_http_core_srv_conf_t */
} ngx_http_conf_addr_t;
```

对每一个 `ngx_http_conf_addr_t` (监听地址) 来说，都会有一个 `ngx_listening_t` 与其对应 - 其 `handler` 回调函数被设置为 `ngx_http_init_connection()` - 即 TCP 连接建立成功后，调用该函数初始化 HTTP 相关信息。

## 10.4 Server (主机名) 的快速检索

每个 `server{}` 虚拟主机的配置项都由一个 `ngx_http_core_src_conf_t` 结构体来表示，所有这样的结构体由全局配置结构体中的 `servers` 动态数组组织起来。当处理 HTTP 连接时，接收到 HTTP header 并取到 Host 后，理论上需要遍历动态数组，才能找到 server name 匹配的 `server{}` 块 - 时间复杂度太高！

Nginx 将所有 `server{}` 配置项结构体组织为 hash 表。Key 是 server name 字符串，value 是 `ngx_http_conf_addr_t` 结构体。这样能够快速检索相应的配置项。

## 10.5 Location (URI) 的快速检索

每个 `server{}` 块下都有多个 `location{}` 块，`location{}` 块中还可以嵌套 `location{}` 块。每一批 `location{}` 块由 **双向链表** 与父配置块 (`server{}` 或 `location{}`) 关联。那么在检索某个 URI 时，显然也需要遍历所有的 loc 配置块才行 - 时间复杂度太高！

Nginx 将双向链表中的所有的 loc 配置项按照 **匹配字符串** 进行排序。针对排序后的双向链表，快速构建 **静态的二叉查找树**。由于 `location{}` 块是可以递归的，因此这个建树操作也是递归的。

为什么没有构造红黑树呢？因为配置项在启动以后肯定是静态不变的，因此可以使用静态的完全平衡二叉树来组织，效率上会比红黑树更好。红黑树是查询效率和修改效率的折衷。

---

## 10.6 HTTP 请求的 11 个处理阶段

Nginx 的模块化设计使得每个 HTTP 模块专注于完成一个独立、简单的功能，一个请求的处理可以由无数多个 HTTP 模块共同完成。但是，如果模块处理请求的顺序是单一的，将无法满足灵活性需求。每一个处理请求的模块很难灵活地指定下一个处理模块是哪个。因此，HTTP 框架将处理流程划分为 11 个阶段，其中每个阶段都可以由任意多个 HTTP 模块流水式地处理请求。这 11 个阶段是有顺序的，有些阶段是必备的，有些阶段是可选的，也可以有多个模块介入同一阶段。

在全局配置项 `ngx_http_core_main_conf_t` 结构体中，有两个相关结构体：

```c
typedef struct {
    ngx_http_phase_engine_t phase_engine; // 各阶段的处理函数组成的数组
    ngx_http_phase_t phases[NGX_HTTP_LOG_PHASE + 1]; // 初始化时帮助 HTTP 模块添加处理函数
} ngx_http_core_main_conf_t;
```

其中，`phases` 是一个数组，数组中的每个元素都对应了一个 HTTP 阶段。每个元素的定义如下，含义为，每一个 HTTP 模块初始化时，添加到 **该阶段** 的所有处理函数。任何 HTTP 模块都可以在初始化时通过 `ngx_http_module_t` 接口的 `postconfiguration()` 函数将自定义的函数添加到 `handlers` 数组中：

```c
typedef struct {
    ngx_array_t                handlers;
} ngx_http_phase_t;
```

而 `phase_engine` 是一个 `ngx_http_phase_engine_t` 结构体。在初始化后，HTTP 框架将每个阶段的处理函数全部添加到下面的 `handlers` 中，表示一个请求可能经历的所有处理函数：

```c
typedef struct {
    ngx_http_phase_handler_t  *handlers;
    // NGX_HTTP_SERVER_REWRITE_PHASE 阶段的第一个处理函数的 index
    ngx_uint_t                 server_rewrite_index;
    // NGX_HTTP_REWRITE_PHASE 阶段的第一个处理函数的 index
    ngx_uint_t                 location_rewrite_index;
} ngx_http_phase_engine_t;
```

而具体每一个处理函数又是如何定义的呢？

```c
typedef struct ngx_http_phase_handler_s  ngx_http_phase_handler_t;

struct ngx_http_phase_handler_s {
    ngx_http_phase_handler_pt  checker;
    ngx_http_handler_pt        handler;
    ngx_uint_t                 next; // 下一个 HTTP 处理阶段的序号
};

typedef ngx_int_t (*ngx_http_phase_handler_pt)(ngx_http_request_t *r, ngx_http_phase_handler_t *ph);
typedef ngx_int_t (*ngx_http_handler_pt)(ngx_http_request_t *r);
```

处理函数中定义了 `check()` 函数和 `handler()` 函数。HTTP 框架会在 `checker()` 函数已实现的前提下调用 `checker()` 函数来处理请求，在其中调用 `handler()` 函数 (而不是直接调用 `handler()`)。所有的 `checker()` 函数都是由 HTTP 的核心模块实现的，普通的 HTTP 模块无法重定义这个函数；除 HTTP 核心模块以外的模块，只能通过定义 `handler()` 函数介入请求处理的某个阶段。

### 10.6.2 NGX_HTTP_POST_READ_PHASE 阶段

当 HTTP 框架在建立的 TCP 连接上接收到完整的 HTTP header 时，开始执行这个阶段的 `checker()` 函数。该阶段的 `checker()` 函数如下：

```c
ngx_int_t
ngx_http_core_generic_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    ngx_int_t  rc;

    /*
     * generic phase checker,
     * used by the post read and pre-access phases
     */

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "generic phase: %ui", r->phase_handler);

    rc = ph->handler(r); // 调用各 HTTP 模块添加的 handler 函数

    if (rc == NGX_OK) {
        // 进入下一个 HTTP 阶段，忽略当前阶段的其它 handler
        r->phase_handler = ph->next;
        return NGX_AGAIN;
    }

    if (rc == NGX_DECLINED) {
        // 进入下一个 handler (可能是当前阶段，也可能是下一个阶段)
        r->phase_handler++;
        return NGX_AGAIN;
    }

    if (rc == NGX_AGAIN || rc == NGX_DONE) {
        // 请求依旧停留在当前阶段中
        return NGX_OK;
    }

    /* rc == NGX_ERROR || rc == NGX_HTTP_...  */

    // 如果返回上述注释中的情况，那么结束请求
    ngx_http_finalize_request(r, rc);

    return NGX_OK;
}
```

由此可见，`handler()` 的返回值会对 HTTP 的请求处理流程造成影响。在不同的 HTTP 处理阶段中，相同的返回值可能会有不一样的效果。

| 返回值         | 意义和行为                                           |
| -------------- | ---------------------------------------------------- |
| `NGX_OK`       | 直接进入下一阶段，当前阶段的其它处理函数被忽略       |
| `NGX_DECLINED` | 按数组顺序执行下一个处理函数                         |
| `NGX_AGAIN`    | 当前处理函数有机会再次被调用，控制权被转交给事件模块 |
| `NGX_DONE`     | 同上                                                 |
| `NGX_ERROR`    | 调用 `ngx_http_finalize_request()` 结束请求          |
| 其它           | 同上                                                 |

以一个官方模块 `ngx_http_realip_module` 为例，它的 `postconfiguration()` 函数中会把自定义的处理函数介入这个阶段。具体方式是，把处理函数添加到 `ngx_http_core_main_conf_t` 中 `phases[NGX_HTTP_POST_READ_PHASE]` 的 `handlers` 动态数组中：

```c
static ngx_int_t
ngx_http_realip_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    // 获取全局配置项
    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    // 获取该阶段的 handlers 动态数组
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_POST_READ_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    // 添加 handler
    *h = ngx_http_realip_handler;

    // 这里把同一个函数添加到了另一个阶段中，这样是完全可以的
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_realip_handler;

    return NGX_OK;
}
```

### 10.6.3 NGX_HTTP_SERVER_REWRITE_PHASE 阶段

该阶段用于在 URI 与 location 表达式匹配前，修改 URI (即所谓重定向)。该阶段不存在返回值可以使流程直接进入下一阶段。

| 返回值         | 意义和行为                           |
| -------------- | ------------------------------------ |
| `NGX_DONE`     | 当前阶段尚未结束，控制权返回当前阶段 |
| `NGX_DECLINED` | 执行数组中的下一个处理函数           |
| `NGX_AGAIN`    | 结束请求                             |
| `NGX_DONE`     | 同上                                 |
| `NGX_ERROR`    | 同上                                 |
| 其它           | 同上                                 |

### 10.6.4 NGX_HTTP_FIND_CONFIG_PHASE 阶段

这个阶段不能被跳过，主要工作是根据请求的 URI 匹配 location 表达式，即使用 location 组成的静态平衡二叉树进行快速检索。任何 HTTP 模块无法向这一个阶段添加处理函数。

### 10.6.5 NGX_HTTP_REWRITE_PHASE 阶段

这一阶段使请求在检索到 location 之后有机会再次修改 URL。因此与 `NGX_HTTP_SERVER_REWRITE_PHASE` 阶段逻辑类似，返回值含义也相同。

### 10.6.6 NGX_HTTP_POST_REWRITE_PHASE 阶段

这一阶段也只能由 HTTP 框架实现，不允许 HTTP 模块向该阶段添加自定义的处理函数。主要工作是检查 `rewrite()` 重写 URL 的次数是否超过 10 次，防止死循环。

### 10.6.7 NGX_HTTP_PREACCESS_PHASE 阶段

该阶段用于在决定请求的访问权限前，为 HTTP 模块提供机会介入。它的 `checker()` 函数也是 `ngx_http_core_generic_phase()`，因此返回值含义与 `NGX_HTTP_POST_READ_PHASE` 完全相同。

### 10.6.8 NGX_HTTP_ACCESS_PHASE 阶段

该阶段用于让 HTTP 模块判断是否允许这个请求访问 Nginx 服务器。这个阶段使用了新的 `checker()` 函数，因此返回值有了新的含义：

```c
ngx_int_t
ngx_http_core_access_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    ngx_int_t                  rc;
    ngx_http_core_loc_conf_t  *clcf;

    if (r != r->main) {
        r->phase_handler = ph->next;
        return NGX_AGAIN;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "access phase: %ui", r->phase_handler);

    // 调用处理函数
    rc = ph->handler(r);

    // 按数组顺序执行下一个处理函数
    if (rc == NGX_DECLINED) {
        r->phase_handler++;
        return NGX_AGAIN;
    }

    // 控制权交还给事件模块，有机会被再次调用
    if (rc == NGX_AGAIN || rc == NGX_DONE) {
        return NGX_OK;
    }

    // 取得全局配置项
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    // 配置项中的 satisfy 为 satisfy all
    // 也就是说，需要满足每个处理函数
    if (clcf->satisfy == NGX_HTTP_SATISFY_ALL) {

        // 按顺序执行下一个处理函数
        if (rc == NGX_OK) {
            r->phase_handler++;
            return NGX_AGAIN;
        }

    } else {
        // satisfy any，即在该阶段满足一个处理函数就 ok 了
        
        // 直接执行下一个阶段的第一个处理函数
        if (rc == NGX_OK) {
            r->access_code = 0;

            if (r->headers_out.www_authenticate) {
                r->headers_out.www_authenticate->hash = 0;
            }

            r->phase_handler = ph->next;
            return NGX_AGAIN;
        }

        // 设置 access_code，然后按顺序执行下一个处理函数
        if (rc == NGX_HTTP_FORBIDDEN || rc == NGX_HTTP_UNAUTHORIZED) {
            if (r->access_code != NGX_HTTP_UNAUTHORIZED) {
                r->access_code = rc;
            }

            r->phase_handler++;
            return NGX_AGAIN;
        }
    }

    /* rc == NGX_ERROR || rc == NGX_HTTP_...  */

    // 到这里只可能被配置为 satisfy all，那么结束请求
    if (rc == NGX_HTTP_UNAUTHORIZED) {
        return ngx_http_core_auth_delay(r);
    }

    ngx_http_finalize_request(r, rc);
    return NGX_OK;
}
```

返回值含义如代码中所示，就不列表了。这里可以看到，处理流程与配置文件中的 `satisfy` 配置项有着密切关系。

### 10.6.9 NGX_HTTP_POST_ACCESS_PHASE 阶段

这个阶段负责给上一个阶段收尾，向用户发送拒绝服务的响应。这个阶段只能由 HTTP 框架实现。该阶段的 `checker()` 函数如下：

```c
ngx_int_t
ngx_http_core_post_access_phase(ngx_http_request_t *r,
    ngx_http_phase_handler_t *ph)
{
    ngx_int_t  access_code;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "post access phase: %ui", r->phase_handler);

    // 获得前一阶段中设置的 access code
    access_code = r->access_code;

    // 如果 access code 不为 0，那么没有访问权限，结束请求
    if (access_code) {
        r->access_code = 0;

        if (access_code == NGX_HTTP_FORBIDDEN) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "access forbidden by rule");
        }

        if (access_code == NGX_HTTP_UNAUTHORIZED) {
            return ngx_http_core_auth_delay(r);
        }

        ngx_http_finalize_request(r, access_code);
        return NGX_OK;
    }

    // 否则继续执行下一个处理函数
    r->phase_handler++;
    return NGX_AGAIN;
}
```

### 10.6.10 NGX_HTTP_TRY_FILES_PHASE 阶段

这个阶段用于让 HTTP 请求访问静态资源，与 `try_files` 配置项密切相关。模块只能由 HTTP 框架实现。如果指定的静态文件资源中有一个可以访问，那么读取文件并发送响应给用户，不再执行后续阶段；如果所有静态文件资源都无法访问，那么再继续执行下一个处理函数。

> 源代码咋没找到？？？

### 10.6.11 NGX_HTTP_CONTENT_PHASE 阶段

这是一个核心的 HTTP 阶段，大部分 HTTP 模块都愿意在这个阶段介入请求处理。原因有二：

1. 这样可以复用之前的所有基础性阶段的功能
2. 该阶段向 HTTP 模块提供了两种介入方式

第一种方式，也就是与其它阶段相同的方式，通过 `postconfiguration()` 函数向全局配置结构体中的 `phases[NGX_HTTP_CONTENT_PHASE]` 结构体的动态数组中添加处理函数。这样，处理函数会应用于 **全部** 的 HTTP 请求。

第二种方式，设置 `ngx_http_core_loc_conf_t` 结构体 (即每一个 `location{}` 块的配置结构体) 中的 `handler` 指针实现。这样，处理函数不再应用于所有的 HTTP 请求，仅当请求的 URI 匹配 `location{}` 块时才会被调用。

> 由于 `location{}` 块的配置结构体中的 `handler` 是一个指针，而不是一个数组。因此，如果使用第二种方式介入 HTTP 请求的处理流程，在这个阶段将只能执行这一个处理函数！如果使用第一种方式，那么该阶段可以经由任意多个 HTTP 模块处理。
>
> 如果同时设置了两种处理方式，第二种方式的优先级更高，第一种方式的处理函数将不会生效。如果有多个 HTTP 模块都试图用第二种方式介入这一阶段，那么后面的配置项将有可能覆盖前面的配置项中的 `handler` 指针。

该阶段的 `checker()` 函数如下 (暂未读懂，之后再加注释)：

```c
ngx_int_t
ngx_http_core_content_phase(ngx_http_request_t *r,
    ngx_http_phase_handler_t *ph)
{
    size_t     root;
    ngx_int_t  rc;
    ngx_str_t  path;

    if (r->content_handler) {
        r->write_event_handler = ngx_http_request_empty_handler;
        ngx_http_finalize_request(r, r->content_handler(r));
        return NGX_OK;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "content phase: %ui", r->phase_handler);

    rc = ph->handler(r);

    if (rc != NGX_DECLINED) {
        ngx_http_finalize_request(r, rc);
        return NGX_OK;
    }

    /* rc == NGX_DECLINED */

    ph++;

    if (ph->checker) {
        r->phase_handler++;
        return NGX_AGAIN;
    }

    /* no content handler was found */

    if (r->uri.data[r->uri.len - 1] == '/') {

        if (ngx_http_map_uri_to_path(r, &path, &root, 0) != NULL) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "directory index of \"%s\" is forbidden", path.data);
        }

        ngx_http_finalize_request(r, NGX_HTTP_FORBIDDEN);
        return NGX_OK;
    }

    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "no handler found");

    ngx_http_finalize_request(r, NGX_HTTP_NOT_FOUND);
    return NGX_OK;
}
```

### 10.6.12 NGX_HTTP_LOG_PHASE 阶段

这个阶段用于记录日志，并做一些共性的收尾工作。使用的 `checker()` 函数为 `ngx_http_core_generic_phase()`。

---

## 10.7 HTTP 框架的初始化流程

当配置文件中出现了 `http{}` 块时，就回调 `ngx_http_block()` 函数，其中包含了 HTTP 框架的初始化流程：

1. 由 0 开始递增地设置所有 HTTP 模块地 `ctx_index` 字段 (第几个 HTTP 模块)
2. 初始化到 `http{}` 块时，产生 `ngx_http_conf_ctx_t` 结构体 (里面包含三个等级的配置项数组指针)
3. 依次调用所有模块的 `create_main_conf()` 函数，初始化 `main_conf` 数组
4. 依次调用所有模块的 `create_srv_conf()` 函数，初始化 `srv_conf` 数组
5. 依次调用所有模块的 `create_loc_conf()` 函数，初始化 `loc_conf` 数组
6. 依次调用所有模块的 `preconfiguration()` 函数
7. 解析 `http{}` 块下的 main 级别配置项 - 其中会遇到 `server{}` 块、`location{}` 块，都会触发相应的解析函数
8. 依次调用所有模块的 `init_main_conf()` 函数
9. 调用 `ngx_http_merge_servers()` 函数合并配置项
10. 创建 `location{}` 块组成的平衡 BST
11. 将 `phase[]` 中可以由用户自行定义处理函数的 7 个阶段的 `handlers[]` 数组初始化
12. 依次调用所有模块的 `postconfiguration()` 函数，将各阶段介入的 handler 加入 `phase[]` 中
13. 遍历每个阶段的 `handlers`，将所有的处理函数构成 `phase_engine.handlers[]` 数组，放入全局配置结构体中
14. 构造 `server{}` 中的虚拟主机通配符 hash 表
15. 构造监听端口与 server 之间的联系，设置新连接回调函数为 `ngx_http_init_connection()`

---

