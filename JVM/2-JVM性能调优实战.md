# JVM 性能调优实战

## 1. 调优目标

```
JVM 调优目标:

┌─────────────────────────────────────────────────────────────┐
│                    调优维度                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  延迟 (Latency):                                           │
│  ├── GC 停顿时间最小化                                      │
│  ├── P99/P999 响应时间达标                                   │
│  └── 目标: 单次 GC < 200ms                                  │
│                                                             │
│  吞吐量 (Throughput):                                       │
│  ├── 应用运行时间占比最大化                                  │
│  ├── GC 时间占比最小化                                      │
│  └── 目标: GC 时间占比 < 5%                                 │
│                                                             │
│  内存占用 (Footprint):                                      │
│  ├── 堆内存合理使用                                         │
│  ├── 避免内存泄漏                                           │
│  └── 目标: 堆使用率 < 70%                                   │
│                                                             │
│  调优公式:                                                  │
│  吞吐量 = 应用运行时间 / (应用运行时间 + GC时间)            │
│  目标: 吞吐量 >= 95%                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 内存配置最佳实践

```bash
# 生产环境推荐配置

# 场景1: Web 应用（4G 内存服务器）
-server \
-Xms2g -Xmx2g \
-Xmn768m \
-XX:MetaspaceSize=256m \
-XX:MaxMetaspaceSize=512m \
-XX:SurvivorRatio=8 \
-XX:+UseG1GC \
-XX:MaxGCPauseMillis=200 \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/var/log/app/heapdump.hprof

# 场景2: 高并发服务（8G 内存服务器）
-server \
-Xms6g -Xmx6g \
-Xmn2g \
-XX:MetaspaceSize=512m \
-XX:MaxMetaspaceSize=1g \
-XX:+UseG1GC \
-XX:G1HeapRegionSize=16m \
-XX:MaxGCPauseMillis=100 \
-XX:InitiatingHeapOccupancyPercent=45

# 场景3: 大数据处理（16G 内存服务器）
-server \
-Xms12g -Xmx12g \
-Xmn4g \
-XX:MetaspaceSize=512m \
-XX:MaxMetaspaceSize=1g \
-XX:+UseZGC \
-XX:+ZGenerational
```

## 3. GC 调优步骤

```
GC 调优流程:

1. 监控现状
   ├── 收集 GC 日志
   ├── 分析 GC 频率和耗时
   └── 识别性能瓶颈

2. 确定目标
   ├── 延迟目标（P99 < 200ms）
   ├── 吞吐量目标（> 95%）
   └── 内存使用目标（< 70%）

3. 调整参数
   ├── 调整堆大小
   ├── 调整新生代比例
   ├── 选择合适的收集器
   └── 调整 GC 触发阈值

4. 验证效果
   ├── 压测验证
   ├── 监控 GC 指标
   └── 对比调优前后

5. 持续优化
   ├── 根据业务变化调整
   ├── 定期审查 GC 日志
   └── 关注新版本 GC 特性
```

## 4. 常见问题调优

### 4.1 频繁 Minor GC

```
现象: Minor GC 频繁，每次回收对象少

原因:
- 新生代太小
- 对象分配速率过高
- Survivor 区太小导致过早晋升

调优:
# 增大新生代
-Xmn1g  # 或 -XX:NewRatio=2

# 增大 Survivor 区
-XX:SurvivorRatio=6  # Eden:S0:S1 = 6:1:1

# 提高晋升年龄阈值
-XX:MaxTenuringThreshold=15
```

### 4.2 频繁 Full GC

```
现象: Full GC 频繁，耗时长

原因:
- 老年代空间不足
- 内存泄漏
- 大对象直接进入老年代
- CMS 并发模式失败

调优:
# 增大堆内存
-Xms4g -Xmx4g

# 增大老年代
-XX:NewRatio=3  # 老年代:新生代 = 3:1

# 避免大对象直接进入老年代
-XX:PretenureSizeThreshold=4m

# G1: 调整触发阈值
-XX:InitiatingHeapOccupancyPercent=35
```

### 4.3 GC 停顿时间过长

```
现象: 单次 GC 停顿超过 1 秒

原因:
- 堆太大
- 使用低效收集器
- 对象存活率高

调优:
# 使用低延迟收集器
-XX:+UseZGC  # JDK15+
-XX:+UseG1GC -XX:MaxGCPauseMillis=100

# G1: 减小 Region 大小
-XX:G1HeapRegionSize=8m

# 增加并发标记线程
-XX:ConcGCThreads=4
```

## 5. 调优案例

### 5.1 电商大促场景

```bash
# 业务特点:
# - 流量高峰期 10 倍增长
# - 订单处理需要低延迟
# - 大量临时对象（订单、商品）

# JVM 配置:
-server \
-Xms8g -Xmx8g \
-Xmn3g \
-XX:MetaspaceSize=512m \
-XX:MaxMetaspaceSize=1g \
-XX:+UseG1GC \
-XX:G1HeapRegionSize=16m \
-XX:MaxGCPauseMillis=150 \
-XX:G1ReservePercent=25 \
-XX:InitiatingHeapOccupancyPercent=40 \
-XX:ConcGCThreads=4 \
-XX:+ParallelRefProcEnabled \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/var/log/app/heapdump.hprof \
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags
```

### 5.2 微服务场景

```bash
# 业务特点:
# - 多个独立服务
# - 每个服务内存需求小
# - 快速启动要求

# JVM 配置:
-server \
-Xms512m -Xmx512m \
-Xmn192m \
-XX:MetaspaceSize=128m \
-XX:MaxMetaspaceSize=256m \
-XX:+UseG1GC \
-XX:MaxGCPauseMillis=50 \
-XX:+UseStringDeduplication \
-XX:+UseCompressedOops \
-XX:+TieredCompilation \
-XX:TieredStopAtLevel=3
```

## 6. 性能测试

```bash
# 压测工具
# JMeter
jmeter -n -t test.jmx -l result.jtl

# wrk
wrk -t12 -c400 -d30s http://localhost:8080/api/test

# 监控 GC
jstat -gcutil <pid> 1000

# 分析 GC 日志
# GCEasy: https://gceasy.io
# GCViewer: https://github.com/chewiebug/GCViewer
```

## 7. 调优检查清单

```
JVM 调优 Checklist:

□ 内存配置:
  - Xms 和 Xmx 设置相同
  - 新生代大小合理（堆的 1/3 - 1/4）
  - 元空间有明确上限

□ GC 选择:
  - 低延迟场景用 ZGC/G1
  - 高吞吐场景用 Parallel
  - 大堆(4G+)用 G1

□ GC 日志:
  - 开启 GC 日志记录
  - 日志包含时间戳
  - 配置日志轮转

□ OOM 处理:
  - 开启 HeapDump
  - 指定 Dump 路径
  - 配置 OOM 后动作

□ 监控告警:
  - 监控 GC 频率和耗时
  - 监控堆内存使用率
  - 设置告警阈值
```
