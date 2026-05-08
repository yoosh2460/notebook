# 集合与并发

本目录用于 Java 集合与并发方向的面试准备。组织原则是：一个核心概念对应一个 Markdown 文件，不把多个概念混写到同一篇笔记中。

## 文件索引

- [01-ArrayList](./01-ArrayList.md)
- [02-HashMap](./02-HashMap.md)
- [03-ConcurrentHashMap](./03-ConcurrentHashMap.md)
- [04-CopyOnWriteArrayList](./04-CopyOnWriteArrayList.md)
- [05-BlockingQueue](./05-BlockingQueue.md)
- [06-线程池-ThreadPoolExecutor](./06-线程池-ThreadPoolExecutor.md)
- [07-synchronized](./07-synchronized.md)
- [08-ReentrantLock](./08-ReentrantLock.md)
- [09-volatile](./09-volatile.md)
- [10-ThreadLocal](./10-ThreadLocal.md)
- [11-CompletableFuture](./11-CompletableFuture.md)
- [12-AQS](./12-AQS.md)

## 面试复习方式

- 先看“面试常见问法”，明确这个知识点会怎么被问。
- 再背“核心回答”，形成 1 到 3 分钟的口述框架。
- 然后看“项目结合”和代码示例，把答案落到 Java 后端项目经验。
- 最后看“深入追问”和“易错点”，准备面试官继续深挖。

## 写作约定

- 每篇只讨论一个概念，关联知识只做必要引用。
- 必须结合 Java 后端真实场景，例如接口聚合、缓存加载、异步任务、订单状态流转、批量数据处理。
- 涉及代码时，关键逻辑必须使用简体中文注释。
- 涉及流程、状态或线程协作时，优先使用 Mermaid 辅助说明。
