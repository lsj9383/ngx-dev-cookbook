# 链表

<!-- TOC -->

- [链表](#链表)
    - [概述](#概述)
    - [函数](#函数)
        - [ngx_list_create](#ngx_list_create)
        - [ngx_list_init](#ngx_list_init)
        - [ngx_list_push](#ngx_list_push)
    - [原理](#原理)
        - [结构体](#结构体)
        - [内存布局](#内存布局)
        - [分配原理](#分配原理)

<!-- /TOC -->

## 概述

Nginx 实现了链表，结构体为 `ngx_list_t`。Nginx 的链表是一个特殊的链表，其中每个链表节点都是一个数组，而链表的元素实质上是放在数组中。

源代码位置：

- src/core/ngx_list.h
- src/core/ngx_list.c

函数声明 | 描述
-|-
[ngx_list_create](#ngx_array_create) | 创建新的链表。
[ngx_list_init](#ngx_array_init) | 初始化链表（如果调用了 ngx_list_create 则已经初始化）。
[ngx_list_push](#ngx_array_destroy) | 向链表中 Push 一个元素。

**注意**

- 因为链表的元素实际存放在数组中，因此没有提供删除每个元素的方法。
- 链表不允许释放，只能通过内存池的销毁来释放掉链表。

## 函数

### ngx_list_create

创建新的链表，会初始化链表中的第一个节点的数组空间。

声明：

```c
ngx_list_t * ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size)
```

输入参数 | 描述
-|-
pool | 用于分配内存空间的内存池。
n | 每个数组节点中预分配的元素个数。
size | 每个元素的大小。

返回值 | 描述
-|-
非 NULL | 链表对象指针。
NULL | 链表对象分配失败。

示例：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_pool_t                  *pool;

    pool = ngx_create_pool(NGX_DEFAULT_POOL_SIZE, r->connection->log);
    if (pool == NULL) {
        return NGX_ERROR;
    }

    // 分配链表，链表中的元素是 ngx_str_t，每次会预分配大小为 10 的数组空间
    ngx_list_t* name_list = ngx_list_create(pool, 10, sizeof(ngx_str_t))

    ngx_destroy_pool(pool);
    return NGX_DECLINED;
}
```

### ngx_list_init

初始化链表，这要求 `ngx_list_t` 内存已经存在。通常是一些全局遍变量中的链表初始化。

```c
ngx_int_t ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size)
```

输入参数 | 描述
-|-
list | 已经分配空间的 `ngx_list_t` 对象。
pool | 用于分配内存空间的内存池。
n | 链表节点中的数组预分配元素个数。
size | 每个元素的大小。

返回值 | 描述
-|-
NGX_OK | 初始化成功。
NGX_ERROR | 初始化失败。

```c
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    ngx_pool_t                  *pool;
    log = old_cycle->log;
    pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);

    // cycle 是 Nginx 全局变量
    if (ngx_list_init(&cycle->open_files, pool, n, sizeof(ngx_open_file_t))
        != NGX_OK)
    {
        ngx_destroy_pool(pool);
        return NULL;
    }
}
```

**注意：**

- 初始化元数据其实就是给 `ngx_list_t` 的相应字段初始化值。
- 初始化首个节点的数组空间，即 `ngx_list_t.parts.elts` 指向的内存空间，以及 parts 的其他字段。

### ngx_list_push

向链表中 Push 一个元素，实质上是从预分配空间中返回一个未使用的。

声明：

```c
void *ngx_list_push(ngx_list_t *list);
```

输入参数 | 描述
-|-
list | 链表对象的指针。

返回值 | 描述
-|-
非 NULL | 分配成功，返回分配的元素指针。
NULL | 分配失败。

示例：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_pool_t                  *pool;
    ngx_str_t                   *name;
    ngx_int_t                   i;

    pool = ngx_create_pool(NGX_DEFAULT_POOL_SIZE, r->connection->log);
    if (pool == NULL) {
        return NGX_ERROR;
    }

    ngx_list_t* name_list = ngx_list_create(pool, 2, sizeof(ngx_str_t))

    name = ngx_list_push(name_list);  ngx_str_set(name, "test name 1");
    name = ngx_list_push(name_list);  ngx_str_set(name, "test name 2");
    name = ngx_list_push(name_list);  ngx_str_set(name, "test name 3");
    name = ngx_list_push(name_list);  ngx_str_set(name, "test name 4");
    name = ngx_list_push(name_list);  ngx_str_set(name, "test name 5");

    // 遍历链表
    ngx_list_part_s* part = &name_list.part;
    ngx_str_s* names = data = part->elts;
    for (i = 0 ;; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }

            part = part->next;
            data = part->elts;
            i = 0;
        }

        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "name_array[%ui]: %V", i, names[i]);
    }

    ngx_destroy_pool(pool);
    return NGX_DECLINED;
}
```

## 原理

### 结构体

```c
typedef struct ngx_list_part_s  ngx_list_part_t;        // 链表节点
typedef struct ngx_list_s       ngx_list_t

struct ngx_list_s {
    ngx_list_part_t  *last;         // 链表的最后一个元素
    ngx_list_part_t   part;         // 链表的第一个节点
    size_t            size;         // 每个元素的大小
    ngx_uint_t        nalloc;       // 每个节点中数组可以存储的元素个数
    ngx_pool_t       *pool;         // 分配内存池
};

struct ngx_list_part_s {
    void             *elts;         // 节点中的数组，是一个连续内存空间的首地址，大小为 size * nalloc
    ngx_uint_t        nelts;        // 数组中已经使用的元素个数
    ngx_list_part_t  *next;         // 下一个节点
};
```

### 内存布局

![](/resource/ngx_list_t.png)

### 分配原理

链表中添加一个元素，其实就是从预分配的空间中返回一个未使用的元素，当预分配元素使用完毕，则会再次申请预分配空间。每个预分配空间就是一个数组，通过链表的方式将多个预分配空间串联起来。

正因为链表会通过新增节点的方式预分配空间，避免反复的申请内存，所以链表的插入效率很高。
