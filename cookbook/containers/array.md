# 数组

<!-- TOC -->

- [数组](#数组)
    - [概述](#概述)
    - [函数](#函数)
        - [ngx_array_create](#ngx_array_create)
        - [ngx_array_init](#ngx_array_init)
        - [ngx_array_destroy](#ngx_array_destroy)
        - [ngx_array_push](#ngx_array_push)
        - [ngx_array_push_n](#ngx_array_push_n)
    - [原理](#原理)
        - [结构体](#结构体)
        - [内存布局](#内存布局)
        - [动态分配](#动态分配)
        - [回收优化](#回收优化)

<!-- /TOC -->

## 概述

Nginx 实现了动态数组，结构体为 `ngx_array_t`。数组源代码位置：

- src/core/ngx_array.h
- src/core/ngx_array.c

函数声明 | 描述
-|-
[ngx_array_create](#ngx_array_create) | 创建新的数组。
[ngx_array_init](#ngx_array_init) | 初始化数组（如果调用了 ngx_array_create 则已经初始化）。
[ngx_array_destroy](#ngx_array_destroy) | 释放数组空间。
[ngx_array_push](#ngx_array_push) | 从数组中分配一个元素。
[ngx_array_push_n](#ngx_array_push_n) | 从数组中分配 n 个元素。

## 函数

### ngx_array_create

创建新的数组。

声明：

```c
ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);
```

输入参数 | 描述
-|-
p | 用于分配内存空间的内存池。
n | 预分配的元素个数。
size | 每个元素的大小。

返回值 | 描述
-|-
非 NULL | 数组对象指针。
NULL | 数组分配失败。

示例：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_pool_t                  *pool;

    pool = ngx_create_pool(NGX_DEFAULT_POOL_SIZE, r->connection->log);
    if (pool == NULL) {
        return NGX_ERROR;
    }

    ngx_array_t *names = ngx_array_create(p, 3, sizeof(ngx_str_t));

    ngx_destroy_pool(pool);
    return NGX_DECLINED;
}
```

### ngx_array_init

初始化数组存储空间和元数据，这要求 `ngx_array_t` 内存已经存在。通常是一些全局遍变量中的数组初始化。

```c
ngx_int_t ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size)
```

输入参数 | 描述
-|-
array | 已经分配空间的 `ngx_array_t` 对象。
pool | 用于分配内存空间的内存池。
n | 预分配的元素个数。
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
    if (ngx_array_init(&cycle->paths, pool, n, sizeof(ngx_path_t *))
        != NGX_OK)
    {
        ngx_destroy_pool(pool);
        return NULL;
    }
}
```

**注意：**

- 初始化数组存储空间其实就是初始化 `ngx_array_t.elts` 指向的内存空间。
- 初始化元数据其实就是给 `ngx_array_t` 的相应字段初始化值。

### ngx_array_destroy

销毁数组空间。

声明：

```c
void ngx_array_destroy(ngx_array_t *a);
```

输入参数 | 描述
-|-
a | 数组指针。

示例：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_pool_t                  *pool;

    pool = ngx_create_pool(NGX_DEFAULT_POOL_SIZE, r->connection->log);
    if (pool == NULL) {
        return NGX_ERROR;
    }

    ngx_array_t *names = ngx_array_create(pool, 3, sizeof(ngx_str_t));
    ngx_array_destroy(names);

    ngx_destroy_pool(pool);
    return NGX_DECLINED;
}
```

### ngx_array_push

向数组中 Push 一个元素。函数会返回一个新的元素内存空间，其实本质就是从数组中申请一个元素。

声明：

```c
void ngx_array_push(ngx_array_t *a);
```

输入参数 | 描述
-|-
a | 数组指针。

示例：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_pool_t                  *pool;
    ngx_str_t                   *name;
    pool = ngx_create_pool(NGX_DEFAULT_POOL_SIZE, r->connection->log);
    if (pool == NULL) {
        return NGX_ERROR;
    }

    ngx_array_t *name_array = ngx_array_create(pool, 3, sizeof(ngx_str_t));
    name = ngx_array_push(name_array); ngx_str_set(name, "test 1");
    name = ngx_array_push(name_array); ngx_str_set(name, "test 2");
    name = ngx_array_push(name_array); ngx_str_set(name, "test 3");
    name = ngx_array_push(name_array); ngx_str_set(name, "test 4");

    ngx_uint_t i = 0;
    for (i = 0; i < name_array->nelts; ++i) {
        name = ((ngx_str_t *) name_array->elts) + i;
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "name_array[%ui]: %V", i, name);
    }

    ngx_array_destroy(name_array);
    ngx_destroy_pool(pool);
    return NGX_DECLINED;
}
```

**注意：**

- 动态分配时，每次都将内存给翻倍。

### ngx_array_push_n

向数组中 Push N 个元素。函数会返回 N 个新的元素连续内存空间，其实本质就是从数组中申请 N 个元素。

声明：

```c
void ngx_array_push_t(ngx_array_t *a);
```

输入参数 | 描述
-|-
a | 数组指针。
n | 申请的元素个数。

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_pool_t                  *pool;

    pool = ngx_create_pool(NGX_DEFAULT_POOL_SIZE, r->connection->log);
    if (pool == NULL) {
        return NGX_ERROR;
    }

    ngx_array_t *name_array = ngx_array_create(pool, 3, sizeof(ngx_str_t));

    ngx_str_t *names = ngx_array_push_n(name_array, 4);
    ngx_str_set(&names[0], "test 1");
    ngx_str_set(&names[1], "test 2");
    ngx_str_set(&names[2], "test 3");
    ngx_str_set(&names[3], "test 4");

    ngx_uint_t i = 0;
    for (i = 0; i < name_array->nelts; ++i) {
        ngx_str_t *name = ((ngx_str_t *) name_array->elts) + i;
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "name[%ui]: %V", i, name);
    }

    ngx_array_destroy(name_array);
    ngx_destroy_pool(pool);
    return NGX_DECLINED;
}
```

**注意：**

- 动态分配时，不再是直接将内存空间翻倍，而是根据 push N 的具体值来进行判断扩展的内存空间。

## 原理

### 结构体

```c
typedef struct {
    void        *elts;          // 数组实际存储数据的空间，总大小为 nalloc * size，已使用 nelts * size
    ngx_uint_t   nelts;         // 数组中实际占用的元素个数
    size_t       size;          // 每个数组元素的大小为 size(Byte)
    ngx_uint_t   nalloc;        // 数组可容纳的最大元素个数（数组预分配的元素个数），可动态扩展
    ngx_pool_t  *pool;          // 用于进行数组内存分配的内存池
} ngx_array_t;
```

### 内存布局

![](/resource/ngx_array_t.png)

### 动态分配

C 语言中的数组结构长度是固定的，使用Nginx 的 `ngx_array_t` 结构，可以得到一个长度不固定的数组，这是依赖动态分配实现的。

`ngx_array_t` 会预先分配一块连续内存空间，当新增元素时，会从连续内存空间中分配元素，若内存空间已经耗尽，会申请额外的内存空间。

动态分配这里有两种情况的处理：

- 当数组是内存池中最后申请的空间，并且扩展后的数组长度仍然没有超过内存池，则直接从内存池中分配连续的空间。
- 如果不满足上述条件，则从内存池中分配两倍原长度的数组空间，并且将原理的数组值复制到新数组空间。需要注意，原数组并不会被释放。

```c
void *
ngx_array_push(ngx_array_t *a)
{
    void        *elt, *new;
    size_t       size;
    ngx_pool_t  *p;

    if (a->nelts == a->nalloc) {

        /* the array is full */

        size = a->size * a->nalloc;

        p = a->pool;

        /*
        如果数组空间位于内存池的最后，并且直接顺序扩充数组也没有超过内存池的空间， 则数组直接将内存池后面的空间占为己有，
        否则需要分配更大的内存空间，并将原来的数组数据复制到新的内存空间。
        */
        if ((u_char *) a->elts + size == p->d.last
            && p->d.last + a->size <= p->d.end)
        {
            /*
             * the array allocation is the last in the pool
             * and there is space for new allocation
             */

            p->d.last += a->size;
            a->nalloc++;

        } else {
            /* allocate a new array */

            new = ngx_palloc(p, 2 * size);
            if (new == NULL) {
                return NULL;
            }

            ngx_memcpy(new, a->elts, size);
            a->elts = new;
            a->nalloc *= 2;
        }
    }

    elt = (u_char *) a->elts + a->size * a->nelts;
    a->nelts++;

    return elt;
}
```

### 回收优化

相比于直接分配一个内存空间当作数组使用（`ngx_str_t *names = ngx_palloc(p, n * sizeof(ngx_str_t))`）而言，用 ngx_array_t 的优势之一就是可以在 ngx_array_t 进行回收时，有一定可能回收内存池的部分。

在满足以下条件时，会将内存池中的空间进行释放：

- 数组是内存池最后分配的元素。

实现原理如下：

```c
void
ngx_array_destroy(ngx_array_t *a)
{
    ngx_pool_t  *p;

    p = a->pool;

    /*
    如果刚好数组位于内存池的最后分配的部分，则将内存池最后部分内存重新交给内存池
    注意：
    - 这仅对第一块内存池有效，有可能内存池已经在使用后面的内存池了
    - 适用于分配并在使用完成立即回收的场景
    */
    if ((u_char *) a->elts + a->size * a->nalloc == p->d.last) {
        p->d.last -= a->size * a->nalloc;
    }

    /*
    和上面的逻辑是一样的，但是回收的是元数据部分
    */
    if ((u_char *) a + sizeof(ngx_array_t) == p->d.last) {
        p->d.last = (u_char *) a;
    }
}
```

**注意：**

- 正如注释提到的，只有当使用第一块内存池时这样的回收操作才有意义。
