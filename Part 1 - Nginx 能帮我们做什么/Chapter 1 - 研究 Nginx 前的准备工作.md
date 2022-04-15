# Chapter 1 - 研究 Nginx 前的准备工作

Created by : Mr Dk.

2020 / 07 / 16 15:47

Nanjing, Jiangsu, China

---

## Nginx 是什么？

Nginx - 2012 Cloud Award for Developer of the Year，世界第二大 Web 服务器。业界高性能 Web 服务器的代名词。

作为 Web 服务器的基本功能：基于 REST 架构风格，以 URI 或 URL 作为沟通依据，通过 HTTP 为浏览器等客户端程序提供各种网络服务。目前 Nginx 与其它 Web 服务器的差别如下：

- Tomcat / Jetty 是重量级服务器，与 Nginx 没有可比性
- IIS 只能在 Windows OS 上运行
- Apache 比较中重量级，不支持高并发 (进程量过多)

Nginx 以性能为王，跨平台，架构适合高并发，并使用当前 OS 中的高效 API 来提高性能。特点：

1. 请求响应速度快
2. 高扩展性 - 由多个不同功能、不同层次、不同类型、耦合度低的模块组成，且模块嵌入到二进制文件中执行
3. 高可靠性 - 核心框架代码设计优秀，模块设计简单
4. 低内存消耗 - 维护的连接对象占内存极小
5. 单机支持 100K 以上并发连接
6. 热部署 - master 管理进程与 worker 工作进程的分离设计
7. 最自由的 BSD 许可协议

## Nginx 所需软件

- GCC 编译器
- PCRE (Perl Compatible Regular Expression) 用于解析正则表达式
- zlib 对 HTTP 包的内容进行 gzip 格式的压缩，以减少网络传输量
- OpenSSL 用于支持 HTTPS

## Linux 内核参数优化

可以使得 Nginx 可以拥有更高的性能。比如：

- `file-max` - 进程可以同时打开的最大句柄数
- `tcp_tw_reuse` - 允许将 `TIME-WAIT` 状态的 socket 重新用于新的 TCP 连接
- ...

以及一些与网络协议栈相关的参数。

## 编译安装

除了少量核心代码，Nginx 完全由各个功能模块组成：

- 事件模块
- 默认编译进 Nginx 中的 HTTP 模块
- 默认不编译进 Nginx 中的 HTTP 模块
- 邮件代理服务器相关模块
- 其它模块

## 命令行控制

- 启动
- 配置文件路径、启动目录
- 停止服务 (立刻停止 / 正常处理完当前所有请求再停止)
- 重读配置文件 (reload)
- 平滑升级 Nginx (同时启动新旧 master 进程，以 _优雅_ 的方式关闭旧版本 Nginx)
