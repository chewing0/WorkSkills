## 认识Redis
### 什么是Redis

## Redis线程模式
### Redis是单线程吗？

Redis 不是完全单线程；它的核心命令执行模型主要是单线程，但现代 Redis 会使用后台线程、I/O 线程和子进程处理网络、磁盘、持久化、异步释放等辅助工作。
Redis 官方文档说明，Redis 使用 mostly single threaded design，一个进程通过 multiplexing 服务客户端请求，客户端请求是顺序处理的。
### Redis单线程还这么快

Redis大部分操作都在内存中完成，并且不用给核心数据结构加很多锁，使用I/O多路复用处理大量连接。
### Redis为什么后续又引入了多线程？

Redis 2.4 开始，Redis 就使用线程在后台执行一些较慢的 I/O 操作，主要和磁盘 I/O 有关；从 Redis 6.0 开始，I/O threading 会把网络读写工作交给后台线程；Redis 8 也继续改进了 I/O threading，可以通过 `io-threads` 配置启用更多线程来提升吞吐。
```
Redis-server：主线程，主要负责执行命令；
bio_close_file、bio_aof_fsync、bio_lazy_free：三个后台线程，分别异步处理关闭文件任务、AOF刷盘任务、释放内存任务；
io_thd_1、io_thd_2、io_thd_3：三个 I/O 线程，io-threads 默认是 4 ，所以会启动 3（4-1）个 I/O 多线程，用来分担 Redis 网络 I/O 的压力。
```
**问题：** 如果某个命令执行很慢，后面的请求就要等待，有可能阻碍主线程。一个请求很慢时，其他客户端会等待这个请求完成；因此要注意大集合上的慢命令，并查看命令复杂度。

## Redis持久化
### Redis怎么实现数据不丢失

Redis 的读写操作都是在内存中，所以 Redis 性能才会高，但是当 Redis 重启后，内存中的数据就会丢失，那为了保证内存中的数据不会丢失，Redis 实现了数据持久化的机制，这个机制会把数据存储到磁盘，这样在 Redis 重启就能够从磁盘中恢复原有的数据。
**Redis一共有三种方式：AOF、RDB、混合模式**

### AOF日志是如何实现的？

AOF，全称 **Append Only File**，中文一般叫**追加日志文件**。AOF文件记录的是写操作命令，只要Redis重新执行这些命令就可以恢复。

#### 为什么先执行命令，再写入日志？


