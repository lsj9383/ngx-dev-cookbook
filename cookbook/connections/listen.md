# 监听连接

## 概述

在 HTTP 模块、Stream 模块、Mail 模块的配置文件解析时，如果发现了需要监听的端口，会调用 `ngx_create_listening` 创建监听 socket。

**注意：**

- 这里创建的监听连接只是在 `cycle->listening` 进行记录，并不会实际创建 socket。
- event_module 的配置文件初始化完毕的函数中，依据 `cycle->listening` 的 reuseport 情况，判断是否进行 clone，如果为 reuseport，则会给每个进程都克隆一个独立的监听 socket（监听相同的端口和地址）。

实际的监听 socket 在配置文件解析完毕后，于 master 进程中调用 `ngx_open_listening_sockets` 函数，统一进行 socket 创建。

在 event_core_module 的 worker 进程初始化函数中，会将 master 进程初始化得到的监听 socket 分配给各个进程，方便各个进程进行监听。

## 结构体

```c
struct ngx_listening_s {
    ngx_socket_t        fd;

    struct sockaddr    *sockaddr;
    socklen_t           socklen;    /* size of sockaddr */
    size_t              addr_text_max_len;
    ngx_str_t           addr_text;

    int                 type;

    int                 backlog;
    int                 rcvbuf;
    int                 sndbuf;

#if (NGX_HAVE_KEEPALIVE_TUNABLE)
    int                 keepidle;
    int                 keepintvl;
    int                 keepcnt;
#endif

    /* handler of accepted connection */
    ngx_connection_handler_pt   handler;

    void               *servers;  /* array of ngx_http_in_addr_t, for example */

    ngx_log_t           log;
    ngx_log_t          *logp;

    size_t              pool_size;
    /* should be here because of the AcceptEx() preread */
    size_t              post_accept_buffer_size;
    /* should be here because of the deferred accept */
    ngx_msec_t          post_accept_timeout;

    ngx_listening_t    *previous;
    ngx_connection_t   *connection;

    ngx_rbtree_t        rbtree;
    ngx_rbtree_node_t   sentinel;

    ngx_uint_t          worker;

    unsigned            open:1;         // 表示监听的套接字有效，在 ngx_init_cycle 时不关闭监听端口
    unsigned            remain:1;       // 
    unsigned            ignore:1;

    unsigned            bound:1;       /* already bound */
    unsigned            inherited:1;   /* inherited from previous process */
    unsigned            nonblocking_accept:1;
    unsigned            listen:1;
    unsigned            nonblocking:1;
    unsigned            shared:1;    /* shared between threads or processes */
    unsigned            addr_ntop:1;
    unsigned            wildcard:1;

#if (NGX_HAVE_INET6)
    unsigned            ipv6only:1;
#endif
    unsigned            reuseport:1;
    unsigned            add_reuseport:1;
    unsigned            keepalive:2;

    unsigned            deferred_accept:1;
    unsigned            delete_deferred:1;
    unsigned            add_deferred:1;
#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
    char               *accept_filter;
#endif
#if (NGX_HAVE_SETFIB)
    int                 setfib;
#endif

#if (NGX_HAVE_TCP_FASTOPEN)
    int                 fastopen;
#endif

};
```
