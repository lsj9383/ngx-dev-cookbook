# 红黑树

<!-- TOC -->

- [红黑树](#红黑树)
    - [概览](#概览)
    - [函数](#函数)
        - [ngx_rbtree_init](#ngx_rbtree_init)
        - [ngx_rbtree_insert](#ngx_rbtree_insert)
        - [ngx_rbtree_delete](#ngx_rbtree_delete)
        - [ngx_rbtree_next](#ngx_rbtree_next)
        - [ngx_str_rbtree_lookup](#ngx_str_rbtree_lookup)
        - [ngx_rbtree_insert_value](#ngx_rbtree_insert_value)
        - [ngx_rbtree_insert_timer_value](#ngx_rbtree_insert_timer_value)
        - [ngx_str_rbtree_insert_value](#ngx_str_rbtree_insert_value)
        - [ngx_rbtree_min](#ngx_rbtree_min)
        - [ngx_rbtree_sentinel_init](#ngx_rbtree_sentinel_init)
        - [ngx_rbt_red](#ngx_rbt_red)
        - [ngx_rbt_black](#ngx_rbt_black)
        - [ngx_rbt_is_red](#ngx_rbt_is_red)
        - [ngx_rbt_is_black](#ngx_rbt_is_black)
        - [ngx_rbt_copy_color](#ngx_rbt_copy_color)
    - [原理](#原理)
        - [结构体](#结构体)
        - [内存布局](#内存布局)

<!-- /TOC -->

## 概览

Nginx 通过红黑树 `ngx_rbtree_t` 提供可以进行快速插入和查询的字典（`ngx_hash_t` 也能提供字典，但是只能进行查询）。

源码位置：

- src/core/ngx_rbtree.h
- src/core/ngx_rbtree.c

Nginx 红黑树相关函数：

函数 | 描述
-|-
ngx_rbtree_init | rbt 初始化。
ngx_rbtree_insert | 向红黑树中插入一个节点，并通过左旋右旋调整平衡。
ngx_rbtree_delete | 从红黑树中删除一个节点。
ngx_rbtree_next | 获取红黑树中的下一个节点，用于树的遍历迭代。
ngx_str_rbtree_lookup | 查询红黑树中的节点，仅支持字符串作为 key。

Nginx 自带的插入处理函数：

函数 | 描述
-|-
ngx_rbtree_insert_value | 用 key 作为插入的依据，如果 key 重复，则新的插入将无法被查询出来。
ngx_rbtree_insert_timer_value | 类似于 ngx_rbtree_insert_value。
ngx_str_rbtree_insert_value | 用 key 和字符串作为插入的依据，如果 key 和字符串重复，则新的插入将无法被查询出来。

Nginx 红黑树节点相关函数：

函数 | 描述
-|-
ngx_rbtree_min | 获得 Key 最小的节点。也就是 rbtree 中最左侧的叶子节点。
ngx_rbtree_sentinel_init | 初始化 rbt 的哨兵节点。哨兵节点其实就是通用的尾巴节点，是所有叶子节点的子节点。
ngx_rbt_red | 设置节点为红色。
ngx_rbt_black | 设置节点为黑色。
ngx_rbt_is_red | 判断节点是否为红色。
ngx_rbt_is_black | 判断节点是否为黑色。
ngx_rbt_copy_color | 将节点 A 的颜色复制给节点 B。

## 函数

### ngx_rbtree_init

红黑树初始化。

通过宏实现：

```c
#define ngx_rbtree_init(tree, s, i)                                           \
    ngx_rbtree_sentinel_init(s);                                              \
    (tree)->root = s;                                                         \
    (tree)->sentinel = s;                                                     \
    (tree)->insert = i
```

输入参数 | 描述
-|-
tree | 红黑树指针，需要已分配好的内存空间。
s | 初始节点，该节点即是根节点 root，也是哨兵节点 sentinel，要求已经分配好内存。
insert | 函数，由插入时调用，需要判断在什么位置进行节点插入。

**注意：**

- insert 函数指针只需要负责插入，树的平衡由 Nginx 控制。
- 目前 Nginx 已经实现了可以直接使用的 insert 函数：
  - ngx_rbtree_insert_value
  - ngx_str_rbtree_insert_value

示例：

```c
ngx_rbtree_node_t sentinel;
ngx_rbtree_t rbtree;
ngx_rbtree_init(&rbtree, &sentinel, ngx_str_rbtree_insert_value);
```

如果某个数据结构中需要维护多个可以用 Key 进行快速查询的数据，通常会在该数据结构中存放 sentienl 和 rbtree，例如：

```c
// 对于 UDP，监听端口结构体根据请求地址维护了多个 ngx_connection_t
// 可以通过 sockaddr 快速检索 ngx_connection_t
// 也就是相同 sockaddr 的 UDP 请求使用的是相同的 ngx_connection_t

struct ngx_listening_s {
    ...
    ngx_rbtree_t        rbtree;
    ngx_rbtree_node_t   sentinel;
    ...
}

ngx_rbtree_init(&ls->rbtree, &ls->sentinel, ngx_udp_rbtree_insert_value);
```

### ngx_rbtree_insert

向红黑树中插入一个节点，并通过左旋右旋调整平衡。

函数声明：

```c
void ngx_rbtree_insert(ngx_rbtree_t *tree, ngx_rbtree_node_t *node);
```

输入参数 | 描述
-|-
tree | 红黑树指针。
node | 需要插入的节点。

示例：

```c
struct NumberNode {
    ngx_rbtree_node_t node;
    ngx_uint_t number;
};

// ... 初始化所有的 node ...
struct NumberNode nns[5];
nns[0].node.key = 3;
nns[0].number = 3;

// ... 初始化红黑树 ...
ngx_rbtree_node_t sentinel;
ngx_rbtree_t dict;
ngx_rbtree_init(&dict, &sentinel, ngx_rbtree_insert_value);

// ... 插入多个节点 ...
ngx_rbtree_insert(&dict, &nns[0].node);
ngx_rbtree_insert(&dict, &nns[1].node);
ngx_rbtree_insert(&dict, &nns[2].node);
ngx_rbtree_insert(&dict, &nns[3].node);
```

为了方便直接使用字符串作为 Key，可以借助 `ngx_str_node_t`。

对于使用 `ngx_str_node_t` 的示例：

```c
struct NumberNode {
    ngx_str_node_t node;
    ngx_uint_t number;
};

// ... 初始化所有的 node ...
struct NumberNode nns[5];
ngx_str_set(&nns[0].node.str, "name");
nns[0].node.node.key = ngx_hash_key(nns[0].node.str.data, nns[0].node.str.len);
nns[0].number = 3;

// ... 初始化红黑树 ...
ngx_rbtree_node_t sentinel;
ngx_rbtree_t dict;
ngx_rbtree_init(&dict, &sentinel, ngx_str_rbtree_insert_value);

// ... 插入多个节点 ...
ngx_rbtree_insert(&dict, &nns[0].node.node);
ngx_rbtree_insert(&dict, &nns[1].node.node);
ngx_rbtree_insert(&dict, &nns[2].node.node);
ngx_rbtree_insert(&dict, &nns[3].node.node);
```

### ngx_rbtree_delete

从红黑树中删除一个节点，并通过左旋右旋调整平衡。

函数声明：

```c
void ngx_rbtree_delete(ngx_rbtree_t *tree, ngx_rbtree_node_t *node);
```

输入参数 | 描述
-|-
tree | 红黑树指针。
node | 需要删除的节点。

示例：

```c
// ... 初始化节点 ...
// ... 初始化红黑树 ...
// ... 初始化红黑树 ...

// ... 删除红黑树节点 ...
ngx_rbtree_delete(&dict, &nns[1].node.node);
```

### ngx_rbtree_next

获取红黑树中的下一个节点，用于树的遍历迭代。

函数声明：

```c
ngx_rbtree_node_t *ngx_rbtree_next(ngx_rbtree_t *tree, ngx_rbtree_node_t *node);
```

输入参数 | 描述
-|-
tree | 红黑树指针。
node | 当前节点。

返回值 | 描述
-|-
非 NULL | 下一个节点。
NULL | 迭代完毕。

示例：

```c
// 通过以下方式迭代，可以获得 Key 从小到大排序的节点

// ... 初始化红黑树 ...
ngx_rbtree_node_t sentinel;
ngx_rbtree_t dict;
ngx_rbtree_init(&dict, &sentinel, ngx_str_rbtree_insert_value);

// ... 插入数据 ...

// 迭代查询
ngx_rbtree_node_t* node;
for (node = ngx_rbtree_min(dict.root, dict.sentinel);
    node;
    node = ngx_rbtree_next(&dict, node))
{
    ngx_log_stderr(0, "node key: %ui", node->key);
}
```

### ngx_str_rbtree_lookup

查询红黑树中的节点，仅支持字符串作为 key。源码存放位置：`src/core/nginx_string.c`

函数声明：

```c
ngx_str_node_t *ngx_str_rbtree_lookup(ngx_rbtree_t *rbtree, ngx_str_t *name, uint32_t hash);
```

输入参数 | 描述
-|-
tree | 红黑树指针。
name | 节点名称。
hash | 节点 Key。

**注意：**

- 要求 hash 和 name 都和 rbtree 维护的节点相同是，才会返回该节点。
- 要使用该函数，要求节点自定义数据结构的第一个字段是 `ngx_str_rbtree_t`。

示例：

```c
struct NumberNode {
    ngx_str_node_t node;
    ngx_uint_t number;
};

// ... 初始化节点数据 ...
struct NumberNode nns[5];
ngx_str_set(&nns[0].node.str, "name");
nns[0].node.node.key = ngx_hash_key(nns[0].node.str.data, nns[0].node.str.len);
nns[0].number = 3;

ngx_str_set(&nns[1].node.str, "age");
nns[1].node.node.key = ngx_hash_key(nns[1].node.str.data, nns[1].node.str.len);
nns[1].number = 4;

ngx_str_set(&nns[2].node.str, "id");
nns[2].node.node.key = ngx_hash_key(nns[2].node.str.data, nns[2].node.str.len);
nns[2].number = 9;

ngx_str_set(&nns[3].node.str, "icon");
nns[3].node.node.key = ngx_hash_key(nns[3].node.str.data, nns[3].node.str.len);
nns[3].number = 6;

// 设置一个重复的 str
ngx_str_set(&nns[4].node.str, "name");
nns[4].node.node.key = ngx_hash_key(nns[4].node.str.data, nns[4].node.str.len);
nns[4].number = 7;

// ... 初始化 rbtree ...
ngx_rbtree_node_t sentinel;
ngx_rbtree_t dict;
ngx_rbtree_init(&dict, &sentinel, ngx_str_rbtree_insert_value);

// ... 插入数据 ...
ngx_rbtree_insert(&dict, &nns[0].node.node);
ngx_rbtree_insert(&dict, &nns[1].node.node);
ngx_rbtree_insert(&dict, &nns[2].node.node);
ngx_rbtree_insert(&dict, &nns[3].node.node);
ngx_rbtree_insert(&dict, &nns[4].node.node);

// 查询数据
ngx_str_t find = ngx_string("name");
struct NumberNode *found_node = (struct NumberNode *)ngx_str_rbtree_lookup(&dict, &find, ngx_hash_key(find.data, find.len));
if (found_node) {
    ngx_log_stderr(0, "%V found, value: %ui", &find, found_node->number);
} else {
    ngx_log_stderr(0, "%V not found", &find);
}
```

### ngx_rbtree_insert_value

用节点的 Key 作为插入的依据，要求 key 不能重复。

### ngx_rbtree_insert_timer_value

类似于 ngx_rbtree_insert_value。

### ngx_str_rbtree_insert_value

用节点的 Key 和字符串作为插入的依据，若 Key 和字符串重复，则覆盖。

### ngx_rbtree_min

获得 Key 最小的节点。也就是 rbtree 中最左侧的叶子节点。

函数声明：

```c
ngx_rbtree_node_t *ngx_rbtree_min(ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel);
```

输入参数 | 描述
-|-
node | 首个节点。该函数查询该节点的最左侧的叶子节点。通常会选择使用 rbtree 的根节点。
sentienl | 末尾节点。通常会选择使用 rbtree 的哨兵节点。

返回 Key 最小的节点。

### ngx_rbtree_sentinel_init

初始化 rbt 的哨兵节点。哨兵节点其实就是通用的尾巴节点，是所有叶子节点的子节点。

通过宏实现：

```c
#define ngx_rbtree_sentinel_init(node)  ngx_rbt_black(node)
```

输入参数 | 描述
-|-
node | sentienl 节点

**注意：**

- sentienl 节点必须为黑色。
- 通常不会直接调用该函数，在 ngx_rbtree_init 的函数中就会初始化 sentienl 节点。

### ngx_rbt_red

设置节点为红色。

通过宏实现：

```c
#define ngx_rbt_red(node)               ((node)->color = 1)
```

输入参数 | 描述
-|-
node | 需要设置的节点。

**注意：**

- 通常不会直接调用该函数。

### ngx_rbt_black

设置节点为黑色。

通过宏实现：

```c
#define ngx_rbt_black(node)             ((node)->color = 0)
```

输入参数 | 描述
-|-
node | 需要设置的节点。

**注意：**

- 通常不会直接调用该函数。

### ngx_rbt_is_red

判断节点是否为红色。

通过宏实现：

```c
#define ngx_rbt_is_red(node)            ((node)->color)
```

输入参数 | 描述
-|-
node | 需要判断的节点。

返回值 | 描述
-|-
true | 红色节点。
false | 黑色节点。

**注意：**

- 通常不会直接调用该函数。

### ngx_rbt_is_black

判断节点是否为黑色。

通过宏实现：

```c
#define ngx_rbt_is_black(node)          (!ngx_rbt_is_red(node))
```

输入参数 | 描述
-|-
node | 需要判断的节点。

返回值 | 描述
-|-
true | 黑色节点。
false | 红色节点。

**注意：**

- 通常不会直接调用该函数。

### ngx_rbt_copy_color

将节点 A 的颜色复制给节点 B。

通过宏实现：

```c
#define ngx_rbt_copy_color(n1, n2)      (n1->color = n2->color)
```

**注意：**

- 通常不会直接调用该函数。

## 原理

### 结构体

```c
typedef ngx_rbtree_node_s       ngx_rbtree_node_t;
typedef ngx_rbtree_s            ngx_rbtree_t;


// 红黑树结构体
struct ngx_rbtree_s {
    ngx_rbtree_node_t     *root;            // 红黑树根节点
    ngx_rbtree_node_t     *sentinel;        // 红黑树哨兵
    ngx_rbtree_insert_pt   insert;          // 插入节点是调用的函数
};


// 红黑树节点结构
struct ngx_rbtree_node_s {
    ngx_rbtree_key_t       key;             // 节点 Key
    ngx_rbtree_node_t     *left;            // 左节点
    ngx_rbtree_node_t     *right;           // 右节点
    ngx_rbtree_node_t     *parent;          // 父节点
    u_char                 color;           // 节点颜色
    u_char                 data;            // 节点数据 只有 1 字节，基本上不会使用
};

// src/core/ngx_string.h
// 如果需要字符串作为节点的 Key，可以使用 ngx_str_node_t 作为节点的结构体
typedef struct {
    ngx_rbtree_node_t         node;
    ngx_str_t                 str;
} ngx_str_node_t;
```

### 内存布局
