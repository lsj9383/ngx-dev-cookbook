# 字符串


<!-- TOC -->

- [字符串](#字符串)
    - [概述](#概述)
        - [初始化和赋值](#初始化和赋值)
        - [字符串处理](#字符串处理)
        - [字符串转换相关](#字符串转换相关)
        - [字符串 Base 64](#字符串-base-64)
        - [字符串 UTF-8](#字符串-utf-8)
        - [格式化函数](#格式化函数)
        - [内存空间处理](#内存空间处理)
    - [函数](#函数)
        - [ngx_atoi](#ngx_atoi)
        - [ngx_atosz](#ngx_atosz)
        - [ngx_atoof](#ngx_atoof)
        - [ngx_atotm](#ngx_atotm)
        - [ngx_hextoi](#ngx_hextoi)
        - [ngx_hex_dump](#ngx_hex_dump)
        - [ngx_sprintf](#ngx_sprintf)
        - [ngx_snprintf](#ngx_snprintf)
        - [ngx_slprintf](#ngx_slprintf)
        - [ngx_vslprintf](#ngx_vslprintf)
        - [ngx_vsnprintf](#ngx_vsnprintf)
    - [原理](#原理)
        - [数据结构](#数据结构)

<!-- /TOC -->
## 概述

Nginx 中对字符串进行了封装，类型为 `ngx_str_t`，并且提供了非常丰富的方法进行字符串的处理。

在 Nginx 中绝大多数使用的是 `ngx_str_t` 而非 C 风格字符串（以 `\0` 作为结尾符的数组，后面称这类字符串为 CString）。`ngx_str_t` 通常不能直接使用 CString 相关的库函数，而是使用 Nginx 提供的字符串函数。

源代码位置：

- src/core/ngx_string.h
- src/core/ngx_string.c

### 初始化和赋值

函数 | 示例 | 描述
-|-|-
ngx_string | ngx_str_t s = ngx_string("hello"); | 通过 CString 对 `ngx_str_t` 进行**初始化**（不是赋值运算）。
ngx_null_string | ngx_str_s empty = ngx_null_string; | 初始化 `ngx_str_t` 空字符串（不是赋值运算） 。
ngx_str_set | ngx_str_set(&s, "hello"); | 通过 CString 为 `ngx_str_t` 赋值，要求传入的是 `ngx_str_t*` 。
ngx_str_null | ngx_str_null(&s); | 赋值 `ngx_str_t` 为空字符串，要求传入的是 `ngx_str_t*`。

**注意：**

- 需要区分初始化和赋值的概念，初始化只有在定义变量的时候可以用，赋值是在定义变量后可以使用。
  - 正确 `ngx_str_t s = ngx_string("hello");` 这是一个初始化。
  - 错误 `data.s = ngx_string("hello");` 这是一个赋值运算。
- 当 ngx_str_t s 为空字符串时，s->data 为 NULL，s->len 为 0。

### 字符串处理

函数 | 示例 | 描述
-|-|-
ngx_strlow | ngx_strlow(s->data, s->data, s->len); | 将指定地址的前 n 个字符转换为小写。
ngx_strncmp | ngx_strncmp("hello", s->data, s->len); | 将请 n 个字符进行比较，相同返回 0，不同返回非 0。
ngx_strcmp | ngx_strcmp(cstring, "auto") | 比较两字符串是否相等，以 `\0` 作为结尾符号（CString）。
ngx_strstr | ngx_strstr(cstring, "auto") | 从指定地址搜索字符串，并返回子串第一次出现的地址。以 `\0` 作为搜索的结尾符（CString）。
ngx_strstrn | ngx_strstrn(cstring, "Chrome/", 6); | 从指定地址搜索字符串，搜索的字符串长度为 n。以 `\0` 作为搜索的结尾符（CString）。
ngx_strnstr | |
ngx_strlen | ngx_strlen(cstring); | 返回指定地址的字符串长度，通过判断 `\0` 实现（CString）。
ngx_strnlen | ngx_strnlen(cstring, max); | 返回指定地址的字符串长度，通过判断 `\0` 实现（CString）。判断长度不能超过 n 字节。
ngx_strchr | ngx_strchr(s->data, '/'); | 从地址中查找目标字符首次出现的地址。
ngx_strlchr | ngx_strlchr(start, last, '/'); | 在指定的地址范围内查找目标字符串首次出现的地址。
ngx_cpystrn | |
ngx_cpystrn | |
ngx_strcasecmp | |
ngx_strncasecmp | |
ngx_strlcasestrn | |
ngx_rstrncmp | |
ngx_rstrncasecmp | |
ngx_dns_strcmp | |
ngx_filename_cmp | |
ngx_escape_uri | |
ngx_unescape_uri | |
ngx_escape_html | |
ngx_escape_json | |

**注意：**

- 字符串处理相关的函数很多字符串参数都是 `const char *` 或者 `u_char *`，这并不代表无法处理 `ngx_str_t`。实际在使用中是将 `ngx_str_t.data` 取出后进行传参使用。

### 字符串转换相关

函数 | 示例 | 描述
-|-|-
[ngx_atoi](#ngx_atoi) | ngx_int_t n = ngx_atoi("123", 3); | 将指定地址的 n 个字符解析为 `ngx_int_t`。
ngx_atofp | ngx_atofp("10.5", 4, 2) | 将指定地址的 n 个字符解析为 `ngx_int_t`。支持小数，例如 ngx_atofp("10.5", 4, 2) 返回 1050。
[ngx_atosz](#ngx_atosz) | | 将指定地址的 n 个字符解析为 `ssize_t`。
[ngx_atoof](#ngx_atoof) | | 将指定地址的 n 个字符解析为 `off_t`。
[ngx_atotm](#ngx_atotm) | | 将指定地址的 n 个字符解析为 `time_t`。
[ngx_hextoi](#ngx_hextoi) | ngx_string("AA", 2); | 将指定地址的 n 个字符（十六进制表示）解析为 `ngx_int_t`。

### 字符串 Base 64

函数 | 示例 | 描述
-|-|-
ngx_base64_encoded_length | |
ngx_base64_decoded_length | |
ngx_encode_base64 | |
ngx_encode_base64url | |
ngx_decode_base64 | |
ngx_decode_base64url | |

### 字符串 UTF-8

函数 | 示例 | 描述
-|-|-
ngx_utf8_decode | |
ngx_utf8_length | |
ngx_utf8_cpystrn | |

### 格式化函数

函数 | 描述
-|-
[ngx_sprintf](#ngx_sprintf) | 将字符串格式化，并将结果存储在指定的地址空间，支持可变参数。
[ngx_snprintf](ngx_snprintf) | 将字符串格式化，并将结果存储在指定的地址空间，支持可变参数。通过 max 参数限制格式化的长度。
[ngx_slprintf](#ngx_slprintf) | 将字符串格式化，并将结果存储在指定的地址空间，支持可变参数。通过 last 参数限制格式化长度。
[ngx_vslprintf](#ngx_vslprintf) | 格式化的核心函数，避免直接调用。通过 last 参数限制格式化长度。
[ngx_vsnprintf](#ngx_vsnprintf) | 类似于 `ngx_vslprintf`，通过 max 参数限制格式化长度。

### 内存空间处理

函数 | 示例 | 描述
-|-|-
ngx_memzero | | 将指定地址的前 n 字节置为 0。
ngx_memset | | 将指定地址的前 n 字节置为固定的值 c。
ngx_explicit_memzero | |
ngx_memcpy | |
ngx_cpymem | |
ngx_copy | |
ngx_memmove | |
ngx_movemem | |
ngx_memcmp | |
ngx_memn2cmp | |
[ngx_hex_dump](#ngx_hex_dump) | | 将指定地址的数据转换为十六进制的方式进行存储。


## 函数

### ngx_atoi

将指定地址的 n 字节字符串转化为 `ngx_int_t`。只支持正整数。

**注意：**

- 最大值为 `NGX_MAX_INT_T_VALUE`。
- 仅为正整数。

```C
ngx_str_t s = ngx_string("123");
ngx_int_t n = ngx_atoi(s.data, s.len);
ngx_log_stderr(0, "number: %i", n);
```

### ngx_atosz

**注意：**

- 最大值为 `NGX_MAX_SIZE_T_VALUE`。
- 仅为正整数。

### ngx_atoof

**注意：**

- 最大值为  `NGX_MAX_OFF_T_VALUE`。
- 仅为正整数。

### ngx_atotm

**注意：**

- 最大值为 `NGX_MAX_TIME_T_VALUE`。
- 仅为正整数。

### ngx_hextoi

**注意：**

- 最大值为 `NGX_MAX_INT_T_VALUE`。
- 仅为正整数。

### ngx_hex_dump

将指定地址的数据转换为十六进制的方式进行存储。

函数声明：

```c
u_char *ngx_hex_dump(u_char *dst, u_char *src, size_t len)
```

输入参数 | 描述
-|-
dst | 将 dump 后的字符串存储的地址。
src | 需要进行 dump 的数据源。
len | src 的数据长度。

返回值就是传入的 dst 地址。

示例：

```c
u_char data[] = {38, 170, 255};

u_char buf[32] = {0};
ngx_hex_dump(buf, data, 3);

// 输出:
// dump: 26aaff
ngx_log_stderr(0, "dump: %s", buf);
```

### ngx_sprintf

将字符串格式化，并将结果存储在指定的地址空间，支持可变参数。

函数声明：

```c
u_char * ngx_cdecl ngx_sprintf(u_char *buf, const char *fmt, ...)
```

输入参数 | 描述
-|-
buf | 格式化后的字符串存储地址。
fmt | 格式化字符串。
... | 可变参数。

返回值就是传入的 buf 地址。

示例：

```c
ngx_str_t name = ngx_string("david");
ngx_int_t age = 18;

u_char buf[64] = {0};
ngx_sprintf(buf, "my name is %V, age is %ui", &name, age);

ngx_log_stderr(0, "message: %s", buf);
```

**注意：**

- 因为该函数并不明确格式化长度，调用可能会发生内存溢出，因此使用该函数有风险，不推荐。

### ngx_snprintf

将字符串格式化，并将结果存储在指定的地址空间，支持可变参数。限制了格式化长度。

函数声明：

```c
u_char * ngx_cdecl ngx_snprintf(u_char *buf, size_t max, const char *fmt, ...);
```

输入参数 | 描述
-|-
buf | 格式化后的字符串存储地址。
max | 是可以进行格式化的大小，通常取值是 buf 所对应存储空间的大小。
fmt | 格式化字符串。
... | 可变参数。

返回值就是传入的 buf 地址。

示例：

```c
ngx_str_t name = ngx_string("david");
ngx_int_t age = 18;

u_char buf[64] = {0};
ngx_snprintf(buf, 64, "my name is %V, age is %ui", &name, age);

ngx_log_stderr(0, "message: %s", buf);
```

### ngx_slprintf

将字符串格式化，并将结果存储在指定的地址空间，支持可变参数。限制了格式化长度。

函数声明：

```c
u_char * ngx_cdecl ngx_snprintf(u_char *buf, size_t max, const char *fmt, ...);
```

输入参数 | 描述
-|-
buf | 格式化后的字符串存储地址。
last | 是可以存储空间末尾，当格式化结构存储到该地址后，将停止格式化。
fmt | 格式化字符串。
... | 可变参数。

返回值就是传入的 buf 地址。

示例：

```c
ngx_str_t name = ngx_string("david");
ngx_int_t age = 18;

u_char buf[64] = {0};
ngx_slprintf(buf, buf + 64, "my name is %V, age is %ui", &name, age);

ngx_log_stderr(0, "message: %s", buf);
```

### ngx_vslprintf

格式化的核心函数，避免直接调用。通过 last 参数限制格式化长度。

函数声明：

```c
u_char * ngx_vslprintf(u_char *buf, u_char *last, const char *fmt, va_list args)
```

### ngx_vsnprintf

似于 `ngx_vslprintf`，通过 max 参数限制格式化长度。

这是一个宏：

```c
#define ngx_vsnprintf(buf, max, fmt, args)                                   \
    ngx_vslprintf(buf, buf + (max), fmt, args)
```

## 原理

### 数据结构

```c
typedef struct {
    size_t      len;            // 字符串长度
    u_char     *data;           // 字符串首地址
} ngx_str_t;
```
