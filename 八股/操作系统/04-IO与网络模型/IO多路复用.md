---
tags:
  - 八股
  - 操作系统
  - IO模型
  - epoll
status: 待复习
created: 2026-05-06
---

# IO多路复用

> [!tip] 核心速记
> **"红黑树管全连接，就绪链表只返活，LT反复催你拿，ET只说一次话。"**
> 一个服务员（线程）同时服务10桌客人——epoll让服务员只去有需求的桌。

---

## 1. 核心场景

> **比喻**：一个餐厅服务员（一个线程）要同时服务 10 桌客人（10 个 socket 连接）。
>
> **没有 IO 多路复用之前**：
> - **BIO（阻塞IO）**：服务员站在 1 号桌前，等客人点菜（阻塞），其他 9 桌干等着。一个线程只能服务一个连接。
>
> **有了 IO 多路复用**：
> - 服务员注册到"餐厅总管"（内核的 epoll），哪桌有需求（数据就绪），总管通知服务员去处理。一个服务员可以高效服务全部 10 桌！

---

## 2. BIO / NIO / AIO 完整对比

| 维度 | BIO（同步阻塞） | NIO（同步非阻塞） | AIO（异步非阻塞） |
|------|-----------------|-------------------|-------------------|
| **模型** | 一个连接一个线程 | 一个线程处理多个连接（Selector轮询） | 内核完成IO后通知应用（回调） |
| **线程行为** | read()阻塞等待数据到达 | read()立即返回（有数据就读，没数据返回0） | read()立即返回，内核异步完成IO |
| **数据就绪时** | 内核拷贝数据到用户空间，线程阻塞在拷贝阶段 | 应用线程自己拷贝数据（同步） | 内核自动拷贝数据到用户空间，通知应用 |
| **适用场景** | 连接数少、固定 | 连接数多、短请求 | 连接数多、长请求 |
| **Java对应** | `java.io`（传统IO） | `java.nio`（Selector+Channel） | `java.nio` AsynchronousChannel（Windows用IOCP） |
| **典型应用** | 传统Tomcat BIO模式 | Netty、Tomcat NIO模式 | Windows IOCP（Java AIO在Linux上仍是epoll模拟） |

> [!warning] 面试关键区分
> - **同步 vs 异步**：关注的是"数据拷贝谁来完成"——同步=应用线程自己做，异步=内核做完通知应用
> - **阻塞 vs 非阻塞**：关注的是"数据未就绪时线程是否挂起"——阻塞=线程等待，非阻塞=线程继续做其他事
> - NIO 是**同步非阻塞**：数据拷贝仍由应用线程完成（同步），但数据未就绪时线程不阻塞（非阻塞）
> - AIO 是**异步非阻塞**：数据拷贝由内核完成，应用线程完全不参与

---

## 3. select / poll / epoll 对比

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| **数据结构** | 固定长度 `fd_set` 位图 | 动态数组 `pollfd` | 红黑树 + 就绪链表 |
| **最大连接数** | 1024（硬限制） | 无限制 | 无限制（数十万） |
| **时间复杂度** | O(n)，每次全量轮询 | O(n)，每次全量轮询 | O(1)，只返回就绪的 |
| **内核态→用户态拷贝** | 全量拷贝所有 fd | 全量拷贝所有 fd | 只拷贝就绪的 fd |
| **触发模式** | 仅水平触发 | 仅水平触发 | **LT（水平触发）+ ET（边缘触发）** |
| **适用场景** | 连接数少 | 连接数少 | **高并发（C10K/C100K）** |

> [!note] epoll 为什么快？三句话总结
> 1. **数据结构**：红黑树管理所有连接（增删O(log n)），就绪链表只存有事件的fd（返回O(1)）
> 2. **减少拷贝**：只拷贝就绪的fd从内核态到用户态，不用全量拷贝
> 3. **事件驱动**：不需要遍历所有fd，内核通过中断回调主动通知

---

## 4. epoll 的 LT vs ET

> **比喻**：快递到了，物业通知你。
>
> **LT（水平触发）** = 物业一直打电话催你拿快递，直到你取走为止。你不取他就一直打。
> - 优点：不丢事件，编程简单
> - 缺点：可能重复通知
>
> **ET（边缘触发）** = 物业只打一次电话通知，然后不再管你。你不取是你的问题。
> - 优点：减少通知次数，高效
> - 缺点：必须一次读完数据（非阻塞IO + 循环 read），否则数据会丢

```c
// LT模式：每次epoll_wait都会返回就绪的fd，只要缓冲区有数据
// ET模式：只在状态变化时返回一次（无数据→有数据），必须循环读到EAGAIN
while (1) {
    n = read(fd, buf, sizeof(buf));     // 非阻塞读
    if (n == -1 && errno == EAGAIN)
        break;  // 读完
}
```

> [!warning] ET 模式编程要求
> ET 模式必须使用**非阻塞IO** + **循环读写直到EAGAIN**，否则：
> - 只读了一部分数据，后续数据不会被再次通知（事件丢失）
> - 只写了一部分数据，后续写缓冲区满了不会再次通知（写挂起）

---

## 5. Reactor 模式 vs Proactor 模式

| 模式 | 核心 | 类比 | 实现代表 |
|------|------|------|----------|
| **Reactor** | 事件就绪时通知应用，应用自己完成IO操作 | 物业告诉你快递到了，你自己去取 | Netty、Redis、Nginx（epoll） |
| **Proactor** | 内核完成IO操作后通知应用，应用直接处理结果 | 物业把快递送到你家门口，你直接拆 | Windows IOCP、io_uring |

### 5.1 Reactor 三种模式

| 模式 | 说明 | 适用 |
|------|------|------|
| **单Reactor单线程** | 一个线程处理所有连接+业务逻辑 | Redis（单线程版） |
| **单Reactor多线程** | 一个线程处理IO，业务逻辑交给线程池 | 适合业务逻辑耗时的场景 |
| **主从Reactor多线程** | 主Reactor接收连接，从Reactor处理IO，线程池处理业务 | Netty（主从Reactor模型） |

> [!note] Netty 的 Reactor 模型
> - **Boss Group**（主Reactor）：负责 Accept 新连接
> - **Worker Group**（从Reactor）：负责已建立连接的IO读写
> - 每个Group包含多个EventLoop（每个EventLoop = 一个线程 + 一个Selector）

---

## 6. io_uring 简介

### 6.1 什么是 io_uring？

io_uring = Linux 5.1+ 引入的新异步IO框架，核心思想：**共享环形缓冲区 + 内核主动轮询**。

```
传统epoll流程：
  应用提交IO请求 → 系统调用 → 内核处理 → 内核通知应用 → 应用再系统调用读取数据
  （2次系统调用）

io_uring流程：
  应用将IO请求写入共享环形缓冲区（Submission Queue）→ 内核轮询SQ消费请求
  → 内核完成后将结果写入完成环形缓冲区（Completion Queue）→ 应用从CQ读取结果
  （0次系统调用！）
```

### 6.2 io_uring vs epoll

| 维度 | epoll | io_uring |
|------|-------|----------|
| **系统调用次数** | 每次 epoll_wait + read/write = 多次 | SQ/CQ共享内存，0次系统调用 |
| **数据拷贝** | 就绪fd需要内核→用户态拷贝 | 共享内存，无拷贝 |
| **IO模型** | 事件通知型（需要应用自己做IO） | 真异步型（内核完成IO，通知应用） |
| **内核版本** | Linux 2.5.44+ | Linux 5.1+ |
| **编程复杂度** | 相对简单 | 较复杂（需要理解SQ/CQ模型） |

> [!note] io_uring 两大特性
> 1. **Submission Queue (SQ)** + **Completion Queue (CQ)**：两个环形缓冲区通过 mmap 共享，应用写SQ、内核写CQ，零拷贝交互
> 2. **内核轮询 (IORING_SETUP_SQPOLL)**：内核创建专门线程轮询SQ，应用无需调用 `io_uring_enter()` 系统调用——真正的"零系统调用"

---

## 7. Java NIO 的 Selector 和 Channel 原理

### 7.1 Java NIO 三大核心

| 组件 | 作用 | 类比 |
|------|------|------|
| **Selector** | 多路复用器，管理多个Channel的IO事件 | 餐厅总管——监视所有桌子 |
| **Channel** | 双向数据通道（可读可写） | 每桌的通信管道 |
| **Buffer** | 数据容器（读写都通过Buffer） | 服务员的托盘——数据先装在托盘上再传输 |

### 7.2 工作流程

```java
// Java NIO 核心代码
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.bind(new InetSocketAddress(8080));
serverChannel.register(selector, SelectionKey.OP_ACCEPT); // 注册Accept事件

while (true) {
    selector.select(); // 阻塞等待事件（底层调用epoll_wait）
    Set<SelectionKey> keys = selector.selectedKeys();
    for (SelectionKey key : keys) {
        if (key.isAcceptable()) {
            // 接收新连接
            SocketChannel client = serverChannel.accept();
            client.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            // 读取数据
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            client.read(buffer);
        }
    }
    keys.clear();
}
```

> [!note] Java NIO 底层实现
> - Linux 上：`EpollSelectorProvider` → 使用 epoll
> - Mac 上：`KQueueSelectorProvider` → 使用 kqueue
> - Windows 上：`WindowsSelectorProvider` → 使用 select/WSAPoll
> - `selector.select()` 底层就是 `epoll_wait()` / `kevent()` 调用

---

## 8. 和 Java / 后端框架的关联

| 技术 | 使用的IO模型 | 说明 |
|------|-------------|------|
| **Netty** | epoll（Linux）/ kqueue（Mac） | Java NIO 底层可以通过 `EpollSelectorProvider` 使用 epoll |
| **Redis** | epoll（单线程） | Redis 6.0 前单线程用 epoll 处理所有连接，性能天花板 |
| **Nginx** | epoll（多进程） | master 进程 + 多个 worker 进程，每个 worker 独立用 epoll |
| **Tomcat NIO** | Java NIO Selector | 基于 epoll（Linux）实现非阻塞IO |
| **Kafka** | Java NIO | Broker 使用 Selector 处理生产者和消费者的连接 |

---

## 面试题速查

> [!example] 本模块高频面试题

1. **select/poll/epoll的区别？epoll为什么快？**
   - 红黑树+就绪链表+只拷贝就绪fd → O(1) vs O(n)

2. **LT vs ET 模式各是什么？编程有什么区别？**
   - LT反复通知，ET只通知一次；ET必须非阻塞循环读写到EAGAIN

3. **BIO/NIO/AIO完整对比？**
   - BIO=同步阻塞，NIO=同步非阻塞，AIO=异步非阻塞

4. **Reactor vs Proactor模式？Netty用的是哪个？**
   - Reactor=事件就绪通知应用自己做IO；Proactor=内核完成IO通知应用；Netty用主从Reactor

5. **io_uring是什么？为什么比epoll更高效？**
   - 共享环形缓冲区+内核轮询→0次系统调用；epoll每次select/read都是系统调用

6. **Java NIO的Selector和Channel原理？底层用什么？**
   - Selector管理多个Channel事件，Channel双向数据通道；Linux上用epoll

7. **IO多路复用在Redis/Nginx/Netty中分别是怎么用的？**
   - Redis单线程+epoll，Nginx多进程每个worker用epoll，Netty主从Reactor+epoll

---

## 交叉链接

- [[零拷贝|零拷贝]] — 同在IO模块，减少数据拷贝的另一条路
- [[01-进程管理/进程与线程|进程与线程]] — 系统调用开销、用户态/内核态切换
- [[03-内存管理/内存管理|内存管理]] — mmap也是一种减少拷贝的IO方式
- [[../操作系统总览|操作系统总览]] — 回到总览