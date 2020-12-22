# 哈希表

<!-- TOC -->

- [哈希表](#哈希表)
    - [概述](#概述)
    - [函数](#函数)
        - [ngx_hash_key](#ngx_hash_key)
        - [ngx_hash_key_lc](#ngx_hash_key_lc)
        - [ngx_hash_strlow](#ngx_hash_strlow)
        - [ngx_hash_init](#ngx_hash_init)
        - [ngx_hash_find](#ngx_hash_find)
        - [ngx_hash_find_wc_head](#ngx_hash_find_wc_head)
        - [ngx_hash_find_wc_tail](#ngx_hash_find_wc_tail)
        - [ngx_hash_find_combined](#ngx_hash_find_combined)
        - [ngx_hash_wildcard_init](#ngx_hash_wildcard_init)
        - [ngx_hash_keys_array_init](#ngx_hash_keys_array_init)
        - [ngx_hash_add_key](#ngx_hash_add_key)
    - [原理](#原理)
        - [数据结构](#数据结构)
        - [内存布局](#内存布局)
        - [数据存储](#数据存储)
        - [通配符检索](#通配符检索)

<!-- /TOC -->

## 概述

Nginx 提供了：

- 哈希表结构 `ngx_hash_t`，用于快速检索关键词
- 哈希表结构 `ngx_hash_whildcard_t`，用于检索通配关键词。
- 哈希表结构 `ngx_hash_combined_t`，用于检索关键词和通配关键词。

**注意：**

- `ngx_hash_t` 是只读的，在初始化时确定数据后，后续只能进行查询。
- 如果希望支持读写的字典，需要借助[红黑树](rbtree.md)。

哈希值计算相关函数：

函数 | 描述
-|-
[ngx_hash_key](#ngx_hash_key) | 计算关键词的 Hash 值。
[ngx_hash_key_lc](#ngx_hash_key_lc) | 将关键词转换为小写后计算 Hash 值。
[ngx_hash_strlow](#ngx_hash_strlow) | 将关键词转化为小写后计算 Hash 值，并返回小写的字符串。

哈希表相关函数

函数 | 描述
-|-
[ngx_hash_init](#ngx_hash_init) | 哈希表初始化。
[ngx_hash_find](#ngx_hash_find) | 从哈希表中检索关键词。

支持通配的哈希表相关函数：

函数 | 描述
-|-
[ngx_hash_wildcard_init](#ngx_hash_wildcard_init) |
[ngx_hash_find_wc_head](#ngx_hash_find_wc_head) |
[ngx_hash_find_wc_tail](#ngx_hash_find_wc_tail) |
[ngx_hash_find_combined](#ngx_hash_find_combined) |
[ngx_hash_keys_array_init](#ngx_hash_keys_array_init) |
[ngx_hash_add_key](#ngx_hash_add_key) |

## 函数

### ngx_hash_key

计算 Key 的 Hash 值，采用 BKDR 算法。

```c
ngx_uint_t ngx_hash_key(u_char *data, size_t len);
```

输入参数 | 描述
-|-
data | Key 数据。
len | 数据长度。

返回值是 Hash 值。

示例：

```c
ngx_str_t key = ngx_string("name");
ngx_uint_t key_hash = ngx_hash_key(key.data, key.len);
```

### ngx_hash_key_lc

将关键词转换为小写后计算 Hash 值。

```c
ngx_uint_t ngx_hash_key_lc(u_char *data, size_t len);
```

输入参数 | 描述
-|-
data | Key 数据。
len | 数据长度。

返回值是 Hash 值。

示例：

```c
ngx_str_t key = ngx_string("NAME");
ngx_uint_t key_hash = ngx_hash_key_lc(key.data, key.len);
```

### ngx_hash_strlow

```c
ngx_uint_t ngx_hash_strlow(u_char *dst, u_char *src, size_t n);
```

输入参数 | 描述
-|-
dst | 存放 src 对应小写的的空间。
src | 原始字符串。
n | dst 和 src 的数据长度。

返回值是小写的 Hash 值。

示例：

```c
ngx_str_t key = ngx_string("NaMe");
char* dst = ngx_calloc(key.len, log);

ngx_uint_t key_hash = ngx_hash_strlow(dst, key.data, key.len);

ngx_free(dst, log);
```

### ngx_hash_init

哈希表初始化。

```c
ngx_int_t ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts);
```

输入参数 | 描述
-|-
hinit | 存放 src 对应小写的的空间。
names | keyvalue 对数组。
nelts | keyvalue 对的个数。

返回 NGX_OK 时成功。

**注意：**

- 如果 key 有大写，在写入到哈希表时会转换为小写。
- 需要对 hinit 数据进行初始化，主要包括：
  - 设置 `hinit->hash = NULL` （可以借助 ngx_memzero 进行所有数据赋 0）。
  - 设置 `hinit->max_size`，bucket 的最大个数。
  - 设置 `hint->bucket_size`，每个 bucket 的最大大小。
- 最终需要的 bucket 数量以及每个 bucket 的大小都需要经过 Nginx 平衡，但是都不会超过 `hinit` 中的限定值。
- 初始化完成后，哈希表为 `hinit->hash`。

示例：

```c
static ngx_int_t ngx_http_test_handler(ngx_http_request_t *r) {
    ngx_hash_key_t kvs[5];

    ngx_str_set(&kvs[0].key, "alibaba");
    kvs[0].key_hash = ngx_hash_key(kvs[0].key.data, kvs[0].key.len);
    kvs[0].value = "hello alibaba";

    ngx_str_set(&kvs[1].key, "baidu");
    kvs[1].key_hash = ngx_hash_key(kvs[1].key.data, kvs[1].key.len);
    kvs[1].value = "hello baidu";

    ngx_str_set(&kvs[2].key, "bytedance");
    kvs[2].key_hash = ngx_hash_key(kvs[2].key.data, kvs[2].key.len);
    kvs[2].value = "hello bytedance";

    ngx_str_set(&kvs[3].key, "Google");
    kvs[3].key_hash = ngx_hash_key(kvs[3].key.data, kvs[3].key.len);
    kvs[3].value = "hello Google";

    ngx_str_set(&kvs[4].key, "Tencent");
    kvs[4].key_hash = ngx_hash_key(kvs[4].key.data, kvs[4].key.len);
    kvs[4].value = "hello Tencent";

    ngx_hash_init_t hinit;
    hinit.hash = NULL;
    hinit.name = "test-hash-table";
    hinit.max_size = 128;
    hinit.bucket_size = 128;
    hinit.pool = r->pool;

    // 初始化
    ngx_int_t rc = ngx_hash_init(&hinit, kvs, sizeof(kvs) / szieof(ngx_hash_key_t));
    if (rc == NGX_ERROR) {
        return NGX_ERROR;
    }

    // 迭代每一个 bucket
    ngx_uint_t i = 0;
    for (i = 0; i < hinit.hash->size; ++i) {
        ngx_hash_elt_t* bucket = hinit.hash->buckets[i];

        // 迭代 bucket 中的每一个元素
        while(bucket->value) {
            ngx_str_t key;
            key.data = bucket->name;
            key.len = bucket->len;
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "========= key: %V, value: %s", &key, bucket->value);

            bucket = (ngx_hash_elt_t *) ngx_align_ptr(&bucket->name[0] + bucket->len, sizeof(void *));
        }
    }
}
```

### ngx_hash_find

从哈希表中检索关键词。

```c
void *ngx_hash_find(ngx_hash_t *hash, ngx_uint_t key, u_char *name, size_t len);
```

输入参数 | 描述
-|-
hash | 哈希表。
key | key 的哈希值。
name | key 的数据。
len | key 的数据长度。

返回非 NULL 时存在数据，否则没有对应 key 的 value 值。

**注意：**

- key 的计算可以通过函数：`ngx_hash_key`。

示例：

```c
static ngx_int_t ngx_http_test_handler(ngx_http_request_t *r) {
    // 初始化 kvs
    ngx_hash_key_t kvs[5];
    ngx_str_set(&kvs[0].key, "alibaba");
    kvs[0].key_hash = ngx_hash_key(kvs[0].key.data, kvs[0].key.len);
    kvs[0].value = "hello alibaba";
    // ...

    // 初始化 hinit
    ngx_hash_init_t hinit;
    hinit.hash = NULL;
    hinit.name = "test-hash-table";
    hinit.max_size = 128;
    hinit.bucket_size = 128;
    hinit.pool = r->pool;
    

    // 初始化
    ngx_int_t rc = ngx_hash_init(&hinit, kvs, 5);
    if (rc == NGX_ERROR) {
        return NGX_ERROR;
    }

    // 从 hash 表中查询
    ngx_str_t find_key = ngx_string("alibaba");
    char* res = ngx_hash_find(hinit.hash, ngx_hash_key(find_key.data, find_key.len), find_key.data, find_key.len);
    if (res != NULL) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "key: %V, value: %s", &find_key, res);
    } else {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "key: %V, not found", &find_key);
    }
}
```

### ngx_hash_find_wc_head

```c
void *ngx_hash_find_wc_head(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);
```

### ngx_hash_find_wc_tail

```c
void *ngx_hash_find_wc_tail(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);
```

### ngx_hash_find_combined

```c
void *ngx_hash_find_combined(ngx_hash_combined_t *hash, ngx_uint_t key, u_char *name, size_t len);
```

### ngx_hash_wildcard_init

```c
ngx_int_t ngx_hash_wildcard_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts);
```

### ngx_hash_keys_array_init

```c
ngx_int_t ngx_hash_keys_array_init(ngx_hash_keys_arrays_t *ha, ngx_uint_t type);
```

### ngx_hash_add_key

```c
ngx_int_t ngx_hash_add_key(ngx_hash_keys_arrays_t *ha, ngx_str_t *key, void *value, ngx_uint_t flags);
```

## 原理

### 数据结构

```c
typedef ngx_hash_elt_s              ngx_hash_elt_t;
typedef ngx_hash_s                  ngx_hash_t;
typedef ngx_hash_key_s              ngx_hash_key_t;
typedef ngx_hash_init_s             ngx_hash_init_t;
typedef ngx_hash_wildcard_s         ngx_hash_wildcard_t;
typedef ngx_hash_combined_s         ngx_hash_combined_t;
typedef ngx_hash_keys_arrays_s      ngx_hash_keys_arrays_t;
typedef ngx_table_elt_s             ngx_table_elt_t;


// 哈希表中的每个元素，这是一个 key-value 结构
struct {
    void             *value;        // 每个 kv 对的 value 指针
    u_short           len;          // k 的长度
    u_char            name[1];      // k 的指针
} ngx_hash_elt_s;

// 哈希表数据结构
struct {
    ngx_hash_elt_t  **buckets;      // bucket 数组，每个 bucket 中存放了 ngx_hash_elt_s 元素
    ngx_uint_t        size;         // bucket 的个数
} ngx_hash_s;

// 用于进行初始化的 hash 的 kv 对 k 是个字符串
struct {
    ngx_str_t         key;          // key
    ngx_uint_t        key_hash;     // key 对应的 hash 值
    void             *value;        // value
} ngx_hash_key_s;

// 通过 ngx_hash_init_s 对哈希表进行初始化
struct {
    ngx_hash_t       *hash;             // 初始化完成的哈希表就是该指针指向空间
    ngx_hash_key_pt   key;              // 计算 key 的 hash 值，用于通配符形式的哈希表

    ngx_uint_t        max_size;         // 设定最大的 bucket 数量
    ngx_uint_t        bucket_size;      // 设定每个 bucket 的大小

    char             *name;             // 哈希表的名称
    ngx_pool_t       *pool;             // 内存池，用于分配内存空间
    ngx_pool_t       *temp_pool;
} ngx_hash_init_s;

struct {
    ngx_hash_t        hash;
    void             *value;
} ngx_hash_wildcard_s;

struct {
    ngx_hash_t            hash;
    ngx_hash_wildcard_t  *wc_head;
    ngx_hash_wildcard_t  *wc_tail;
} ngx_hash_combined_s;

struct {
    ngx_uint_t        hsize;

    ngx_pool_t       *pool;
    ngx_pool_t       *temp_pool;

    ngx_array_t       keys;
    ngx_array_t      *keys_hash;

    ngx_array_t       dns_wc_head;
    ngx_array_t      *dns_wc_head_hash;

    ngx_array_t       dns_wc_tail;
    ngx_array_t      *dns_wc_tail_hash;
} ngx_hash_keys_arrays_s;


struct {
    ngx_uint_t        hash;
    ngx_str_t         key;
    ngx_str_t         value;
    u_char           *lowcase_key;
} ngx_table_elt_s;
```

### 内存布局

### 数据存储

### 通配符检索
