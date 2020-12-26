# Nginx 模块开发教程

## 目录

- [概览](overview.md)
- [快速开始](quick-start.md)
- [日志](log.md)
- [基本数据结构](base-data-structure.md)
- [容器](containers/overview.md)
  - [字符串](containers/string.md)
  - [内存池](containers/pool.md)
  - [数组](containers/array.md)
  - [链表](containers/list.md)
  - [缓冲区](containers/buf.md)
  - [队列](containers/queue.md)
  - [哈希表](containers/hash.md)
  - [红黑树](containers/rbt.md)
  - [共享内存](containers/share-memory.md)
- HTTP 模块开发
  - 概览
  - Nginx HTTP 阶段
  - HTTP 模块开发结构
  - HTTP 模块配置文件
  - HTTP 请求
    - 获得请求行
    - 获得请求头
    - 获得请求查询字符串
    - 获得请求体
  - HTTP 响应
    - 设置响应头
    - 设置响应 Cookie
    - 设置响应体
    - 重定向响应
  - 子请求
  - HTTP 过滤模块
- Nginx 变量
- 事件
  - 定时任务
- 进程通信
  - 共享内存
  - 锁
  - Nginx 频道
  - 信号
- 连接管理
  - [套接字](connections/socket.md)
  - [监听连接](connections/listen.md)

## 附录、参考文献

- [Nginx Development Guide](https://nginx.org/en/docs/dev/development_guide.html)
- [Niginx Directives](https://nginx.org/en/docs/dirindex.html)
- [Niginx Variables](https://nginx.org/en/docs/varindex.html)
- [Nginx-1.19.4](https://nginx.org/en/download.html)
- [深入理解 Nginx 模块开发与架构解析](https://book.douban.com/subject/26745255/)
- [Agentzh 的 Nginx 教程](https://openresty.net.cn/agentzh-nginx-guide.html)