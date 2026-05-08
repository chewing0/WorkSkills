## 认识Redis
### 什么是Redis

## Redis线程模式
### Redis是单线程吗？

Redis 不是完全单线程；它的核心命令执行模型主要是单线程，但现代 Redis 会使用后台线程、I/O 线程和子进程处理网络、磁盘、持久化、异步释放等辅助工作。
Redis 官方文档说明，Redis 使用 mostly single threaded design，一个进程通过 multiplexing 服务客户端请求，客户端请求是顺序处理的。
Redis单线程还这么快的原因是Redis大部分操作都在内存中完成，并且不用给核心数据结构加很多锁。
Redis 2.4 开始，Redis 就使用线程在后台执行一些较慢的 I/O 操作，主要和磁盘 I/O 有关；从 Redis 6.0 开始，I/O threading 会把网络读写工作交给后台线程；Redis 8 也继续改进了 I/O threading，可以通过 `io-threads` 配置启用更多线程来提升吞吐。
问题：如果某个命令执行很慢，后面的请求就要等待，有可能阻碍主线程。