# Minimal Kafka 任务列表

## 基础阶段（核心功能构建）

### 学习目标
1. 了解消息系统的基本概念：**Producer**、**Consumer**、**Topic**、**Partition**、**Offset**、**日志存储**  
2. 能够搭建一个最基础的**消息队列**模型，支持单机环境下的生产、消费  
3. 掌握**消息持久化**和**简单文件日志**的实现思路  
4. 为后续的**副本、分布式**扩展打下基础

---

### 任务1：搭建项目结构与环境
**场景**：初始化最基础的项目框架，类似 “Minimal Kafka Core”

- [x] **创建 Maven 项目**  
  - 目录结构：`src/main/java` + `src/test/java`  
  - 在 `pom.xml` 中定义 `groupId`, `artifactId`, `version`，并确保能正常编译  
  - 可以先引入一些基础依赖（如日志框架 log4j 或 slf4j + logback）

- [x] **定义核心类**：
  1. `Broker`：模拟单机 Broker 的“服务器”类，负责协调 Producer/Consumer 连接、管理 Topic/Partition  
  2. `Topic`：封装主题相关信息，如名字、分区列表等  
  3. `Partition`：每个分区对应一段日志文件或内存队列  
  4. `Record`：消息的实体结构（包含 key、value、timestamp 等）

- [x] **编写简单测试**或 `main` 方法，验证工程能正常运行**并打印一条欢迎语**  
  - “Minimal Kafka 启动成功” 或类似提示

**产出要求**：
1. 控制台可打印 “Minimal Kafka 启动”  
2. 代码能成功编译、运行  
3. `Broker` 等核心类可被简单创建（哪怕方法体暂时为空）  

---

### 任务2：实现最基本的消息队列（内存版）
**场景**：先只在**内存**中存储消息，支持简单的生产与消费

- [x] **实现 `Broker`：**  
  - 提供方法 `createTopic(String topicName, int numPartitions)` 创建 Topic 和分区  
  - 提供管理 `topics: Map<String, Topic>` 的容器  

- [x] **编写 `Producer`**：  
  - 提供 `send(String topic, String key, String value)` 方法  
  - 根据 topicName 找到对应 Topic，再根据 key 或简单的轮询算法选择一个 Partition  
  - 将消息（Record）写入 `Partition` 的内存队列 `List<Record>` 或 `Queue<Record>`

- [x] **编写 `Consumer`**：  
  - 提供 `poll(String topic, int partitionId)` 或 `poll(String topic, int partitionId, long offset)` 等方法  
  - 拉取该分区内的新消息，将 offset 往前推进

- [x] **简单的 Offset 机制**：  
  - 在 `Consumer` 中维护一个 `currentOffset`（可以先放在内存变量）  
  - 每次消费后 +1  
  - 不考虑持久化，仅做基础演示

**产出要求**：
1. 能在单机里创建一个 Topic，发送几条消息，然后用 Consumer 拉取到这些消息  
2. 日志或控制台可打印 “Producer sent message” / “Consumer polled message: offset=XXX”  
3. 多次运行后能看到 offset 不断前进（但此时重启后 offset 会重置，因为尚未持久化）  

---

### 任务3：实现日志存储（文件版 Partition Log）
**场景**：让消息可以落盘持久化，模拟 Kafka 的日志文件

- [x] **Partition 文件结构**：
  1. 在每个 Partition 下，对应一个或多个日志文件（如 `topicName-part0.log`）  
  2. 每次写入一条 Record，就序列化后追加到文件末尾  
  3. 建议先用**追加写**的方式，不做索引，后续再扩展

- [x] **Record 序列化/反序列化**：  
  - 简单实现：可以用 Java 自带的 `ObjectOutputStream` / `ObjectInputStream`，或 JSON 方式  
  - 写到文件时，在末尾写入一条消息的大小与消息内容

- [x] **Offset 管理**：  
  - 文件中可以隐式使用行号/条目号当 offset（第几条消息）  
  - 或在文件头/另一个索引文件中记录当前写入位置

- [x] **Consumer 读取日志**：  
  - 通过 `offset` 定位到文件的对应位置（可以先从头顺序读到 offset，再开始读新的消息）  
  - 优化：可以存一个 “offset → 文件位置” 的小型索引，减少重复扫描

**产出要求**：
1. 重启后，之前写入的消息依然存在  
2. Consumer 可以在上次读取的 offset 基础上继续消费后续消息  
3. 日志/控制台打印写入与读取的文件位置、消息信息，验证文件读写是否成功  

---

### 任务4：Broker 启动/停止流程 & 简单网络交互（可选）
**场景**：让 `Broker` 具备基本的启动/关闭逻辑，支持客户端通过网络发送/消费消息（可先用 Socket）

- [x] **`Broker.start()` & `Broker.shutdown()`**：  
  - 启动时加载已有的日志文件，初始化各 Partition、offset 状态  
  - 关闭时将内存中的未落盘数据或元信息刷写到磁盘（如果有）

- [x] **网络通信（初步）**：  
  - 在 Broker 里开启一个 Socket Server（可以用 BIO、NIO 或 Netty）  
  - 定义简单协议：例如“`SEND|topic|key|value`” 字符串方式，Broker 收到后写入对应 Partition  
  - `Consumer` 可以通过 Socket 发送类似 “`POLL|topic|partition|offset`” 指令获取消息

- [x] **简单测试**：  
  - 启动 Broker（监听某个端口），然后写一个 Producer/Consumer CLI 或测试方法，通过 Socket 连接 Broker  
  - Producer 发送消息 → Broker 落盘 → Consumer 请求消息 → Broker 返回

**产出要求**：
1. Broker 能够启动，并在控制台输出 “Broker started, listening on port ...”  
2. 手动关停时输出 “Broker shutdown, flushing logs...” 并保证文件写入完整  
3. Producer/Consumer 在网络层面也能工作，完成基本的发送和拉取  

---

## 中级阶段（增强功能与集群雏形）

### 学习目标
1. 支持 **多副本**（Replication）和 **Leader/Follower** 机制的基本概念  
2. 掌握**Consumer Group** 与**分区再平衡**的基础思想  
3. 尝试**分布式协调**（简易版），引入**Zookeeper** 或自定义协调模块  
4. 优化日志、实现**Segment+Index** 等更高级特性

---

### 任务5：多副本 & Leader/Follower 机制（简易）
**场景**：模拟 Kafka 的多副本架构，提升数据可靠性

- [x] **副本结构**：  
  - 在 Broker 中，每个 Partition 都有一个 Leader，若干 Follower  
  - Leader 负责处理写请求，Follower 定期从 Leader 同步数据

- [x] **Leader 选举（最简单版）**：  
  - 可以固定某台 Broker 是 Leader，其他是 Follower  
  - 或在一组 Broker 启动时，先选第一个上线的作为 Leader

- [x] **同步流程**：  
  - Follower 周期性地拉取 Leader 的最新 offset 之后的记录，并写到本地日志文件  
  - 可以手动模拟延迟、网络抖动，以测试同步的健壮性

- [x] **确认机制**：  
  - Producer 发送时等待 “ISR”（In-Sync Replicas）中的多数副本都写入成功才返回 ack  
  - 最基础版本里可以只支持 Leader 写完就返回

**产出要求**：
1. 启动多个 Broker（在不同端口），定义 Topic 有 2 副本  
2. 观察 Follower 同步数据到自己的日志文件  
3. 测试简单的容错：手动关闭 Leader Broker，再启动，看看 Follower 能不能成为新的 Leader（若实现了选举逻辑）  

---

### 任务6：Consumer Group 与分区再平衡
**场景**：多个 Consumer 组成一个消费组，对同一 Topic 的多个分区进行负载均衡

- [x] **Group 协调器**：  
  - 需要一个中心（可使用其中一台 Broker 或一个单独的协调服务，比如简化版 Zookeeper），记录 groupId、consumerId、分配方案  
  - 当有新的 Consumer 加入或退出时，触发 rebalancing

- [x] **分区分配策略**：  
  - RoundRobin / Range 等常见方式  
  - 例如给每个 Consumer 分配若干分区

- [x] **Offset 提交**：  
  - 每个 Consumer 周期性地将当前消费进度（offset）提交给协作方（Broker 或 ZK），便于在重平衡或崩溃恢复时继续消费

- [x] **测试**：  
  - 启动 2~3 个 Consumer（groupId 相同），让它们消费同一个 Topic 的多分区  
  - 日志里看到不同分区分配到不同 Consumer  
  - 动态添加/移除 Consumer，观察分区再平衡过程

**产出要求**：
1. 多个 Consumer 能共同消费 Topic 的所有分区，每个分区只给一个 Consumer 拉取  
2. 当 Consumer 数量改变时，分区重新分配，并继续正常消费  
3. offset 能正确保存并在重新分配后继续  

---

### 任务7：日志分段与索引（Segment + Index）
**场景**：学习 Kafka 常用的日志段管理，避免单个大文件过度膨胀

- [x] **Segment 切分**：  
  - 按固定大小或时间间隔，将日志拆分为多个段文件（`log-0`, `log-1`, `log-2`, ...）  
  - 每个段文件都有自己的起始 offset

- [x] **Index 文件**：  
  - 对于每个段文件，可以建立简单索引（`index-0`）存储 “相对 offset → 文件物理位置”  
  - Consumer 如果要从中间 offset 读取，可以根据索引快速定位文件位置

- [x] **Segment 清理**：  
  - 可以保留一定时间或大小后，删除或压缩旧段  
  - 简单实现：通过配置 “保留7天” 或 “最大100MB” 等策略，在 Broker 启动/定期扫描时删除老段

**产出要求**：
1. 发送大量消息后能看到自动滚动生成多个段文件  
2. Consumer 可以基于索引快速定位 offset，不必从头扫描  
3. 观测到自动清理删除过期或超量日志段  

---

### 任务8：引入 Zookeeper 或自定义集群协调模块
**场景**：学习 Kafka 早期依赖 ZK 做**元数据管理**和**选举**的思想

- [x] **Zookeeper 存储 Broker/Topic/Partition 信息**：
  - 每个 Broker 启动时，在 ZK 注册自己  
  - Topic/Partition 元信息也写进 ZK  
  - Leader 选举：每个分区的 Leader 在 ZK 里记录当前 Leader ID

- [x] **自定义协调模块**（若不想用 ZK）：
  - 用单台 Broker 做主协调器，保存集群信息（类似 etcd/raft）  
  - 其他 Broker 向协调器报告心跳、元数据

- [x] **故障恢复**：  
  - 当 Leader Broker 宕机时，ZK/协调器检测到失联，自动将副本中最优者推举为新 Leader  
  - Producer/Consumer 通过 ZK/协调器查询最新 Leader

**产出要求**：
1. 多台 Broker 能启动并在 ZK/自定义协调模块中注册自身  
2. Topic/Partition 的 Leader 信息通过 ZK/协调模块存储、更新  
3. Leader 宕机后自动完成选举，新 Leader 替代旧 Leader 提供写服务  

---

## 高级阶段（完善与优化）

### 学习目标
1. 优化消息系统的吞吐和可靠性：零拷贝、批量发送、压缩  
2. 学习更多真实场景：**压缩**、**TLS/安全**、**控制器**、**事务**、**Exactly-once** 语义  
3. 处理复杂异常和运维管理：如**抓 metrics**、**监控**、**重平衡错误**、**延迟**等  
4. 提供**可插拔**的序列化方式与认证机制

---

### 任务9：提升吞吐与安全（批量发送、压缩、TLS）
**场景**：在实际生产中，Kafka 常使用**批量**与**压缩**来提升效率，同时可启用 SSL/TLS 加密

- [ ] **批量发送**：  
  - 在 Producer 中，收集到一定数量/大小的消息再一次性写盘或发给 Broker  
  - 可以配置 `batch.size`, `linger.ms` 等模拟 Kafka 参数

- [ ] **压缩**：  
  - 支持简单压缩算法（如 GZIP/Snappy/LZ4）  
  - 发送或落盘前先压缩，Consumer 取到后解压

- [ ] **TLS/SSL 加密**（可选）  
  - 在 Socket 通信层加 TLS，加密传输中的消息  
  - 简单地引入自签名证书即可演示

**产出要求**：
1. 发送 1000 条消息，观察批量模式是否能减少网络/文件 IO 次数  
2. 验证压缩功能：日志文件中显示明显变小，Consumer 能正确解压  
3. 若启用 TLS，可看到通信加密，抓包也无法直接看到明文消息  

---

### 任务10：监控与可视化（Metrics & Admin Tools）
**场景**：类似 Kafka 自带的 JMX metrics、CLI 工具

- [ ] **Metrics**：  
  - 统计 Producer/Consumer 每秒发送/接收的消息数、Broker 端内存使用、磁盘写入速率等  
  - 暴露一个简单的 HTTP 接口或 JMX 指标

- [ ] **Admin 工具**：  
  - 编写命令行工具 “`minikafka-topics.sh`”，可以列出当前已有的 Topic、分区、Leader/Follower 状态  
  - 可以创建/删除 Topic、查看 Broker 列表等

- [ ] **可视化（可选）**：  
  - 做一个简易 Web UI 或 Grafana dashboard 让人查看 Topic、Broker 运行状态

**产出要求**：
1. 能通过命令行或 HTTP 看到实时的消息吞吐指标  
2. 能列出 Topic 和分区信息，并能创建/删除  
3. 若实现了可视化界面，可以演示查看集群状态  

---

### 任务11：事务与 Exactly-once（进阶可选）
**场景**：在金融或关键业务中，需要**精确一次**语义

- [ ] **Producer 事务**：  
  - Producer 发起事务，写入多条消息，再 commit 或 abort  
  - Broker 需要在日志里记录事务状态

- [ ] **Consumer 事务感知**：  
  - 只读取已 commit 的事务消息  
  - 避免消费到中途 abort 的消息

- [ ] **幂等性**：  
  - 同样的消息发送多次，不会在日志中重复存储  
  - 需要 Producer 层有“序列号”机制

**产出要求**：
1. 演示 Producer 在一个事务中写多条消息，Consumer 只能看到 commit 后的结果  
2. 测试异常中断看是否可以保留原子性  
3. 如果做了幂等性功能，多次发送同一条消息只记录一次  

---

### 任务12：增强异常体系与容错处理
**场景**：捕获更多错误，提供友好提示及自动恢复措施

- [ ] **自定义异常**：`BrokerNotAvailableException`, `OffsetOutOfRangeException`, `LeaderNotFoundException`, 等  
- [ ] **在生产、消费、同步、选举等阶段** 捕获并包装异常：  
  - 提示当前 Topic/Partition/offset、Broker ID、错误上下文  
- [ ] **自动重试与告警**：  
  - 当 Follower 同步失败可自动重试 N 次；无法恢复则上报或告警  
  - Consumer offset 越界时自动回退到最近有效位置

**产出要求**：
1. 当网络或磁盘失败时，能抛出明确异常并记录日志  
2. 便于快速定位问题：日志包含 Broker、Topic、分区、offset 信息  
3. 可在配置文件中设定重试次数、告警方式  

---

### 任务13：可插拔序列化和鉴权机制
**场景**：允许用户灵活自定义消息格式和安全认证

- [ ] **序列化 SPI**：  
  - 定义 `Serializer`/`Deserializer` 接口  
  - 提供默认 JSON 或二进制实现  
  - 用户可通过配置文件或 SPI 插件替换实现

- [ ] **鉴权模块**：  
  - 用户通过用户名/密码、Token、OAuth2 等方式连接 Broker  
  - Broker 验证成功后才允许生产/消费指定 Topic（可扩展 ACL 机制）

**产出要求**：
1. 在不改核心代码的前提下，能替换 `JSONSerializer` 为自定义 `ProtobufSerializer` 等  
2. Broker 可对连接进行简单鉴权，验证失败则拒绝请求  
3. 通过日志或命令行工具可查看哪些客户端已连接  

---

## 附加任务（可选拓展）
1. **零拷贝优化**：模拟 Kafka 的 FileChannel.transferTo 或 sendfile  
2. **搭建 CI/CD 流程**：自动化构建 & 测试  
3. **性能压测**：编写压测脚本，统计不同批量大小、压缩方式、硬件配置下的吞吐  
4. **多数据中心复制**：跨机房复制，模拟 MirrorMaker 功能  
5. **消息过期 & 清理策略**：按时间或按大小自动删除旧消息  

---

## 总结

通过以上**基础 → 中级 → 高级**的任务链，你可以逐步实现一个“最小可用版”的 Kafka 系统原型，从**内存队列**到**文件日志**、再到**分布式多副本**、**Consumer Group**、**事务**等高级特性。每个阶段都对应了核心概念的落地实现，帮助你深入理解 Kafka 背后的原理和架构。

**建议**：
- 每完成一个阶段，就**写好单元测试**与**集成测试**，并**记录设计文档**。  
- 过程中随时可以对照真正的 Kafka 源码做对比、查阅官方文档了解更多实现细节。  
- 如果感觉任务太多，可以挑最核心的部分先完成，也可以跳过部分高级特性，根据个人学习节奏灵活调整。  

祝你在这个“Minimal Kafka”项目中学习愉快，成功掌握分布式消息系统的关键原理与实现技巧！