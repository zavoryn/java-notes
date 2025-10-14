---
tags: [八股, java基础, IO, NIO, BIO]
status: 已完成
---

# Java IO 流概述

> [!summary] 我的速记
> **字节流**（InputStream/OutputStream）vs **字符流**（Reader/Writer），字符流=字节流+编码。IO 体系是**装饰器模式**经典案例。
> **BIO/NIO/AIO：** BIO 阻塞一线一连 → NIO 非阻塞 Channel+Buffer+Selector 一线管多连 → AIO 异步回调。
> **NIO 三大件：** Channel（双向通道）、Buffer（数据容器，读写需 flip）、Selector（多路复用器）。

## 1. 按流向分类

| 分类 | 说明 | 典型类 |
|------|------|--------|
| **输入流** | 从数据源读取数据到程序 | `InputStream` / `Reader` |
| **输出流** | 从程序写入数据到目标 | `OutputStream` / `Writer` |

## 2. 按数据单位分类

| 分类 | 抽象基类 | 典型子类 | 说明 |
|------|---------|---------|------|
| **字节流** | `InputStream` / `OutputStream` | `FileInputStream`, `FileOutputStream` | 操作 byte，适合所有文件（文本/图片/视频） |
| **字符流** | `Reader` / `Writer` | `FileReader`, `FileWriter` | 操作 char，适合纯文本文件 |

> **为什么要字符流？** 字节流按 byte 读，遇到中文等多字节字符会乱码。字符流 = 字节流 + 编码表，能正确处理字符编码。

## 3. 常用类

### 字节流

```java
// 基础字节流
try (FileInputStream fis = new FileInputStream("input.txt");
     FileOutputStream fos = new FileOutputStream("output.txt")) {
    byte[] buf = new byte[1024];
    int len;
    while ((len = fis.read(buf)) != -1) {
        fos.write(buf, 0, len);
    }
}

// 缓冲字节流（提高效率，内部维护 8KB 缓冲区）
try (BufferedInputStream bis = new BufferedInputStream(
         new FileInputStream("input.txt"))) {
    // bis.read() 从缓冲区读，减少磁盘 IO 次数
}
```

### 字符流

```java
// 字符流 = 字节流 + 编码
try (BufferedReader br = new BufferedReader(
         new FileReader("input.txt"))) {  // FileReader 默认使用系统编码
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}

try (BufferedWriter bw = new BufferedWriter(
         new FileWriter("output.txt"))) {
    bw.write("Hello World");
    bw.newLine();  // 跨平台换行
}
```

## 4. 装饰器模式在 IO 中的体现

```java
// BufferedReader 装饰了 FileReader
BufferedReader br = new BufferedReader(new FileReader("file.txt"));

// BufferedInputStream 装饰了 FileInputStream
BufferedInputStream bis = new BufferedInputStream(
    new FileInputStream("file.bin"));
```

> IO 流的设计是**装饰器模式**的经典案例。节点流（如 `FileInputStream`）直接操作数据源，过滤流/处理流（如 `BufferedInputStream`）包装节点流添加额外功能（缓冲、类型转换等），各层职责单一，可灵活组合。

```
FileInputStream (节点流：直接读文件)
    → BufferedInputStream (装饰：加缓冲)
        → DataInputStream (装饰：提供 readInt 等)
```

## 5. BIO vs NIO vs AIO 概念对比

这是面试中**最高频**的 IO 模型考点。

| 对比维度 | BIO | NIO | AIO（NIO.2） |
|----------|-----|-----|-------------|
| 全称 | Blocking IO | Non-blocking IO (New IO) | Asynchronous IO |
| 模型 | **同步阻塞** | **同步非阻塞** | **异步非阻塞** |
| 线程模型 | 一个连接一个线程 | 一个线程处理多个连接 | 回调机制，OS 通知 |
| 核心组件 | `ServerSocket` + `Socket` | `Channel` + `Buffer` + `Selector` | `AsynchronousSocketChannel` |
| 发起时机 | JDK 1.0 | JDK 1.4 | JDK 7 |
| 适用场景 | 连接数少、架构简单 | 连接数多、高并发（如 Netty） | 连接数多、耗时操作（如文件读写） |
| 内核支持 | — | `select/poll/epoll` | OS 异步 IO（如 Windows IOCP） |

### BIO 模型

```java
// 一个连接一个线程，阻塞等待
ServerSocket server = new ServerSocket(8080);
while (true) {
    Socket socket = server.accept();   // 阻塞等待连接
    new Thread(() -> {
        // 每个连接分配一个线程处理
        InputStream in = socket.getInputStream();
        // read() 阻塞等待数据
    }).start();
}
```

### NIO 核心三大件

```java
// Selector：单线程管理多个 Channel，轮询就绪事件
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);  // 设置为非阻塞
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();  // 阻塞直到有就绪事件
    Set<SelectionKey> keys = selector.selectedKeys();
    for (SelectionKey key : keys) {
        if (key.isAcceptable())   { /* 处理新连接 */ }
        if (key.isReadable())     { /* 从 Channel 读数据到 Buffer */ }
    }
    keys.clear();
}
```

| 组件 | 作用 | 类比 |
|------|------|------|
| **Channel** (通道) | 双向读写通道，可同时读写 | 高速公路 |
| **Buffer** (缓冲区) | 数据容器，所有数据通过 Buffer 传输 | 卡车 |
| **Selector** (选择器) | 多路复用器，一个线程监听多个 Channel 事件 | 收费站调度 |

> NIO 核心思想：用**一个线程**通过 **Selector 多路复用**，同时监听**多个 Channel** 的就绪事件，减少线程开销。这也是 **Netty** 和 **Tomcat NIO 模式**的底层原理。

### AIO 模型

```java
// 异步回调，完全不阻塞
AsynchronousSocketChannel client = AsynchronousSocketChannel.open();
client.connect(address, null, new CompletionHandler<Void, Object>() {
    @Override
    public void completed(Void result, Object attachment) {
        // 连接成功后异步读取
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        client.read(buffer, null, new CompletionHandler<Integer, Object>() {
            @Override
            public void completed(Integer bytesRead, Object attachment) {
                // 数据读取完成，处理数据
            }
            // ...
        });
    }
    // ...
});
```

## 6. 面试话术

> "Java IO 体系按数据单位分为字节流（InputStream/OutputStream）和字符流（Reader/Writer），字符流本质是字节流+编码表。按功能分为节点流和处理流，整个 IO 体系是装饰器模式的经典应用——通过层层包装给节点流添加缓冲、转换等功能。"

> "面试中最常被问的是 BIO/NIO/AIO 三者的区别。BIO 是同步阻塞模型，一个连接对应一个线程，简单但无法支撑高并发。NIO 是同步非阻塞模型，通过 Channel + Buffer + Selector 三大核心组件，让一个线程可以管理多个连接，适合高并发场景。AIO 是异步非阻塞模型，基于回调机制，操作系统完成后通知应用，适合大文件读写等耗时操作。Netty 框架就是基于 NIO 实现的，是当前最主流的高性能网络通信框架。"

> "NIO 的 Buffer 和传统 IO 的 Stream 有本质区别：Stream 是单向的，只能读或写；Buffer 是双向的。Buffer 有三种模式：写模式→`flip()`→读模式→`clear()`→写模式。读取时必须 `flip()` 切换模式，这是 NIO 编程的典型坑。"

## 7. 记忆口诀

> **BIO 阻塞一个连一个，NIO 复用一线管多个，AIO 异步回调不啰嗦。**

> **字节字符两分流，装饰模式包着走。读要 Buffer 加 flip，NIO 三件套 Chan Sel Buf。**
