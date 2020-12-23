# 哈希表

<!-- TOC -->

- [哈希表](#哈希表)
    - [概述](#概述)
    - [函数](#函数)
        - [ngx_hash_key](#ngx_hash_key)
        - [ngx_hash_key_lc](#ngx_hash_key_lc)
        - [ngx_hash_strlow](#ngx_hash_strlow)
        - [ngx_hash_init](#ngx_hash_init)
        - [ngx_hash_keys_array_init](#ngx_hash_keys_array_init)
        - [ngx_hash_add_key](#ngx_hash_add_key)
        - [ngx_hash_wildcard_init](#ngx_hash_wildcard_init)
        - [ngx_hash_find](#ngx_hash_find)
        - [ngx_hash_find_wc_head](#ngx_hash_find_wc_head)
        - [ngx_hash_find_wc_tail](#ngx_hash_find_wc_tail)
        - [ngx_hash_find_combined](#ngx_hash_find_combined)
    - [原理](#原理)
        - [数据结构](#数据结构)
        - [内存布局](#内存布局)
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

哈希表初始化

函数 | 描述
-|-
[ngx_hash_init](#ngx_hash_init) | 哈希表初始化。无通配符场景。
[ngx_hash_keys_array_init](#ngx_hash_keys_array_init) | 初始化 kv 集合。这是初始化哈希表辅助函数。
[ngx_hash_add_key](#ngx_hash_add_key) | 向 kv 集合中添加 kv。这是初始化哈希表辅助函数。
[ngx_hash_wildcard_init](#ngx_hash_wildcard_init) | 哈希表初始化。有通配符的场景。

哈希表相关函数

函数 | 描述
-|-
[ngx_hash_find](#ngx_hash_find) | 从哈希表中检索关键词。无通配符场景。
[ngx_hash_find_wc_head](#ngx_hash_find_wc_head) | 查询通配符位于前缀的哈希表。
[ngx_hash_find_wc_tail](#ngx_hash_find_wc_tail) | 查询通配符位于后缀的哈希表。
[ngx_hash_find_combined](#ngx_hash_find_combined) | 结合前缀、后缀和绝对匹配的查询。


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

            // 对于通过 ngx_hash_init 初始化的哈希表必须以这样的方式来迭代 bucket
            // bucket++ 或者 bucket[i] 移动地址空间的问题在于忽略了 name 的部分数据
            // ngx_hash_elt_t 中的 name 是 char name[1]; 而 ngx_hash_init 在初始化数据的时候分配的每个 kv 对长度中包含了整个 name 的。
            bucket = (ngx_hash_elt_t *) ngx_align_ptr(&bucket->name[0] + bucket->len, sizeof(void *));
        }
    }
}
```

### ngx_hash_keys_array_init

初始化 kv 集合。这是初始化哈希表辅助函数，方便完成前缀通配、后缀通配和绝对匹配等场景的哈希表初始化。

kv 集合这里用结构体 `ngx_hash_keys_arrays_t` 表示，该结构体中有多种 `ngx_hash_key_t` 数组，用于表示前缀通配、后缀通配和绝对匹配等场景。

```c
ngx_int_t ngx_hash_keys_array_init(ngx_hash_keys_arrays_t *ha, ngx_uint_t type);
```

输入参数 | 描述
-|-
ha | ngx_hash_keys_arrays_t 结构指针。
type | 支持两种取值：NGX_HASH_SMALL 和 NGX_HASH_LARGE。

返回 NGX_OK 时成功。

**注意：**

- 因为构建 kv 集合时，会初始化简易哈希表进行去重，参数 type 代表该简易哈希表的 bucket 大小。
- 在调用该函数前，需要初始化 `ngx_hash_keys_arrays_t` 并为 `temp_pool` 指定内存池（如果不指定，无法分配内存，进程会崩溃）。

示例：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_hash_keys_arrays_t ha;
    ha.pool = r->pool;          // 到 Nginx 目前版本，没有实际意义
    ha.temp_pool = r->pool;
    ngx_hash_keys_array_init(&ha, NGX_HASH_SMALL);
}
```

### ngx_hash_add_key

向 kv 集合中添加 kv。这是初始化哈希表辅助函数。

```c
ngx_int_t ngx_hash_add_key(ngx_hash_keys_arrays_t *ha, ngx_str_t *key, void *value, ngx_uint_t flags);
```

输入参数 | 描述
-|-
ha | ngx_hash_keys_arrays_t 结构指针，存储了这各个场景的 kv 集合。
key | k。
value | v。
flags | 使用的场景。主要是两个取值：NGX_HASH_WILDCARD_KEY（通配）、NGX_HASH_READONLY_KEY（完全匹配）

**注意：**

- 每个场景下（前缀通配，后缀通配，完全匹配）都有一个简易哈希表用于存储 key，这个哈希表用来校验 key 是否已经存在。
- 如果 key 不包含通配符，则实际会添加到完全匹配到 key 中，并且会转换为小写。
- 非通配场景的哈希表生成，也常会借助该函数。
- 如果使用 NGX_HASH_WOLDCARD_KEY，因为会将其转换为小写，所以传进去的 key 的数据空间一定要是可写的（所以不能直接传常量字符串，例如 "hello"）。

通配符场景：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_hash_keys_arrays_t ha;
    ha.pool = r->pool;          // 到 Nginx 目前版本，没有实际意义
    ha.temp_pool = r->pool;
    ngx_hash_keys_array_init(&ha, NGX_HASH_SMALL);

    char *names[] = {"*.baidu.com", "*.google.com"};
    ngx_uint_t i = 0;
    for (i = 0; i < 2; i++) {
        ngx_str_t host;
        host.data = ngx_pcalloc(r->pool, ngx_strlen(names[i]));
        host.len = ngx_strlen(names[i]);
        ngx_memcpy(host.data, names[i], ngx_strlen(names[i]));

        char* s = ngx_pcalloc(r->pool, ngx_strlen(names[i]) + 1);
        ngx_memcpy(s, names[i], ngx_strlen(names[i]));
        s[ngx_strlen(names[i])] = '\0';
        ngx_log_stderr(0, "s ptr: %p, s: %s", s, s);
        ngx_hash_add_key(&ha, &host, s, NGX_HASH_WILDCARD_KEY);  
    }

    ngx_hash_init_t hinit;
    hinit.hash = NULL;
    hinit.key = ngx_hash_key;
    hinit.name = "test-hash-table";
    hinit.max_size = 128;
    hinit.bucket_size = 128;
    hinit.pool = r->pool;
    hinit.temp_pool = r->pool;

    ngx_hash_wildcard_init(&hinit, ha.dns_wc_head.elts, ha.dns_wc_head.nelts);

    ngx_str_t key = ngx_string("www.google.com");
    char* resp = ngx_hash_find_wc_head((ngx_hash_wildcard_t *) hinit.hash, key.data, key.len);
    if (resp != NULL) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "key: %V, value: %s(%p)", &key, resp, resp);
    } else {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "key: %V not found", &key);
    }

    return NGX_OK;
}
```

完全匹配场景：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_hash_keys_arrays_t ha;
    ha.pool = r->pool;
    ha.temp_pool = r->pool;
    ngx_hash_keys_array_init(&ha, NGX_HASH_SMALL);

    ngx_str_t key1 = ngx_string("hello");
    ngx_hash_add_key(&ha, &key1, "hello", NGX_HASH_READONLY_KEY);  

    ngx_str_t key2 = ngx_string("world");
    ngx_hash_add_key(&ha, &key2, "world", NGX_HASH_READONLY_KEY);  

    ngx_hash_init_t hinit;
    hinit.hash = NULL;
    hinit.key = ngx_hash_key;
    hinit.name = "test-hash-table";
    hinit.max_size = 128;
    hinit.bucket_size = 128;
    hinit.pool = r->pool;
    hinit.temp_pool = r->pool;

    ngx_hash_init(&hinit, ha.keys.elts, ha.keys.nelts);

    ngx_str_t key = ngx_string("hello");
    ngx_uint_t key_hash = ngx_hash_key(key.data, key.len);
    char* resp = ngx_hash_find(hinit.hash, key_hash, key.data, key.len);
    if (resp != NULL) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "key: %V, value: %s", &key, resp);
    } else {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "key: %V not found", &key);
    }
}
```

### ngx_hash_wildcard_init

哈希表初始化。有通配符的场景。

```c
ngx_int_t ngx_hash_wildcard_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts);
```

输入参数 | 描述
-|-
hinit | ngx_hash_init_t 结构，包含了初始化的相关参数以及内存池。
names | kv 数组。
nelts | kv 数组长度。

示例：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    // 先通过 ngx_hash_add_key 初始化相应的 ngx_hash_key_t 数组

    ngx_hash_keys_arrays_t ha;

    // ...

    ngx_hash_wildcard_init(&hinit, ha.dns_wc_head.elts, ha.dns_wc_head.nelts);

    // ...
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

返回值 | 描述
-|-
非 NULL | value 指针。
NULL | 没有找到数据。

**注意：**

- key 的计算可以通过函数：`ngx_hash_key`。

示例：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
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

查询通配符位于前缀的哈希表。

```c
void *ngx_hash_find_wc_head(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);
```

输入参数 | 描述
-|-
hash | 前缀通配的哈希表。
name | key 的数据。
len | key 的数据长度。

返回值 | 描述
-|-
非 NULL | value 指针。
NULL | 没有找到数据。

### ngx_hash_find_wc_tail

查询通配符位于后缀的哈希表。

```c
void *ngx_hash_find_wc_tail(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);
```

输入参数 | 描述
-|-
hwc | 后缀通配的哈希表。
name | key 的数据。
len | key 的数据长度。

返回值 | 描述
-|-
非 NULL | value 指针。
NULL | 没有找到数据。

### ngx_hash_find_combined

```c
void *ngx_hash_find_combined(ngx_hash_combined_t *hash, ngx_uint_t key, u_char *name, size_t len);
```

输入参数 | 描述
-|-
hash | 联合哈希表。
name | key 的数据。
len | key 的数据长度。

返回值 | 描述
-|-
非 NULL | value 指针。
NULL | 没有找到数据。

**注意：**

- 联合哈希表中有三种场景的哈希表：
  - 完全匹配 hash
  - 前缀通配 wc_head
  - 后缀通配 wc_tail
- 联合哈希表的检索顺序：
  - 先查询完全匹配哈希表
  - 再查询前缀通配
  - 最后查后缀通配
- 只有在当前哈希表没有查询到 value 时，才会继续往下一个哈希表查询。

示例：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r) {
    ngx_hash_keys_arrays_t ha;
    ha.pool = r->pool;
    ha.temp_pool = r->pool;
    ngx_hash_keys_array_init(&ha, NGX_HASH_SMALL);

    char *names[] = {"*.baidu.com", "*.google.com", "www.example.*"};
    ngx_uint_t i = 0;
    for (i = 0; i < 3; i++) {
        ngx_str_t host;
        host.data = ngx_pcalloc(r->pool, ngx_strlen(names[i]));
        host.len = ngx_strlen(names[i]);
        ngx_memcpy(host.data, names[i], ngx_strlen(names[i]));

        char* s = ngx_pcalloc(r->pool, ngx_strlen(names[i]) + 1);
        ngx_memcpy(s, names[i], ngx_strlen(names[i]));
        s[ngx_strlen(names[i])] = '\0';
        ngx_log_stderr(0, "s ptr: %p, s: %s", s, s);
        ngx_hash_add_key(&ha, &host, s, NGX_HASH_WILDCARD_KEY);  
    }

    ngx_str_t h = ngx_string("www.qq.cn");
    ngx_hash_add_key(&ha, &h, "www.qq.cn", NGX_HASH_READONLY_KEY);  

    ngx_hash_init_t hinit1;
    hinit1.hash = NULL;
    hinit1.key = ngx_hash_key;
    hinit1.name = "test-hash-table";
    hinit1.max_size = 128;
    hinit1.bucket_size = 128;
    hinit1.pool = r->pool;
    hinit1.temp_pool = r->pool;

    ngx_hash_init_t hinit2 = hinit1;
    ngx_hash_init_t hinit3 = hinit1;

    ngx_hash_combined_t combined_hash;
    combined_hash.wc_head = NULL;
    combined_hash.wc_tail = NULL;

    ngx_hash_init(&hinit1, ha.keys.elts, ha.keys.nelts);
    combined_hash.hash = *hinit1.hash;

    ngx_hash_wildcard_init(&hinit2, ha.dns_wc_head.elts, ha.dns_wc_head.nelts);
    combined_hash.wc_head = (ngx_hash_wildcard_t  *) hinit2.hash;

    ngx_hash_wildcard_init(&hinit3, ha.dns_wc_tail.elts, ha.dns_wc_tail.nelts);
    combined_hash.wc_tail = (ngx_hash_wildcard_t  *) hinit3.hash;

    ngx_str_t key = ngx_string("www.qq.cn");
    ngx_uint_t key_hash = ngx_hash_key(key.data, key.len);
    char* resp = ngx_hash_find_combined(&combined_hash, key_hash, key.data, key.len);
    if (resp != NULL) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "========= key: %V, value: %s(%p)", &key, resp, resp);
    } else {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "========= key: %V not found", &key);
    }
    return NGX_OK;
}
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

// 通过 ngx_hash_keys_arrays_s 对哈希表初始化，该结构可以对 ngx_hash_init_s 初始化
struct {
    ngx_uint_t        hsize;                // 临时简易哈希表的 bucket 个数，因为为了校验 key 是否存在，需要一个临时哈希表。

    ngx_pool_t       *pool;                 // 目前没有实际使用。
    ngx_pool_t       *temp_pool;            // 用于分配临时哈希表、ngx_hash_keys_arrays_s 的内存池。

    ngx_array_t       keys;                 // 完全匹配的 key，元素类型为 ngx_hash_key_s
    ngx_array_t      *keys_hash;            // 临时哈希表，用于校验完全匹配的 key，每一个元素是 ngx_str_t 的动态数组，即 bucket。

    ngx_array_t       dns_wc_head;          // 前缀通配的 key，元素类型为 ngx_hash_key_s
    ngx_array_t      *dns_wc_head_hash;     // 临时哈希表，用于校验前缀通配的 key，每一个元素是 ngx_str_t 的动态数组，即 bucket。

    ngx_array_t       dns_wc_tail;          // 后缀通配的 key，元素类型为 ngx_hash_key_s
    ngx_array_t      *dns_wc_tail_hash;     // 临时哈希表，用于校验后缀通配的 key，每一个元素是 ngx_str_t 的动态数组，即 bucket。
} ngx_hash_keys_arrays_s;


struct {
    ngx_hash_t        hash;

    // 多级 hash 时，父级的 key 指向子 hash。
    // 是若该 key 除了子 hash 外，还存在绝对匹配 value，通过该字段指向。
    void             *value;
} ngx_hash_wildcard_s;


struct {
    ngx_hash_t            hash;             // 完全匹配哈希表
    ngx_hash_wildcard_t  *wc_head;          // 前缀通配哈希表
    ngx_hash_wildcard_t  *wc_tail;          // 后缀通配哈希表
} ngx_hash_combined_s;


struct {
    ngx_uint_t        hash;
    ngx_str_t         key;
    ngx_str_t         value;
    u_char           *lowcase_key;
} ngx_table_elt_s;
```

### 内存布局

![](/resource/ngx_hash_t.png)

### 通配符检索

通配符的哈希表是多级哈希，`.` 是划分多级的依据。

通配符的哈希表存储 Value 时，会用最低的两个比特作为标识，不同类型的 Value 有不同的取值：

标识 | 描述
-|-
3 | Value 对应的是子 Hash 表。
2 | Value 对应的是子 Hash 表，并且子 Hash 表的 value 字段是当前 key 完全匹配的。
1 | Value 是真实的数据地址（除了最低位），通常为 1 时表示已经找到通配的，并且无法继续找下去。
0 | Value 是真实的数据地址，通常为 0 时表示已经找到完全匹配的 Key。

可参考下图：

![](/resource/ngx-hash-table-wildcard-sample.png)
