---
tags:
  - 八股
  - 操作系统
  - Linux
  - 文件系统
status: 待复习
created: 2026-05-06
---

# 文件系统与Linux

> [!tip] 核心速记
> **inode = 图书馆管理卡片，硬链同inode不能跨区，软链独立号可跨区。**
> **rwx + SUID/SGID/Sticky = Linux权限三剑客。**
> **线上排查：top找CPU → jmap找内存 → lsof找文件 → ss找连接。**

---

## 1. inode（索引节点）

> **比喻**：图书馆的管理卡片。
>
> 每本书（文件）都有一张管理卡片（inode），卡片上记录了：
> - 书叫什么名字？—— 文件名（其实文件名在目录项里）
> - 书是哪个出版社的？—— 所有者、权限
> - 书在哪里（哪个书架第几排）？—— 数据块指针
> - 书有多大、什么时候入库的？—— 大小、时间戳
>
> **关键**：卡片的编号（inode号）才是书的唯一标识。书名只是"入口"，一本书可以有多个书名（硬链接），但卡片只有一张。

### 1.1 inode 包含的元数据

```
inode 存储的信息：
  ├── 文件类型和权限（rwx）
  ├── 所有者（UID）和所属组（GID）
  ├── 文件大小（字节数）
  ├── 时间戳（atime: 最后访问, mtime: 最后修改, ctime: 最后状态改变）
  ├── 链接数（硬链接计数）
  └── 数据块指针（直接指针 + 间接指针 + 双重间接 + 三重间接）
```

> [!note] inode 号才是文件的"身份证"
> - 文件名不在inode中，而是在目录项（dentry）中
> - `ls -i` 可以查看文件的inode号
> - inode数量有限（`df -i` 查看），耗尽后即使磁盘有空间也无法创建新文件

---

## 2. 硬链接 vs 软链接

| 特性 | 硬链接 (Hard Link) | 软链接/符号链接 (Soft Link/Symlink) |
|------|-------------------|-----------------------------------|
| **本质** | 同一个 inode 的多个目录项 | 一个独立文件，内容是指向目标路径的字符串 |
| **inode** | 与原文件**共享**同一个 inode | 有自己**独立的** inode |
| **跨文件系统** | **不可以**（inode 号只在同一文件系统有效） | **可以** |
| **删除原文件** | **不影响**（inode还在，链接数-1，归0才删除） | **失效**（变成"死链接"） |
| **目录链接** | **不支持**（防止循环） | **支持** |
| **类比** | 图书馆有好几个门，每个门上都有书名标签，但指向同一张管理卡片 | 图书馆的"引导牌"："《XXX》请往A区查看"，牌子本身不是书 |

```bash
# 创建硬链接
ln source.txt hardlink.txt
# 创建软链接
ln -s source.txt softlink.txt
# 查看inode号和硬链接计数
ls -li  # 第一列是inode号，第三列是链接数
```

> [!warning] 面试易错点
> 删除原文件后硬链接还能访问——因为inode的链接计数减1但不为0，数据块不会释放。
> 删除原文件后软链接失效——因为软链接指向的是一个路径字符串，原路径不存在了。

---

## 3. 和日常开发的关联

- **Docker 镜像分层**：底层利用 UnionFS（Overlay2），多层文件系统叠加。每层的"白名单/黑名单"就是利用类似硬链接/软链接的文件引用机制。
- **删除文件磁盘空间不释放**：因为还有进程持有该文件的 fd（文件描述符），inode 引用计数不为0。用 `lsof | grep deleted` 排查。
- **日志轮转**：`mv app.log app.log.1` 后空间没释放？因为 Java 进程还在往原 inode 写，需要重新打开日志文件或发信号让进程重载。

---

## 4. Linux 文件权限详解

### 4.1 基本权限 rwx

| 符号 | 含义 | 数字 |
|------|------|------|
| r | 读（read） | 4 |
| w | 写（write） | 2 |
| x | 执行（execute） | 1 |

```
权限格式：-rwxrwxrwx
  第1位：文件类型（-普通文件, d目录, l软链接）
  第2-4位：所有者权限（Owner）
  第5-7位：所属组权限（Group）
  第8-10位：其他人权限（Others）

举例：-rwxr-xr-- (754)
  → Owner: rwx(读+写+执行)
  → Group: r-x(读+执行，不能写)
  → Others: r--(只读)
```

### 4.2 SUID / SGID / Sticky Bit

| 特殊权限 | 名称 | 作用 | 数字 | 典型例子 |
|----------|------|------|------|----------|
| **SUID** | Set User ID | 执行时以**文件所有者**身份运行 | 4 | `/usr/bin/passwd`：普通用户执行passwd时，临时获得root权限修改密码文件 |
| **SGID** | Set Group ID | 执行时以**文件所属组**身份运行；对目录：新文件继承目录的组 | 2 | `/usr/bin/wall`；共享目录下新文件自动继承目录组 |
| **Sticky Bit** | 粘滞位 | 对目录：只有文件所有者才能删除该目录下的文件 | 1 | `/tmp`：任何人都能在/tmp下创建文件，但只有文件主人能删除 |

> [!note] 为什么 /tmp 要设 Sticky Bit？
> `/tmp` 权限是 `drwxrwxrwt`（777 + Sticky Bit）
> - 没有Sticky Bit：A在/tmp创建了文件，B可以删除A的文件（因为B对/tmp有w权限）
> - 有Sticky Bit：B不能删除A的文件，只有A自己和root能删

```bash
# 设置SUID
chmod 4755 file  # -rwsr-xr-x
# 设置Sticky Bit
chmod 1777 dir   # drwxrwxrwt
```

---

## 5. /proc 文件系统

/proc = Linux 内核提供的虚拟文件系统，不占用磁盘空间，内容是内核运行时数据。

| 文件 | 内容 | 用途 |
|------|------|------|
| `/proc/meminfo` | 内存使用详情 | `free -h` 的底层来源 |
| `/proc/cpuinfo` | CPU信息（型号、核数、频率） | 查看CPU规格 |
| `/proc/loadavg` | 系统负载（Load Average） | 监控系统繁忙程度 |
| `/proc/<pid>/status` | 进程状态（内存、线程数等） | 查看特定进程信息 |
| `/proc/<pid>/fd/` | 进程打开的文件描述符列表 | `lsof -p <pid>` 的底层来源 |
| `/proc/<pid>/maps` | 进程的内存映射区域 | 查看进程的虚拟地址空间分布 |
| `/proc/net/tcp` | TCP连接列表 | `ss -antp` 的底层来源 |
| `/proc/sys/` | 可调整的内核参数 | `sysctl` 命令的底层来源 |

```bash
# 查看内存信息
cat /proc/meminfo | head -10
# 查看CPU核数
cat /proc/cpuinfo | grep "processor" | wc -l
# 查看系统负载
cat /proc/loadavg
```

---

## 6. 常见文件系统对比

| 特性 | ext4 | xfs | btrfs |
|------|------|-----|-------|
| **最大文件大小** | 16TB | 8EB | 16EB |
| **最大文件系统大小** | 1EB | 8EB | 16EB |
| **日志模式** | 三种（journal/order/writeback） | 元数据日志 | 写时复制（COW）日志 |
| **快照** | 不支持 | 不支持 | **支持**（COW天然支持快照） |
| **在线扩缩容** | 支持（离线扩容更安全） | 支持（在线扩容更安全） | 支持 |
| **数据校验** | 不支持 | 不支持 | **支持**（checksum校验数据完整性） |
| **子卷** | 不支持 | 不支持 | **支持**（类似子目录但独立管理） |
| **适用场景** | 通用（Linux默认） | 大文件/高并发IO（RedHat默认） | 需要快照/校验/压缩的场景 |

> [!note] 为什么 RedHat/CentOS 7+ 默认用 xfs？
> - xfs 在大文件和高并发IO场景下性能优于 ext4
> - xfs 的分配组（Allocation Group）设计支持多线程并行IO
> - ext4 在删除大文件时性能较差

---

## 7. 常见线上故障排查场景

### 7.1 OOM Killer 排查

```
Linux内存不足 → OOM Killer选择一个进程杀掉 → 选择依据：oom_score（越高越容易被杀）

排查步骤：
  1. dmesg -T | grep -i oom → 查看OOM日志，找到被杀的进程
  2. cat /proc/<pid>/oom_score → 查看进程的oom评分
  3. echo -17 > /proc/<pid>/oom_adj → 设置进程不被OOM Killer杀（-17=禁止）

JVM被OOM Killer误杀的原因：
  → JVM堆内存很大（-Xmx设了8G），但实际可能只用了3G
  → Linux按虚拟内存算oom_score，JVM的虚拟内存=堆+元空间+直接内存+线程栈
  → 解决：降低-Xmx，或设置oom_adj=-17保护JVM进程
```

### 7.2 磁盘空间不释放

```
现象：rm删除了大文件，df -h 显示空间没释放

原因：进程还在持有已删除文件的fd → inode引用计数不为0 → 数据块不释放

排查：
  1. lsof | grep deleted → 找到持有已删除文件fd的进程
  2. kill进程 或 让进程重新打开日志文件
  3. 或者 > /proc/<pid>/fd/<fd_no> → 清空文件内容（不删文件）

Java日志轮转问题：
  → mv app.log app.log.1 → Java进程还往原inode写
  → 解决：使用logback/log4j的RollingFileAppender（自动创建新文件写新inode）
```

### 7.3 服务端 CPU 飙高排查

```bash
# 步骤：
top → 找到PID → top -Hp PID → 找到线程TID
→ printf "%x\n" TID → 转为16进制 → jstack PID | grep -A 30 <16进制TID>
→ 分析线程栈：是GC线程？业务线程？阻塞线程？
```

### 7.4 内存飙高排查

```bash
free -h → 确认是物理内存还是swap问题
→ jmap -histo:live PID | head -20 → 统计对象数量
→ jmap -dump:live,format=b,file=heap.hprof PID → 导出堆dump
→ MAT分析heap.hprof → 找到大对象/泄漏点
```

---

## 8. Linux 常用命令速查

### 8.1 系统资源监控

```bash
# CPU和内存
top          # 实时系统监控，按1看各CPU，按M按内存排序，按P按CPU排序
htop         # top的彩色增强版（需安装）
free -h      # 内存使用情况（-h 人类可读）
vmstat 1     # 每秒输出系统状态（r:运行队列, b:阻塞, cs:上下文切换）

# 磁盘
df -h        # 磁盘分区使用情况
du -sh *     # 当前目录下各文件/文件夹大小
iostat -x 1  # 磁盘IO监控（await: IO等待时间, util: 使用率）

# 网络
netstat -antp    # 所有TCP连接（LISTEN/ESTABLISHED/TIME_WAIT）
ss -antp         # netstat的替代品，更快（推荐）
sar -n DEV 1     # 网络流量统计
```

### 8.2 进程管理

```bash
ps aux | grep java              # 查Java进程
ps -eo pid,ppid,cmd,%mem,%cpu  # 按格式显示进程
kill -9 <PID>                   # 强制杀进程（-9 SIGKILL，不给清理机会）
kill -15 <PID>                  # 优雅停进程（SIGTERM，推荐先试试）
jps -l                          # 查看Java进程（JDK自带）
```

### 8.3 文件排查

```bash
lsof -p <PID>           # 列出进程打开的所有文件（包括socket、pipe）
lsof -i :8080           # 查看8080端口被谁占用
lsof /path/to/file      # 查看文件被哪些进程使用
tail -f app.log         # 动态追踪日志
tail -n 100 app.log     # 看最后100行
grep "ERROR" app.log    # 搜索错误日志
less app.log            # 大文件查看（不一次性加载到内存）
```

### 8.4 调优排查场景速查

| 场景 | 命令组合 |
|------|---------|
| 服务CPU飙高 | `top` → 找到PID → `top -Hp PID` → `jstack PID \| grep -A 10 线程ID(16进制)` |
| 内存飙高 | `free -h` → `jmap -histo:live PID \| head -20`（统计对象数） |
| 磁盘满了 | `df -h` → `du -sh /*` 逐层定位大文件 |
| 端口被占用 | `lsof -i :8080` 或 `ss -antp \| grep 8080` |
| 网络连接异常 | `ss -s`（统计连接状态）→ `ss -antp` 看具体连接 |
| OOM排查 | `dmesg -T \| grep -i oom` → 查看系统日志，找到被杀进程 |
| GC问题 | `jstat -gcutil PID 1000`（每秒输出GC情况） |

---

## 9. strace / gdb / perf 基本使用

### 9.1 strace — 跟踪系统调用

```bash
# 跟踪进程的所有系统调用
strace -p <PID>

# 只跟踪网络相关系统调用
strace -p <PID> -e trace=network

# 统计系统调用次数和时间
strace -p <PID> -c

# 常见用途：
#   → 找出进程卡在哪里（哪个系统调用耗时）
#   → 找出进程在做什么IO操作
#   → 调试Java Native方法（JNI）
```

### 9.2 gdb — 程序调试

```bash
# 调试正在运行的进程
gdb -p <PID>

# 常用命令：
#   bt        → 打印调用栈
#   info threads → 查看所有线程
#   thread <n> → 切换到线程n
#   frame <n> → 切换到栈帧n
#   print var → 打印变量值

# Java开发者较少直接用gdb，更多用jdb或arthas
```

### 9.3 perf — 性能分析

```bash
# CPU性能分析（采样）
perf top              # 实时显示CPU热点函数
perf record -g -p <PID> # 采样进程的CPU性能数据
perf report           # 分析采样结果

# 常见用途：
#   → 找出CPU时间花在哪个函数（热点分析）
#   → 分析内核态/用户态的时间分布
#   → 找出cache miss率高的代码位置

perf stat -p <PID>    # 统计进程的性能事件（cache miss、分支预测等）
```

> [!tip] Java 开发者推荐使用 arthas
> arthas 是阿里开源的Java诊断工具，比 jdb/gdb 更方便：
> - `thread` → 查看线程状态
> - `dashboard` → 实时面板
> - `profiler` → 火焰图生成
> - `sc/dump` → 查看类信息
> - `watch/trace` → 方法调用追踪

---

## 面试题速查

> [!example] 本模块高频面试题

1. **硬链接和软链接的区别？删除文件的底层发生了什么？**
   - 硬链同inode不能跨区，软链独立inode可跨区；删原文件硬链不受影响，软链失效

2. **Linux文件权限详解？SUID/SGID/Sticky Bit？**
   - rwx数字法(755)；SUID=以文件所有者身份执行；Sticky Bit=只有文件主人能删(/tmp)

3. **删除文件磁盘空间不释放怎么排查？**
   - `lsof | grep deleted` → 进程持有fd → inode引用计数不为0 → kill进程或重载

4. **OOM Killer排查？JVM为什么容易被误杀？**
   - `dmesg | grep oom` → Linux按虚拟内存算oom_score → JVM虚拟内存远大于实际使用

5. **Linux查看端口占用、查看进程打开的文件、查看磁盘IO的命令？**
   - 端口：`lsof -i :8080` 或 `ss -antp | grep 8080`
   - 文件：`lsof -p <PID>`
   - 磁盘IO：`iostat -x 1`

6. **strace/gdb/perf基本使用？**
   - strace跟踪系统调用；gdb调试程序；perf性能分析热点函数

7. **常见文件系统对比？ext4/xfs/btrfs？**
   - ext4通用默认；xfs大文件高并发；btrfs快照校验

---

## 交叉链接

- [[CPU缓存与伪共享|CPU缓存与伪共享]] — 同在底层原理模块
- [[01-进程管理/进程与线程|进程与线程]] — 进程管理命令、僵尸进程排查
- [[04-IO与网络模型/零拷贝|零拷贝]] — PageCache与文件系统
- [[../操作系统总览|操作系统总览]] — 回到总览