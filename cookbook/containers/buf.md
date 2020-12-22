# 缓冲区

<!-- TOC -->

- [缓冲区](#缓冲区)
    - [概述](#概述)
    - [函数](#函数)
        - [ngx_create_temp_buf](#ngx_create_temp_buf)
        - [ngx_calloc_buf](#ngx_calloc_buf)
        - [ngx_alloc_buf](#ngx_alloc_buf)
        - [ngx_buf_in_memory](#ngx_buf_in_memory)
        - [ngx_buf_in_memory_only](#ngx_buf_in_memory_only)
        - [ngx_buf_special](#ngx_buf_special)
        - [ngx_buf_sync_only](#ngx_buf_sync_only)
        - [ngx_buf_size](#ngx_buf_size)
        - [ngx_alloc_chain_link](#ngx_alloc_chain_link)
        - [ngx_create_chain_of_bufs](#ngx_create_chain_of_bufs)
        - [ngx_chain_add_copy](#ngx_chain_add_copy)
        - [ngx_chain_get_free_buf](#ngx_chain_get_free_buf)
        - [ngx_chain_update_chains](#ngx_chain_update_chains)
        - [ngx_chain_coalesce_file](#ngx_chain_coalesce_file)
        - [ngx_chain_update_sent](#ngx_chain_update_sent)
    - [原理](#原理)
        - [结构体](#结构体)
        - [内存布局](#内存布局)
        - [缓冲区位置标识](#缓冲区位置标识)

<!-- /TOC -->

## 概述

Nginx 实现了缓冲区，结构体为 `ngx_buf_t`，通常用于落盘前的缓冲、响应给客户端前的缓冲。

源代码位置：

- src/core/ngx_buf.h
- src/core/ngx_buf.c

和缓冲结构 `ngx_buf_t` 紧密相关的是 `ngx_chain_t` 结构，

缓冲区 `ngx_buf_t` 相关函数：

函数 | 描述
-|-
[ngx_create_temp_buf](#ngx_create_temp_buf) | 创建临时缓冲区，该缓冲区位于内存中。
[ngx_alloc_buf](#ngx_alloc_buf) | 创建新的链表。
[ngx_calloc_buf](#ngx_calloc_buf) | 创建新的链表。
[ngx_buf_in_memory](#ngx_buf_in_memory) | 判断 buf 的数据是否在内存中。
[ngx_buf_in_memory_only](#ngx_buf_in_memory_only) | 判断缓冲区的数据是否仅在内存中，而没有在文件里。
[ngx_buf_special](#ngx_buf_special) | 判断是否为特殊缓冲区。特殊缓冲区是指没有真正数据，仅有标识位的 buf。
[ngx_buf_sync_only](#ngx_buf_sync_only) | 判断是否为仅含 sync 标识位而没有其他数据。
[ngx_buf_size](#ngx_buf_size) | 获得缓冲区大小。需要注意，这是 buf 的 last 和 pos 之间部分的大小。

缓冲区链 `ngx_chain_t` 相关函数：

函数声明 | 描述
-|-
[ngx_alloc_chain_link](#ngx_alloc_chain_link) | 创建一个 `ngx_chain_t` 结构体。
[ngx_create_chain_of_bufs](#ngx_create_chain_of_bufs) | 创建多个临时缓冲区，并且将这些缓冲区用 chain 串联起来。
[ngx_chain_add_copy](#ngx_chain_add_copy) | 将一个 chain 拷贝到另一个 chain 的末尾。缓冲区本身不会进行拷贝，只是拷贝 chain 结构。
[ngx_chain_get_free_buf](#ngx_chain_get_free_buf) | 待补充。
[ngx_chain_update_chains](#ngx_chain_update_chains) | 待补充。
[ngx_chain_coalesce_file](#ngx_chain_coalesce_file) | 待补充。
[ngx_chain_update_sent](#ngx_chain_update_sent) | 根据要发送的字节数更新 chain 及其缓冲区。

## 函数

### ngx_create_temp_buf

创建临时缓冲区，包括 `ngx_buf_t` 元数据结构初始化以及对应的数据区域内存分配。

```c
ngx_buf_t *ngx_create_temp_buf(ngx_pool_t *pool, size_t size)
```

输入参数 | 描述
-|-
pool | 用于分配内存的内存池。
size | 创建数据区域大小为 size 的缓冲区。数据区域位于用户态内存中。

返回值 | 描述
-|-
非 NULL | 缓冲区指针。
NULL | 创建失败。

示例：

```c
// 使用 buffer 返回客户端响应
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_http_discard_request_body(r);

    ngx_str_t response = ngx_string("{\"result\": 0}");

    // 组装头部
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = response.len;
    ngx_str_set(&r->headers_out.content_type, "application/json");

    // 发送头部
    ngx_int_t rc = ngx_http_send_header(r);
    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }

    // 组装 body
    ngx_buf_t *b = ngx_create_temp_buf(r->pool, response.len);
    ngx_memcpy(b->pos, response.data, response.len);
    b->last = b->pos + response.len;
    b->last_buf = 1;
    b->last_in_chain = 1;

    ngx_chain_t out;
    out.buf = b;
    out.next = NULL;

    // 发送响应
    return ngx_http_output_filter(r, &out);
}
```

### ngx_calloc_buf

`ngx_calloc_buf` 是一个宏，用于分配 `ngx_buf_t` 的空间，并且初始化数据为 0。

```c
#define ngx_calloc_buf(pool) ngx_pcalloc(pool, sizeof(ngx_buf_t))
```

**注意：**

- 并不会分配缓冲区的数据空间，只会分配缓冲区的结构体元数据。

示例:

```c
static ngx_int_t
ngx_http_bar_content_handler(ngx_http_request_t *r)
{
    ngx_int_t     rc;
    ngx_buf_t    *b;
    ngx_chain_t   out;

    /* send header */

    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = 3;

    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }

    /* send body */

    b = ngx_calloc_buf(r->pool);
    if (b == NULL) {
        return NGX_ERROR;
    }

    b->last_buf = (r == r->main) ? 1: 0;
    b->last_in_chain = 1;

    b->memory = 1;

    b->pos = (u_char *) "foo";
    b->last = b->pos + 3;

    out.buf = b;
    out.next = NULL;

    return ngx_http_output_filter(r, &out);
}
```

### ngx_alloc_buf

`ngx_alloc_buf` 是一个宏，用于分配 `ngx_buf_t` 的空间。

**注意：**

- 并不会分配缓冲区的数据空间，只会分配缓冲区的结构体元数据。

```c
#define ngx_alloc_buf(pool)  ngx_palloc(pool, sizeof(ngx_buf_t))
```

示例和 [ngx_calloc_buf](#ngx_calloc_buf) 类似。

### ngx_buf_in_memory

判断 buf 的数据是否在内存中。这是一个宏：

```c
#define ngx_buf_in_memory(b)       ((b)->temporary || (b)->memory || (b)->mmap)
```

### ngx_buf_in_memory_only

判断 buf 的数据是否仅在内存中，不在文件中。这是一个宏：

```c
#define ngx_buf_in_memory_only(b)  (ngx_buf_in_memory(b) && !(b)->in_file)
```

### ngx_buf_special

判断是否为特殊 buf。特殊 buf 是指没有真正数据，仅有标识位的 buf。这是一个宏：

```c
#define ngx_buf_special(b)                                                   \
    (((b)->flush || (b)->last_buf || (b)->sync)                              \
     && !ngx_buf_in_memory(b) && !(b)->in_file)
```

**注意：**

- 特殊标识是指 `flush`, `last_buf` 和 `sync`
- 如果返回 true，则一定没有数据。

### ngx_buf_sync_only

判断是否仅包含 sync 特殊标识。如果返回 true，一定没有其他特殊标识和数据。这是一个宏：

```c
#define ngx_buf_sync_only(b)                                                 \
    ((b)->sync && !ngx_buf_in_memory(b)                                      \
     && !(b)->in_file && !(b)->flush && !(b)->last_buf)
```

### ngx_buf_size

获得缓冲区大小。需要注意，这个缓冲区大小指的是 last 和 pos 之间部分的大小。这是一个宏：

```c
#define ngx_buf_size(b)                                                      \
    (ngx_buf_in_memory(b) ? (off_t) ((b)->last - (b)->pos):                  \
                            ((b)->file_last - (b)->file_pos))
```

**注意：**

- 对于文件是 file_last 和 file_pos 之间的部分。

### ngx_alloc_chain_link

创建一个 `ngx_chain_t` 结构体。

函数声明：

```c
ngx_chain_t * ngx_alloc_chain_link(ngx_pool_t *pool)
```

输入参数 | 描述
-|-
pool | 用于分配内存的内存池。

返回值 | 描述
-|-
非 NULL | chain 指针。
NULL | 内存分配失败。

### ngx_create_chain_of_bufs

创建多个临时缓冲区，并且将这些缓冲区用 chain 串联起来。

函数声明：

```c
ngx_chain_t *ngx_create_chain_of_bufs(ngx_pool_t *pool, ngx_bufs_t *bufs)
```

输入参数 | 描述
-|-
pool | 用于分配内存的内存池。
bufs | 告诉 Nginx 需要多少缓冲区（`bufs->num`），每个缓冲区多大（`bufs->size`）。

返回值 | 描述
-|-
非 NULL | chain 指针。
NULL | 创建失败。

**注意：**

- 创建 num 个缓冲区，每个缓冲区大小为 size。
- 所有缓冲区使用的内存空间其实是连续的内存空间。

### ngx_chain_add_copy

将一个 chain 拷贝到另一个 chain 的末尾。缓冲区本身不会进行拷贝，只是拷贝 chain 结构。

函数声明：

```c
ngx_int_t ngx_chain_add_copy(ngx_pool_t *pool, ngx_chain_t **chain, ngx_chain_t *in)
```

输入参数 | 描述
-|-
pool | 用于分配内存的内存池。
chain | 将 in 进行拷贝，放置在 chain 的末尾。
in | 将 in 进行拷贝，放置在 chain 的末尾。

返回值 | 描述
-|-
NGX_OK | 操作成功。
NGX_ERRORs | 操作失败。

### ngx_chain_get_free_buf

待补充。

### ngx_chain_update_chains

待补充。

### ngx_chain_coalesce_file

待补充。

### ngx_chain_update_sent

根据要发送的字节数更新 chain 及其缓冲区。

函数声明：

```c
ngx_chain_t * ngx_chain_update_sent(ngx_chain_t *in, off_t sent)
```

输入参数 | 描述
-|-
in | 需要进行更新的 chain。
sent | 发送了 sent 字节的数据。

返回值 | 描述
-|-
非 NULL | 需要接下来继续进行处理的 buffer chain。
NULL | chain 中的所有 buffer 的数据都已经发送完毕。

## 原理

### 结构体

```c
typedef void *          ngx_buf_tag_t;
typedef ngx_bufs_s      ngx_bufs_t;
typedef ngx_buf_s       ngx_buf_t;
typedef ngx_chain_s     ngx_chain_t;

struct ngx_buf_s {
    u_char          *pos;                   // 指向 buf 需要进行处理的内存首地址
    u_char          *last;                  // 指向 buf 的有效内容末尾地址

    off_t            file_pos;              // 和 pos 含义相同，代表的是文件的处理位置
    off_t            file_last;             // 和 last 含义相同，代表的是未见有效内容末尾地址

    u_char          *start;                 // buf 的内存起始地址
    u_char          *end;                   // buf 的内存末尾地址

    ngx_buf_tag_t    tag;                   // 用于区分缓冲区的值，由使用者来决定其含义。ngx_buf_tag_t 其实就是 void* 类型。
    ngx_file_t      *file;                  // 当 buf 的内容位于文件中时，该字段指向文件对象。

    unsigned         temporary:1;           // 表示 buf 的内容位于可变的内存中，buf 的数据可以修改
    unsigned         memory:1;              // 表示 buf 的内容位于只读的内存中，buf 的数据不可以修改
    unsigned         mmap:1;                // 表示 buf 的内容是 mmap 映射到的内存中，buf 的数据是不可以修改的
    unsigned         in_file:1;             // 表示 buf 所包含的内容位于文件中

    unsigned         flush:1;               // 表示 buf 需要执行 flush 操作
    unsigned         sync:1;                // 表示 buf 不携带任何数据和特殊标识。默认情况下，Nginx 认为该类缓冲区（不带数据和标识）是个异常，但是此标识别告诉 Nginx 跳过异常。

    unsigned         last_buf:1;            // 表示 buf 位于 chain 中的最后一个。
    unsigned         last_in_chain:1;       // 表示 buf 位于 chain 中的最后一个。

    ngx_buf_t       *shadow;                // 指向另外一个 buffer（shadow），表示该 buf 的数据来自指向的 shadow buffer
    unsigned         last_shadow:1;         // 使用 shadow buffer 时，最新创建的 last_shadow 为 1
    unsigned         recycled:1;            // 表示 buf 可以释放，通常配合 shadow 字段使用

    unsigned         temp_file:1;           // 表示 buf 是一个临时文件

    /* STUB */ int   num;
};


/*
chain 用于将多个 buffer 进行串联。
*/
struct ngx_chain_s {
    ngx_buf_t    *buf;                      // 指向缓冲区
    ngx_chain_t  *next;                     // 指向下一个 chain
};


/*
描述需要多少个 buffer，以及每个 buffer 的大小，用于快速创建 chain。
*/
struct ngx_bufs_s {
    ngx_int_t    num;
    size_t       size;
};
```

### 内存布局

![](/resource/ngx_buf_t.png)

### 缓冲区位置标识

在 `ngx_buf_t` 中，存在四个非常重要的，用于指示位置的指针：

- `buffer->start`
- `buffer->end`
- `buffer->pos`
- `buffer->last`

通过 `buffer->start` 和 `buffer->end` 为缓冲区所使用的内存空间圈出了范围，所有的缓冲区数据活动不得超过该范围。

缓冲区中的数据通常是来自于其他地方写入，由 `buffer->last` 表明写入的最后位置，这样 `buffer->last` 前的内存都是写入过数据且有效的。

缓冲区中的数据在写入后，会被其他地方消费和处理，由 `buffer->pos` 表明还未处理的位置，这样 `buffer->pos` 和 `buffer->last` 就代表写入到缓冲区中，但是还未进行处理部分的空间。

典型的例子是读取请求头并解析请求头：

1. 初始化缓冲区内存空间，由 `start` 和 `end` 限制了缓冲区的大小。
1. 读取 socket 中的数据，将 `last` 后移。一次读取可能并不能将所有的数据读取完。
1. 读取的数据进行解析处理，将 `pos` 后移。一次只处理一行，而一次读取可能读取了多行。
1. 如果还有其余行，回到第三步继续进行解析，`pos` 继续后移。
1. 如果还有数据没有读取完，则回到第二步继续读取剩余数据，`last` 继续后移。
1. 如果数据量太大，超过了限制，会返回失败，关闭连。
