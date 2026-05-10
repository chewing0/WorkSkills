# 计算机八股目录

> 这是所有后端 / 基础架构岗的通用底盘，AI Infra 岗考察权重约占 **30%–40%**。

---

## 1. 操作系统（OS）
- 1.1 进程与线程：区别、通信方式（IPC）、线程模型、协程
- 1.2 内存管理：虚拟内存、页表、TLB、缺页中断、内存映射（mmap）
- 1.3 调度算法：CFS、优先级反转、负载均衡
- 1.4 同步原语：Mutex、Spinlock、Semaphore、RCU、死锁条件与避免
- 1.5 文件系统：页缓存、IO 栈、ext4 vs xfs、零拷贝（sendfile、splice）
- 1.6 容器基础：Namespace（PID/Network/Mount/UTS/IPC/User/Cgroup）、cgroups v1/v2、UnionFS/OverlayFS

## 2. 计算机网络
- 2.1 TCP/UDP：三次握手、四次挥手、TIME_WAIT、滑动窗口、拥塞控制（CUBIC/BBR）、Nagle 算法
- 2.2 HTTP/HTTPS：1.1 vs 2.0 vs 3.0、TLS 握手、证书链、HSTS
- 2.3 DNS 与 CDN：解析流程、递归/迭代查询、负载均衡、边缘缓存
- 2.4 网络模型：OSI 七层、TCP/IP 四层、epoll/io_uring/select 对比、Reactor/Proactor
- 2.5 网络安全：TLS/SSL、中间人攻击、证书固定（Pinning）、防火墙与 iptables

## 3. 数据结构与算法
- 3.1 线性结构：数组、链表、栈、队列、双端队列、单调栈/单调队列
- 3.2 树：二叉树、BST、AVL、红黑树、B+/B- 树、Trie、线段树、树状数组
- 3.3 图：DFS/BFS、拓扑排序、最短路径（Dijkstra/Floyd）、最小生成树（Prim/Kruskal）、网络流
- 3.4 哈希：冲突解决（链地址法、开放定址）、一致性哈希、跳表
- 3.5 排序与查找：快排、归并、堆排、二分及变种、字符串匹配（KMP）
- 3.6 动态规划：背包、线性 DP、区间 DP、状态压缩、记忆化搜索

## 4. 数据库
- 4.1 关系型（MySQL/PostgreSQL）：索引（B+ 树/Hash）、事务 ACID、隔离级别、MVCC、锁（行锁/间隙锁/临键锁）、慢查询优化、分库分表、主从复制
- 4.2 NoSQL：Redis（数据结构、持久化 RDB/AOF、集群模式、哨兵、缓存穿透/击穿/雪崩）、MongoDB、ClickHouse
- 4.3 事务与一致性：CAP、BASE、2PC/3PC、TCC、Saga、最终一致性
- 4.4 SQL 优化：Explain 分析、索引覆盖、最左前缀、避免回表、Join 优化

## 5. 设计模式与软件工程
- 5.1 创建型：单例、工厂、抽象工厂、建造者、原型
- 5.2 结构型：代理、装饰器、适配器、桥接、组合、外观、享元
- 5.3 行为型：观察者、策略、模板方法、责任链、状态、命令、迭代器、访问者
- 5.4 架构模式：MVC、MVVM、微服务拆分、DDD 基础概念

## 6. 分布式系统
- 6.1 理论基础：CAP 定理、FLP 不可能性、一致性模型（线性/顺序/因果/最终）
- 6.2 共识算法：Paxos、Raft（选举/日志复制/安全性）、ZAB
- 6.3 分布式事务：TCC、Saga、本地消息表、Seata
- 6.4 负载均衡：轮询、加权、一致性哈希、最少连接、自适应
- 6.5 限流降级：令牌桶、漏桶、滑动窗口、熔断（Hystrix/Sentinel）
- 6.6 微服务：服务发现（Eureka/Nacos/Consul）、网关（Gateway）、配置中心、链路追踪（OpenTracing/SkyWalking）

## 7. 编程语言深入（Python / Go / Java 为例）
- 7.1 Python：GIL、垃圾回收（引用计数 + 分代回收）、元类、装饰器、协程（asyncio）、内存池
- 7.2 Go：GMP 调度模型、Channel 底层、GC（三色标记 + 混合写屏障）、内存逃逸分析、Interface 底层
- 7.3 Java：JVM 内存模型、GC 算法（CMS/G1/ZGC）、类加载机制、并发包（JUC）、锁升级（偏向/轻量/重量）

## 8. 系统设计题（经典场景）
- 8.1 经典场景：短链系统、秒杀系统、IM 系统、Feed 流、文件存储
- 8.2 AI 相关：大模型推理服务设计、向量数据库选型、Agent 平台架构

---

> **面试定位**：基础底盘，所有后端/基础架构岗通用。建议结合具体岗位调整复习权重。