# Kafka高级特性与源码分析

## 1. 概述

Kafka 作为高吞吐分布式消息系统，除了基础的生产消费功能外，还提供了一系列高级特性满足生产环境的可靠性、性能和可观测性需求。本文从**生产者高级特性**、**消费者高级特性**、**服务端存储机制**、**副本与一致性**、**性能调优**和**核心源码分析**六个维度深入解析。

```
Kafka 知识体系:

基础层                          高级层
┌──────────────┐              ┌──────────────┐
│ Producer API │              │ 幂等生产者   │
│ Consumer API │              │ 事务消息     │
│ Topic/Partition│            │ 精确一次语义 │
│ Broker/ISR   │              │ 消费者再均衡 │
└──────────────┘              │ 偏移量管理   │
                              │ 日志压缩     │
                              │ 副本同步机制 │
                              │ 性能调优     │
                              │ 源码分析     │
                              └──────────────┘
```

## 2. 生产者高级特性

### 2.1 幂等生产者（Idempotent Producer）

```
幂等性解决重复发送问题:

生产者重试导致重复:
┌──────────┐        ┌──────────┐        ┌──────────┐
│ Producer │──Msg──→│  Broker  │──ACK──→│ Producer │
│ 发送 msg │        │ 已写入   │        │ 未收到ACK│
└──────────┘        └──────────┘        └──────────┘
       │                                    │
       │ 重试发送                           │
       ▼                                    ▼
┌──────────┐        结果: msg 被写入两次
│ Producer │──Msg──→ (重复消息)
└──────────┘

幂等生产者原理:
┌──────────┐
│ Producer │  PID (Producer ID) + Sequence Number
│  PID=123 │  每个分区维护单调递增的序列号
│  Seq=0   │  Broker 端去重: (PID, Seq) 相同则丢弃
└──────────┘
```

```java
// 开启幂等生产者
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
    StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
    StringSerializer.class.getName());

// 开启幂等性（默认 false）
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");

// 幂等性开启后，以下参数会被强制设置:
// retries = Integer.MAX_VALUE
// acks = all
// max.in.flight.requests.per.connection = 5

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

### 2.2 事务消息（Transactions）

```
Kafka 事务实现精确一次语义 (Exactly-Once Semantics, EOS):

跨分区事务示例:
┌──────────────┐
│  Producer    │  initTransactions()
│  Transaction │  beginTransaction()
│     ID       │  send(topic-1, msg-1)
│  txn-id-01   │  send(topic-2, msg-2)
│              │  commitTransaction()
└──────────────┘
       │
       ▼
┌────────────────────────────────────────┐
│  Transaction Coordinator               │
│  - 管理事务状态                        │
│  - 记录事务日志 (__transaction_state)  │
│  - 协调提交/回滚                       │
└────────────────────────────────────────┘
       │
   ┌───┴───┐
   ▼       ▼
Partition1 Partition2
 (msg-1)   (msg-2)

事务隔离级别:
- read_uncommitted: 读取所有消息（默认）
- read_committed:   只读取已提交事务消息（过滤未提交和回滚）
```

```java
// 事务生产者
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
    StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
    StringSerializer.class.getName());

// 事务配置
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "my-transactional-id");
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// 初始化事务
producer.initTransactions();

try {
    producer.beginTransaction();
    
    // 发送多条消息
    producer.send(new ProducerRecord<>("topic-a", "key1", "value1"));
    producer.send(new ProducerRecord<>("topic-b", "key2", "value2"));
    
    // 发送消费位移（consume-transform-produce）
    producer.sendOffsetsToTransaction(
        consumer.position(consumer.assignment()),
        consumer.groupMetadata()
    );
    
    producer.commitTransaction();
} catch (ProducerFencedException | OutOfOrderSequenceException e) {
    // 不可恢复异常，关闭生产者
    producer.close();
} catch (KafkaException e) {
    // 可恢复异常，回滚事务
    producer.abortTransaction();
}
```

### 2.3 生产者拦截器

```java
// 自定义拦截器：添加时间戳和来源标识
public class AuditInterceptor implements ProducerInterceptor<String, String> {
    
    @Override
    public ProducerRecord<String, String> onSend(
            ProducerRecord<String, String> record) {
        // 在消息发送前修改
        Headers headers = record.headers();
        headers.add("timestamp", 
            String.valueOf(System.currentTimeMillis()).getBytes());
        headers.add("source", "order-service".getBytes());
        return record;
    }
    
    @Override
    public void onAcknowledgement(
            RecordMetadata metadata, Exception exception) {
        // ACK 回调
        if (exception != null) {
            // 记录发送失败日志
        }
    }
    
    @Override
    public void close() {}
    
    @Override
    public void configure(Map<String, ?> configs) {}
}

// 配置拦截器
props.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, 
    "com.example.kafka.interceptor.AuditInterceptor");
```

### 2.4 生产者压缩与批量优化

```
生产者发送流程:

消息 → 序列化器 → 分区器 → 记录累加器(RecordAccumulator)
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
                    ▼                         ▼                         ▼
              Partition 0               Partition 1               Partition 2
              ┌─────────┐               ┌─────────┐               ┌─────────┐
              │ Batch   │               │ Batch   │               │ Batch   │
              │ Record  │               │ Record  │               │ Record  │
              │ Record  │               │ Record  │               │ Record  │
              └────┬────┘               └────┬────┘               └────┬────┘
                   │                         │                         │
                   └─────────────────────────┼─────────────────────────┘
                                             │ 批量发送 (linger.ms/batch.size)
                                             ▼
                                        Sender 线程
                                             │
                                             ▼
                                        Broker
```

```java
// 生产者性能优化配置
Properties props = new Properties();

// 批量大小：默认 16KB，增大可减少网络请求
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 32 * 1024);  // 32KB

// 等待时间：默认 0，适当增大可提升吞吐量
props.put(ProducerConfig.LINGER_MS_CONFIG, 10);  // 10ms

// 压缩算法：减少网络传输和存储
// none(默认), gzip, snappy, lz4, zstd
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

// 缓冲区大小：默认 32MB
props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 64 * 1024 * 1024);  // 64MB

// 发送确认：0(不等待), 1(Leader确认), all(所有ISR确认)
props.put(ProducerConfig.ACKS_CONFIG, "all");

// 重试次数
props.put(ProducerConfig.RETRIES_CONFIG, 3);

// 客户端 ID（用于监控）
props.put(ProducerConfig.CLIENT_ID_CONFIG, "order-producer-01");
```

## 3. 消费者高级特性

### 3.1 消费者组再均衡（Rebalance）

```
再均衡触发时机:

初始加入:
Consumer-1, Consumer-2, Consumer-3
     │           │           │
     └───────────┼───────────┘
                 ▼
          Group Coordinator
                 │
                 ▼
     Partition 0, 1, 2, 3, 4, 5
     ├─ C1: P0, P1
     ├─ C2: P2, P3
     └─ C3: P4, P5

成员变化（Consumer-2 宕机）:
Consumer-1, Consumer-3
     │           │
     └───────────┘
                 ▼
          触发 Rebalance
                 │
                 ▼
     Partition 0, 1, 2, 3, 4, 5
     ├─ C1: P0, P1, P2
     └─ C3: P3, P4, P5
```

```java
// 再均衡监听器
public class RebalanceListener implements ConsumerRebalanceListener {
    
    private KafkaConsumer<String, String> consumer;
    private Map<TopicPartition, Long> currentOffsets;
    
    public RebalanceListener(KafkaConsumer<String, String> consumer,
                             Map<TopicPartition, Long> currentOffsets) {
        this.consumer = consumer;
        this.currentOffsets = currentOffsets;
    }
    
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // 分区被收回前：提交偏移量
        System.out.println("分区被收回: " + partitions);
        commitOffsets();
    }
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // 分区分配后：重置到指定位置
        System.out.println("分区被分配: " + partitions);
        for (TopicPartition partition : partitions) {
            Long offset = getOffsetFromDB(partition);
            if (offset != null) {
                consumer.seek(partition, offset);
            }
        }
    }
    
    private void commitOffsets() {
        // 将偏移量保存到外部存储（如 MySQL/Redis）
        for (Map.Entry<TopicPartition, Long> entry : currentOffsets.entrySet()) {
            saveOffsetToDB(entry.getKey(), entry.getValue());
        }
    }
}

// 使用再均衡监听器
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("order-topic"), 
    new RebalanceListener(consumer, currentOffsets));
```

### 3.2 偏移量管理策略

```
偏移量提交方式对比:

自动提交（默认）:
┌──────────────┐     每 5 秒提交一次     ┌──────────────┐
│  Consumer    │ ──────────────────────→ │ __consumer   │
│ poll() 消费  │                         │ _offsets     │
└──────────────┘                         └──────────────┘
风险: 可能重复消费（已消费但未提交时崩溃）

手动同步提交:
┌──────────────┐
│  Consumer    │  poll()
│  处理消息     │    │
│  commitSync()│ ←──┘ 处理完成后立即提交
└──────────────┘
特点: 精确但阻塞，吞吐量低

手动异步提交:
┌──────────────┐
│  Consumer    │  poll()
│  处理消息     │    │
│  commitAsync()│ ←─┘ 非阻塞提交，带回调
└──────────────┘
特点: 高吞吐但可能提交失败

外部存储（精确控制）:
Consumer → 消费消息 → 业务处理 → 保存偏移量到 MySQL
     │                                    │
     └────────────────────────────────────┘
               事务保证（消息处理 + 偏移量更新）
```

```java
// 手动提交偏移量
Properties props = new Properties();
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("order-topic"));

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        
        for (ConsumerRecord<String, String> record : records) {
            // 处理消息
            processMessage(record);
        }
        
        // 同步提交（阻塞直到成功或失败）
        try {
            consumer.commitSync();
        } catch (CommitFailedException e) {
            // 提交失败处理
            log.error("偏移量提交失败", e);
        }
        
        // 或异步提交（非阻塞）
        consumer.commitAsync((offsets, exception) -> {
            if (exception != null) {
                log.error("异步提交失败: {}", offsets, exception);
            }
        });
    }
} finally {
    consumer.close();
}
```

### 3.3 消费者心跳与会话超时

```
消费者心跳机制:

Consumer                       Group Coordinator
   │                                 │
   │────── Heartbeat Request ──────→│  每 3 秒（heartbeat.interval.ms）
   │                                 │
   │←───── Heartbeat Response ──────│
   │                                 │
   │                                 │
   │  如果 10 秒（session.timeout.ms）未收到心跳:
   │                                 │
   │  Coordinator 认为 Consumer 死亡
   │                                 │
   │  触发 Rebalance                 │

关键参数:
session.timeout.ms:   会话超时（默认 10s）
heartbeat.interval.ms: 心跳间隔（默认 3s）
max.poll.interval.ms:  两次 poll 最大间隔（默认 5min）
```

```java
// 消费者心跳与超时配置
Properties props = new Properties();

// 心跳间隔（必须小于 session.timeout.ms 的 1/3）
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000);  // 3s

// 会话超时
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 10000);  // 10s

// 两次 poll 最大间隔（处理逻辑耗时需小于此值）
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);  // 5min

// 每次 poll 返回的最大记录数
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
```

## 4. 服务端存储机制

### 4.1 日志存储结构

```
Kafka 日志文件组织:

Topic: order-events
└── Partition-0
    ├── 00000000000000000000.log      (消息日志)
    ├── 00000000000000000000.index    (偏移量索引)
    ├── 00000000000000000000.timeindex (时间戳索引)
    ├── 00000000000000001024.log      (下一个日志段)
    ├── 00000000000000001024.index
    └── 00000000000000001024.timeindex

日志段（LogSegment）:
┌────────────────────────────────────────┐
│  00000000000000000000.log              │
│  ─────────────────────────────────────  │
│  Offset: 0                             │
│  Position: 0                           │
│  MessageSize: 120 bytes                │
│  ─────────────────────────────────────  │
│  Offset: 1                             │
│  Position: 120                         │
│  MessageSize: 98 bytes                 │
│  ─────────────────────────────────────  │
│  Offset: 2                             │
│  Position: 218                         │
│  MessageSize: 156 bytes                │
└────────────────────────────────────────┘

稀疏索引（Index）:
┌──────────────┬──────────────┐
│  Relative    │  Physical    │
│  Offset      │  Position    │
├──────────────┼──────────────┤
│  0           │  0           │
│  10          │  1250        │
│  23          │  3124        │
│  41          │  5680        │
└──────────────┴──────────────┘
索引间隔: log.index.interval.bytes = 4KB
```

### 4.2 日志清理策略

```
两种清理策略:

删除策略（delete）- 基于时间/大小:
┌────────────────────────────────────────┐
│  Log Retention                         │
│  ├── retention.ms = 7 days             │
│  ├── retention.bytes = 1GB             │
│  └── 任一条件满足即删除旧日志段        │
└────────────────────────────────────────┘

压缩策略（compact）- 基于 Key:
┌────────────────────────────────────────┐
│  Log Compaction                        │
│                                        │
│  偏移量  Key       Value               │
│  ─────────────────────────             │
│  0      user-1     {"name":"Alice"}    │
│  1      user-2     {"name":"Bob"}      │
│  2      user-1     {"name":"Alicia"}   │ ← 相同 Key
│  3      user-3     {"name":"Charlie"}  │
│  4      user-1     {"name":"Ali"}      │ ← 相同 Key
│  ─────────────────────────             │
│                                        │
│  压缩后保留每个 Key 的最新值:          │
│  0      user-1     {"name":"Ali"}      │
│  1      user-2     {"name":"Bob"}      │
│  3      user-3     {"name":"Charlie"}  │
│                                        │
│  适用: 配置信息、状态变更日志          │
└────────────────────────────────────────┘
```

## 5. 副本机制与一致性

### 5.1 ISR（In-Sync Replicas）

```
副本同步机制:

Partition-0 (3 副本)
┌────────────────────────────────────────┐
│                                        │
│   Leader (Broker-1)                    │
│   ┌──────────┐                         │
│   │ Offset:  │  10                     │
│   │ LEO:     │  12  (Log End Offset)   │
│   │ HW:      │  10  (High Watermark)   │
│   └──────────┘                         │
│        │                               │
│   ┌────┴────┐                          │
│   ▼         ▼                          │
│ Follower-1 Follower-2                  │
│ (Broker-2)  (Broker-3)                 │
│ LEO: 12     LEO: 10                    │
│ ISR: [1,2]  滞后过多，踢出 ISR          │
│                                        │
│ ISR: Broker-1, Broker-2                │
│ （HW = min(所有 ISR 的 LEO)）          │
│                                        │
└────────────────────────────────────────┘

LEO (Log End Offset): 副本最后一条消息的 offset + 1
HW (High Watermark):  已提交消息的最大 offset（消费者只能读到 HW）
ISR: 与 Leader 保持同步的副本集合
```

### 5.2 Leader 选举

```
Leader 失效后的选举:

场景: Leader (Broker-1) 宕机

Partition-0 状态变化:
┌────────────────────────────────────────┐
│  宕机前:                               │
│  Leader:    Broker-1  (LEO=12, HW=10)  │
│  Follower-1: Broker-2  (LEO=12, HW=10) │
│  Follower-2: Broker-3  (LEO=10, HW=8)  │
│                                        │
│  宕机后:                               │
│  1. Controller 检测到 Leader 失效      │
│  2. 从 ISR 中选择新 Leader             │
│  3. 优先选择 LEO 最大的副本            │
│                                        │
│  新 Leader: Broker-2                   │
│  新 HW: min(12, 10) = 10               │
│                                        │
│  Broker-3 需要同步缺失的消息           │
│  (offset 10, 11)                       │
└────────────────────────────────────────┘

unclean.leader.election.enable:
- false (默认): 只有 ISR 中的副本可成为 Leader，数据不会丢失
- true: 允许非 ISR 副本成为 Leader，可能丢失数据
```

### 5.3 精确一次语义（Exactly-Once Semantics）

```
三种语义对比:

At Most Once（最多一次）:
Producer ──Msg──→ Broker (acks=0, 不重试)
特点: 可能丢失，不会重复
适用: 日志收集、可容忍丢失

At Least Once（至少一次）:
Producer ──Msg──→ Broker (acks=1/all, 失败重试)
     │            │
     └─Retry──→   │
特点: 不会丢失，可能重复
适用: 大多数业务场景 + 幂等消费

Exactly Once（精确一次）:
Producer ──Msg──→ Broker (幂等生产者 + 事务)
     │            │
     └─Retry──→   │ (Broker 去重)
特点: 不丢失、不重复
适用: 金融交易、对账系统

实现方式:
1. 幂等生产者（单分区单会话）
2. 事务（跨分区跨会话）
3. 消费者: 事务消费 + 手动提交偏移量
```

## 6. 性能调优

### 6.1 生产者调优

| 参数 | 默认值 | 调优建议 | 影响 |
|------|--------|---------|------|
| `batch.size` | 16KB | 32-64KB | 增大减少网络请求，提升吞吐 |
| `linger.ms` | 0 | 5-10ms | 等待批处理，提升吞吐 |
| `compression.type` | none | lz4/snappy | 减少网络传输 |
| `buffer.memory` | 32MB | 64-128MB | 增大可缓冲更多消息 |
| `acks` | 1 | all | 提升可靠性，降低吞吐 |
| `retries` | 0 | 3-5 | 提升可靠性 |
| `max.in.flight.requests` | 5 | 1-5 | 幂等时设为 5，非幂等时设为 1 |

### 6.2 消费者调优

| 参数 | 默认值 | 调优建议 | 影响 |
|------|--------|---------|------|
| `max.poll.records` | 500 | 100-500 | 减少处理时间，避免 rebalance |
| `max.poll.interval.ms` | 5min | 根据处理时间调整 | 处理慢时增大 |
| `fetch.min.bytes` | 1 | 1-10KB | 减少空轮询 |
| `fetch.max.wait.ms` | 500 | 500-1000 | 与 fetch.min.bytes 配合 |
| `session.timeout.ms` | 10s | 10-30s | 网络不稳定时增大 |

### 6.3 Broker 调优

| 参数 | 默认值 | 调优建议 | 影响 |
|------|--------|---------|------|
| `num.network.threads` | 3 | CPU 核数 | 处理网络请求 |
| `num.io.threads` | 8 | CPU 核数 * 2 | 处理磁盘 IO |
| `log.segment.bytes` | 1GB | 1-2GB | 日志段大小 |
| `log.retention.hours` | 168 (7d) | 根据磁盘调整 | 日志保留时间 |
| `replica.fetch.max.bytes` | 1MB | 与 message.max.bytes 一致 | 副本同步效率 |

## 7. 源码分析

### 7.1 生产者发送流程

```
KafkaProducer.send() 核心流程:

1. 拦截器处理 (Interceptor)
   └─ onSend() 修改消息

2. 序列化 (Serializer)
   └─ key/value 序列化为字节数组

3. 分区器 (Partitioner)
   └─ 有 key: 按 key 哈希取模
   └─ 无 key: 轮询 (sticky partitioner)

4. 记录累加器 (RecordAccumulator)
   └─ 按分区放入双端队列 (Deque)
   └─ 等待批量发送 (batch.size / linger.ms)

5. Sender 线程
   └─ 从累加器读取就绪的 batch
   └─ 构建 ProduceRequest
   └─ 通过 NetworkClient 发送

6. 响应处理
   └─ 成功: 更新 batch 为 done，触发回调
   └─ 失败: 根据 retries 重试，或报错

关键源码类:
- KafkaProducer: 入口类
- RecordAccumulator: 消息缓冲队列
- Sender: 发送线程
- NetworkClient: 网络通信
- Partitioner: 分区选择
```

```java
// KafkaProducer.send() 简化逻辑
public Future<RecordMetadata> send(ProducerRecord<K, V> record, 
                                   Callback callback) {
    // 1. 拦截器
    ProducerRecord<K, V> intercepted = 
        this.interceptors.onSend(record);
    
    // 2. 序列化
    byte[] serializedKey = keySerializer.serialize(
        topic, intercepted.headers(), intercepted.key());
    byte[] serializedValue = valueSerializer.serialize(
        topic, intercepted.headers(), intercepted.value());
    
    // 3. 分区计算
    int partition = partition(record, serializedKey, 
        serializedValue, cluster);
    
    // 4. 写入累加器
    RecordAccumulator.RecordAppendResult result = 
        accumulator.append(topic, partition, timestamp, 
            serializedKey, serializedValue, headers, 
            callback, maxBlockTimeMs);
    
    // 5. 唤醒 Sender 线程
    if (result.batchIsFull || result.newBatchCreated) {
        this.sender.wakeup();
    }
    
    return result.future;
}
```

### 7.2 消费者拉取流程

```
KafkaConsumer.poll() 核心流程:

1. 更新订阅状态
   └─ 检查是否需要重新加入消费者组

2. 确保 Coordinator 已连接
   └─ 查找 GroupCoordinator
   └─ 发送 JoinGroup 请求
   └─ 等待 SyncGroup 响应（分区分配）

3. 发送 Fetch 请求
   └─ 构建 FetchRequest（多个分区）
   └─ 通过 NetworkClient 发送到 Leader Broker

4. 处理响应
   └─ 解析 FetchResponse
   └─ 反序列化消息
   └─ 放入 completedFetches 队列

5. 返回记录
   └─ 从 completedFetches 取出记录
   └─ 更新 position（消费位置）
   └─ 返回 ConsumerRecords

关键源码类:
- KafkaConsumer: 入口类
- ConsumerCoordinator: 消费者组协调
- Fetcher: 消息拉取逻辑
- SubscriptionState: 订阅状态管理
```

### 7.3 副本同步机制

```
Leader 与 Follower 同步:

1. Follower 启动时:
   Follower ──OffsetForLeaderEpoch──→ Leader
        ←────Epoch+StartOffset───────
   
   确定需要同步的起始位置

2. 实时同步:
   Follower ──FetchRequest(offset=X)──→ Leader
        ←────FetchResponse(data)───────
   
   Follower 写入本地日志，更新 LEO

3. Leader 维护 HW:
   定时计算: HW = min(所有 ISR 副本的 LEO)
   
   新 HW 传播:
   Leader ──FetchResponse(HW=newHW)──→ Follower

4. ISR 变更:
   Follower 滞后超过 replica.lag.time.max.ms
   → 踢出 ISR
   → 通知 Controller
   → 更新 ISR 集合

关键源码类:
- ReplicaManager: 副本管理
- Partition: 分区状态机
- ReplicaFetcherThread: Follower 拉取线程
- LogManager: 日志管理
```

## 8. 监控与运维

### 8.1 关键监控指标

| 指标 | 说明 | 告警阈值 |
|------|------|---------|
| `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec` | 每秒写入消息数 | 低于预期 |
| `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` | 每秒写入字节数 | 磁盘瓶颈 |
| `kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec` | 每秒读出字节数 | 网络瓶颈 |
| `kafka.controller:type=KafkaController,name=ActiveControllerCount` | Active Controller 数量 | != 1 |
| `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | 未同步分区数 | > 0 |
| `kafka.server:type=ReplicaManager,name=PartitionCount` | 分区总数 | 监控增长 |
| `kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*,records-lag-max` | 消费者最大延迟 | > 10000 |

### 8.2 常用运维命令

```bash
# 查看 Topic 详情
kafka-topics.sh --describe --topic order-events --bootstrap-server localhost:9092

# 查看消费者组状态
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-group --describe

# 查看日志段文件
kafka-run-class.sh kafka.tools.DumpLogSegments \
  --files /tmp/kafka-logs/order-events-0/00000000000000000000.log \
  --print-data-log

# 重置消费者偏移量
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-group --topic order-events --reset-offsets \
  --to-latest --execute

# 性能测试
kafka-producer-perf-test.sh --topic test-topic \
  --num-records 1000000 --record-size 1024 \
  --throughput -1 --producer-props \
  bootstrap.servers=localhost:9092

kafka-consumer-perf-test.sh --topic test-topic \
  --messages 1000000 --bootstrap-server localhost:9092
```

## 9. 最佳实践

1. **Topic 设计**:
   - 分区数 = max(生产者并发, 消费者并发)
   - 单分区不要超过 10GB，避免重平衡耗时过长
   - 副本因子 = 3（生产环境最低要求）

2. **生产者配置**:
   - 开启幂等性（`enable.idempotence=true`）
   - 使用事务保证跨分区原子性
   - 合理设置 batch.size 和 linger.ms 提升吞吐

3. **消费者配置**:
   - 手动管理偏移量，确保至少处理一次
   - 业务处理逻辑异步化，避免 poll 超时
   - 实现再均衡监听器，优雅处理分区变更

4. **监控告警**:
   - 监控 UnderReplicatedPartitions（>0 立即处理）
   - 监控消费者 Lag（避免消息堆积）
   - 监控 Controller 切换频率

5. **容量规划**:
   - 磁盘: 保留周期 * 日写入量 * 副本数 * 1.2
   - 内存: 活跃分区数 * 1MB (页缓存)
   - 网络: 峰值吞吐 * 副本数 * 1.5

6. **升级维护**:
   - 滚动升级：先升级 Follower，最后升级 Leader
   - 优先使用 Kafka 自带工具进行数据迁移
   - 升级前验证客户端兼容性
