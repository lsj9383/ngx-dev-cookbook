# 队列

## 概述

Nginx 提供了 `ngx_queue_t` 结构以支持队列，该队列结构提供了排序函数。

**注意：**

- 该结构名称叫做队列，但是并非只能先进先出。该结构提供的函数更类似一个链表。

内存池源代码位置：

- src/core/ngx_queue.h
- src/core/ngx_queue.c

函数 | 描述
-|-
[ngx_queue_init](#ngx_queue_init) | 初始化队列。
[ngx_queue_empty](#ngx_queue_empty) | 判断队列是否为空。
[ngx_queue_insert_head](#ngx_queue_insert_head) | 在队列头部插入一个队列节点
[ngx_queue_insert_tail](#ngx_queue_insert_tail) | 在队列的末尾插入一个队列节点。
[ngx_queue_head](#ngx_queue_head) | 队列的头部。
[ngx_queue_last](#ngx_queue_last) | 队列的尾部。
[ngx_queue_next](#ngx_queue_next) | 队列节点的下一个元素。
[ngx_queue_prev](#ngx_queue_prev) | 队列节点的上一个元素。
[ngx_queue_remove](#ngx_queue_remove) | 删除队列元素。
[ngx_queue_split](#ngx_queue_split) | 将一个队列拆分成两个队列。
[ngx_queue_add](#ngx_queue_add) | 合并两个链表，将第二个链表添加至第一个链表的末尾。
[ngx_queue_data](#ngx_queue_data) | 获得队列对应的数据对象。
[ngx_queue_middle](#ngx_queue_middle) | 找到队列的中间元素。
[ngx_queue_sort](#ngx_queue_sort) | 队列进行排序。

**注意：**

- 队列 `ngx_queue_t` 中不包含用户数据，而是用户数据中包含 `ngx_queue_t`。

  ```c
  // 例如
  class Student {
      ngx_str_t id;
      ngx_str_t name;
      ngx_queue_t q;
  };
  ```

- 队列需要有一个不被用户数据包含的 `ngx_queue_t` 用于表示整个队列，而不是某一个节点。可以看出 `ngx_queue_t` 其实有两岑含义：
  - 队列中的一个节点，即队列节点，通常被放置于用户定义的类中。
  - 队列本身，即队列对象。
- 队列的底层实现是一个双向循环链表：

## 函数

### ngx_queue_init

初始化队列。通过宏实现：

```c
#define ngx_queue_init(q)                                                     \
    (q)->prev = q;                                                            \
    (q)->next = q
```

输入参数 | 描述
-|-
q | 需要进行初始化的队列对象，指针。

示例：

```c
ngx_queue_t q;
ngx_queue_init(&q);
```

### ngx_queue_empty

判断队列对象是否为空。通过宏实现：

```c
#define ngx_queue_empty(h)                                                    \
    (h == (h)->prev)
```

输入参数 | 描述
-|-
q | 队列对象指针。判断该队列对象是否为空。

示例：

```c
ngx_queue_t q;
ngx_queue_init(&q);
ngx_queue_empty(&q)
```

### ngx_queue_insert_head

在队列头部插入一个队列节点。通过宏实现：

```c
#define ngx_queue_insert_head(h, x)                                           \
    (x)->next = (h)->next;                                                    \
    (x)->next->prev = x;                                                      \
    (x)->prev = h;                                                            \
    (h)->next = x
```

输入参数 | 描述
-|-
h | 队列对象指针。
x | 待插入的队列节点指针。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode node;
node.value = 0;

ngx_queue_t q;
ngx_queue_init(&q);
ngx_queue_insert_head(&q, &node.q);
```

### ngx_queue_insert_tail

在队列的末尾插入元素。通过宏实现：

```c
#define ngx_queue_insert_tail(h, x)                                           \
    (x)->prev = (h)->prev;                                                    \
    (x)->prev->next = x;                                                      \
    (x)->next = h;                                                            \
    (h)->prev = x
```

输入参数 | 描述
-|-
h | 队列对象指针。
x | 待插入的队列节点指针。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode node;
node.value = 0;

ngx_queue_t q;
ngx_queue_init(&q);
ngx_queue_insert_tail(&q, &node.q);
```

### ngx_queue_head

队列的头部节点。通过宏实现：

```c
#define ngx_queue_head(h)                                                     \
    (h)->next
```

输入参数 | 描述
-|-
h | 队列对象指针。

返回值就是队列头部元素，

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode nn[3];
nn[0].value = 0;
nn[1].value = 1;
nn[2].value = 2;

ngx_queue_t q;
ngx_queue_init(&q);
ngx_queue_insert_tail(&q, &nn[0].q);
ngx_queue_insert_tail(&q, &nn[1].q);
ngx_queue_insert_tail(&q, &nn[2].q);

ngx_head(&h);
```

### ngx_queue_last

队列的尾部节点。通过宏实现：

```c
#define ngx_queue_last(h)                                                     \
    (h)->prev
```

输入参数 | 描述
-|-
h | 队列对象。

返回值是队列对象末尾节点。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode nn[3];
nn[0].value = 0;
nn[1].value = 1;
nn[2].value = 2;

ngx_queue_t q;
ngx_queue_init(&q);
ngx_queue_insert_tail(&q, &nn[0].q);
ngx_queue_insert_tail(&q, &nn[1].q);
ngx_queue_insert_tail(&q, &nn[2].q);

ngx_queue_last(&h);
```

### ngx_queue_next

队列节点的下一个元素。通过宏实现：

```c
#define ngx_queue_next(q)                                                     \
    (q)->next
```

输入参数 | 描述
-|-
q | 队列节点指针。

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode nn[3];
nn[0].value = 0;
nn[1].value = 1;
nn[2].value = 2;

ngx_queue_t q;
ngx_queue_init(&q);
ngx_queue_insert_tail(&q, &nn[0].q);
ngx_queue_insert_tail(&q, &nn[1].q);
ngx_queue_insert_tail(&q, &nn[2].q);

ngx_queue_next(&nn[1].q);
```

### ngx_queue_prev

队列的上一个元素。通过宏实现：

```c
#define ngx_queue_next(q)                                                     \
    (q)->next
```

输入参数 | 描述
-|-
q | 队列节点指针。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode nn[3];
nn[0].value = 0;
nn[1].value = 1;
nn[2].value = 2;

ngx_queue_t q;
ngx_queue_init(&q);
ngx_queue_insert_tail(&q, &nn[0].q);
ngx_queue_insert_tail(&q, &nn[1].q);
ngx_queue_insert_tail(&q, &nn[2].q);

ngx_queue_prev(&nn[1].q);
```

### ngx_queue_remove

删除队列中的元素。通过宏实现：

```c
#define ngx_queue_remove(x)                                                   \
    (x)->next->prev = (x)->prev;                                              \
    (x)->prev->next = (x)->next;                                              \
    (x)->prev = NULL;                                                         \
    (x)->next = NULL
```

输入参数 | 描述
-|-
x | 需要从队列中剔除的节点指针。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode nn[3];
nn[0].value = 0;
nn[1].value = 1;
nn[2].value = 2;

ngx_queue_t q;
ngx_queue_init(&q);
ngx_queue_insert_tail(&q, &nn[0].q);
ngx_queue_insert_tail(&q, &nn[1].q);
ngx_queue_insert_tail(&q, &nn[2].q);

ngx_queue_remove(&nn[1].q);
```

### ngx_queue_split

将一个队列拆分成两个队列。通过宏实现：

```c
#define ngx_queue_split(h, q, n)                                              \
    (n)->prev = (h)->prev;                                                    \
    (n)->prev->next = n;                                                      \
    (n)->next = q;                                                            \
    (h)->prev = (q)->prev;                                                    \
    (h)->prev->next = h;                                                      \
    (q)->prev = n;
```

输入参数 | 描述
-|-
h | 队列对象指针。
q | 进行分裂的点，队列节点指针。
n | 一个新的队列对象指针。

**注意：**

- 分裂出来两个队列，h 是原本队列对象，新的队列对象用 n。
- 分裂的点 q 会进入新的队列 n。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode nn[3];
nn[0].value = 0;
nn[1].value = 1;
nn[2].value = 2;

ngx_queue_t q;
ngx_queue_init(&q);
ngx_queue_insert_tail(&q, &nn[0].q);
ngx_queue_insert_tail(&q, &nn[1].q);
ngx_queue_insert_tail(&q, &nn[2].q);

ngx_queue_t n;
ngx_queue_init(&n);
ngx_queue_split(&h, &nn[1].q, &n);
```

### ngx_queue_add

合并两个链表，将第二个链表添加至第一个链表的末尾。通过宏实现：

```c
#define ngx_queue_add(h, n)                                                   \
    (h)->prev->next = (n)->next;                                              \
    (n)->next->prev = (h)->prev;                                              \
    (h)->prev = (n)->prev;                                                    \
    (h)->prev->next = h;
```

输入参数 | 描述
-|-
h | 队列对象指针。
n | 另外一个队列对象指针。

**注意：**

- 宏只改变了 h 队列对象的指针信息，因此合并后，新的队列对象用 h 表示。
- 合并后 n 已经无效（不会有节点指向 n）。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode hn[3];
hn[0].value = 0;
hn[1].value = 1;
hn[2].value = 2;

ngx_queue_t h;
ngx_queue_init(&h);
ngx_queue_insert_tail(&h, &hn[0].q);
ngx_queue_insert_tail(&h, &hn[1].q);
ngx_queue_insert_tail(&h, &hn[2].q);

struct NumberNode nn[3];
nn[0].value = 100;
nn[1].value = 101;
nn[2].value = 102;

ngx_queue_t n;
ngx_queue_init(&n);
ngx_queue_insert_tail(&n, &nn[0].q);
ngx_queue_insert_tail(&n, &nn[1].q);
ngx_queue_insert_tail(&n, &nn[2].q);

// 合并
ngx_queue_add(&h, &n);
```

### ngx_queue_data

获得队列对应的数据对象。通过宏实现：

```c
#define ngx_queue_data(q, type, link)                                         \
    (type *) ((u_char *) q - offsetof(type, link))
```

输入参数 | 描述
-|-
q | 队列节点指针。
type | 数据类型。
link | 数据类型中该节点的位置。

返回对应的数据对象指针。

**注意：**

- `offsetof(type, link)` 计算出队列字段在类型中的偏移量。
- 将队列节点指针 - 偏移量就能找到数据对象的首地址。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode hn;
hn.value = 0;


struct NumberNode *n = ngx_queue_data(&hn[0].q, NumberNode, q);
```

### ngx_queue_middle

找到队列的中间元素。

函数声明：

```c
ngx_queue_t *ngx_queue_middle(ngx_queue_t *queue);
```

输入参数 | 描述
-|-
queue | 队列对象指针。

返回值是中间节点指针。

**注意：**

- 寻找遍历一次就够了，内部维护了 next 和 middle 两个指针，next 每次走两步，middle 每次走一步，因此 next 是 middle 的两倍，当 next 走到终点时，middle 就是队列的中间元素。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

struct NumberNode hn[3];
hn[0].value = 0;
hn[1].value = 1;
hn[2].value = 2;

ngx_queue_t h;
ngx_queue_init(&h);
ngx_queue_insert_tail(&h, &hn[0].q);
ngx_queue_insert_tail(&h, &hn[1].q);
ngx_queue_insert_tail(&h, &hn[2].q);

ngx_queue_t *n = ngx_queue_middle(&h);
```

### ngx_queue_sort

队列进行排序。

函数声明：

```c
void ngx_queue_sort(ngx_queue_t *queue, ngx_int_t (*cmp)(const ngx_queue_t *, const ngx_queue_t *));
```

输入参数 | 描述
-|-
queue | 队列对象指针。
cmp | 队列节点的比较方法。

**注意：**

- 该排序算法不适宜大数据量的排序，因为采用的是简单的选择排序。

示例：

```c
struct NumberNode {
    int value;
    ngx_queue_t q;
};

ngx_int_t ngx_number_node_cmp(const ngx_queue_t *n1, const ngx_queue_t *n2) {
    return n1->value > n2->value;
}

struct NumberNode hn[3];
hn[0].value = 0;
hn[1].value = 1;
hn[2].value = 2;

ngx_queue_t h;
ngx_queue_init(&h);
ngx_queue_insert_tail(&h, &hn[0].q);
ngx_queue_insert_tail(&h, &hn[1].q);
ngx_queue_insert_tail(&h, &hn[2].q);

ngx_queue_sort(&h, ngx_number_node_cmp);
```

## 原理

### 内存布局

![](/resource/ngx_queue_t.png)

从内存布局中可以看到，队列其实就是一个循环双向链表结构，并且有一个 `ngx_queue_t` 的特殊节点用来代表整个队列，该节点不被包含于数据结构中。

对于其他 `ngx_queue_t` 的节点，都是处于数据结构中。

很明显，prev 指向上一个元素，last 指向下一个元素。

对于队列对象的特殊节点 `ngx_queue_t`，prev 就是末尾节点，next 就是首节点。

### 队列节点转为数据指针

`ngx_queue_t` 节点只包含了 prev 和 next，并没有包含指向数据的指针，其实现方式是依据地址偏移量的计算：

- 首先计算出 `ngx_queue_t` 结数据结构体中的偏移量。
- 用 `ngx_queue_t` 队列节点 - 偏移量就能获得数据结构体对象的首地址。
