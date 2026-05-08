# OOM 分类与排查

## 面试常见问法

- 线上 OOM 怎么排查？
- `Java heap space` 和 `Metaspace` 有什么区别？
- `Direct buffer memory` 怎么定位？
- `unable to create new native thread` 是堆内存不够吗？
- OOM 时必须保留哪些现场？

## 核心回答

OOM 不是一种问题，而是一类内存资源耗尽问题。排查第一步是看异常信息，判断是堆、元空间、直接内存、线程栈还是 GC overhead。堆 OOM 重点分析 heap dump；元空间 OOM 重点看类加载数量和 ClassLoader 泄漏；直接内存 OOM 关注 NIO、Netty 和 `MaxDirectMemorySize`；无法创建线程通常是线程数量、栈大小、系统进程限制或本地内存不足。

## 核心概念

| OOM 类型 | 典型信息 | 排查方向 |
|---|---|---|
| 堆溢出 | `Java heap space` | heap dump、对象引用链、缓存和集合 |
| 元空间溢出 | `Metaspace` | 动态类生成、ClassLoader 泄漏 |
| 直接内存溢出 | `Direct buffer memory` | NIO、Netty、堆外内存上限 |
| 线程创建失败 | `unable to create new native thread` | 线程数、`-Xss`、系统限制 |
| GC 开销过大 | `GC overhead limit exceeded` | 堆几乎回收不动，疑似泄漏 |

## 项目结合

生产环境建议默认开启 OOM 现场保留。

```bash
java -Xms2g -Xmx2g \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/data/dump/app.hprof \
     -XX:ErrorFile=/data/dump/hs_err_pid%p.log \
     -jar app.jar
```

堆 OOM 的基本排查命令：

```bash
# 查看对象直方图，先快速判断大对象类型
jmap -histo <pid> | head -30

# 导出 heap dump，线上执行要评估暂停风险
jmap -dump:format=b,file=/data/dump/app.hprof <pid>
```

## 深入追问

- **堆 OOM 怎么定位泄漏对象？** 用 MAT 查看 Dominator Tree 和 Leak Suspects，找到占用最大的对象和 GC Roots 引用链。
- **元空间 OOM 常见原因？** CGLIB、Javassist、动态脚本、热部署反复创建 ClassLoader，导致类元数据无法卸载。
- **线程 OOM 为什么不是堆问题？** 线程栈和线程本身占用本地内存，线程数量过多会耗尽进程或系统资源。
- **OOM 后还能用 `jmap` 吗？** 不一定。进程可能已崩溃或响应很慢，所以自动 dump 和错误日志非常重要。

## 常见问题

- 不要看到 OOM 就直接加大 `-Xmx`，要先判断是哪类 OOM。
- 不要只保留应用日志，heap dump、GC 日志和 `hs_err` 文件同样重要。
- 不要在线上高峰期随意 dump 大堆，可能造成长时间暂停。

## 一句话总结

OOM 排查的面试核心是先分类，再保留现场，最后根据堆、元空间、直接内存或线程资源分别定位。
