# JVM 监控与故障排查

## 1. 监控工具概览

```
JVM 监控工具体系:

┌─────────────────────────────────────────────────────────────┐
│                    JVM 监控工具                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  命令行工具:                                                │
│  ├── jps:      查看 Java 进程                              │
│  ├── jstat:    GC 统计信息                                 │
│  ├── jinfo:    查看/修改 JVM 参数                          │
│  ├── jmap:     堆转储                                      │
│  ├── jstack:   线程转储                                    │
│  └── jhat:     分析堆转储                                  │
│                                                             │
│  可视化工具:                                                │
│  ├── JConsole:     JDK 自带监控工具                        │
│  ├── VisualVM:     功能强大的监控工具                      │
│  ├── JMC:          Java Mission Control                    │
│  └── Arthas:       阿里开源在线诊断工具                    │
│                                                             │
│  APM 工具:                                                  │
│  ├── SkyWalking:   链路追踪                                │
│  ├── Pinpoint:     APM 监控                                │
│  └── Prometheus+Grafana: 指标监控                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 命令行工具

### 2.1 jps - 查看 Java 进程

```bash
# 查看所有 Java 进程
jps

# 显示完整类名
jps -l

# 显示 JVM 参数
jps -v

# 输出示例:
# 12345 Bootstrap
# 12346 Jps -l
```

### 2.2 jstat - GC 统计

```bash
# 查看 GC 统计（每秒输出一次，共10次）
jstat -gcutil <pid> 1000 10

# 输出说明:
# S0     S1     E      O      M     CCS    YGC   YGCT    FGC  FGCT     GCT
# 0.00  25.00  45.00  30.00  95.00  92.00   100   1.500    5   0.800   2.300
# S0/S1: Survivor 区使用率
# E: Eden 区使用率
# O: 老年代使用率
# M: 元空间使用率
# YGC/YGCT: Young GC 次数/耗时
# FGC/FGCT: Full GC 次数/耗时
# GCT: 总 GC 耗时

# 查看堆内存统计
jstat -gc <pid> 1000

# 查看类加载统计
jstat -class <pid>

# 查看编译统计
jstat -compiler <pid>
```

### 2.3 jmap - 堆转储

```bash
# 查看堆内存使用
jmap -heap <pid>

# 查看对象统计
jmap -histo <pid> | head -20

# 生成堆转储文件
jmap -dump:format=b,file=heap.hprof <pid>

# OOM 时自动生成堆转储
# JVM 参数: -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/
```

### 2.4 jstack - 线程转储

```bash
# 生成线程转储
jstack <pid> > thread.dump

# 查找死锁
jstack -l <pid>

# 输出说明:
# "main" #1 prio=5 os_prio=0 tid=0x00007f8b4c00a000 nid=0x1234 waiting on condition [0x00007f8b52000000]
#    java.lang.Thread.State: WAITING (parking)
#         at sun.misc.Unsafe.park(Native Method)
#         - parking to wait for <0x000000076ab2b780> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
```

## 3. Arthas 在线诊断

### 3.1 安装与启动

```bash
# 下载 Arthas
curl -O https://arthas.aliyun.com/arthas-boot.jar

# 启动（选择 Java 进程）
java -jar arthas-boot.jar

# 或指定进程 ID
java -jar arthas-boot.jar <pid>
```

### 3.2 常用命令

```bash
# 查看仪表盘
dashboard

# 查看线程信息
thread
thread -n 3        # 最忙的 3 个线程
thread -b          # 查找阻塞线程

# 查看 JVM 信息
jvm
memory

# 查看类信息
sc com.example.UserService
sm com.example.UserService

# 监控方法调用
watch com.example.UserService getUser '{params, returnObj}' -x 2

# 追踪方法调用链
trace com.example.UserService getUser

# 反编译类
jad com.example.UserService

# 热更新代码
mc /tmp/UserService.java
redefine /tmp/UserService.class

# 查看方法耗时
monitor com.example.UserService getUser -c 5
```

## 4. 故障排查流程

### 4.1 CPU 飙高排查

```bash
# 1. 查找 CPU 高的进程
top -c

# 2. 查找 CPU 高的线程
top -Hp <pid>

# 3. 转换线程 ID 为十六进制
printf "%x\n" <tid>

# 4. 查找线程堆栈
jstack <pid> | grep <tid_hex> -A 30

# 5. 分析堆栈
# 常见原因:
# - 死循环
# - 频繁 GC
# - 锁竞争
```

### 4.2 内存泄漏排查

```bash
# 1. 监控堆内存使用
jstat -gcutil <pid> 1000

# 2. 生成堆转储
jmap -dump:format=b,file=heap.hprof <pid>

# 3. 分析堆转储
# 使用 MAT (Memory Analyzer Tool)
# 或 VisualVM

# 4. 查找大对象
jmap -histo <pid> | head -20

# 5. 分析泄漏点
# - 查找占用内存最多的对象
# - 分析对象引用链
# - 检查集合类是否正确清理
```

### 4.3 线程死锁排查

```bash
# 1. 生成线程转储
jstack <pid> > thread.dump

# 2. 查找死锁
jstack -l <pid> | grep -A 20 "Found.*deadlock"

# 3. 使用 Arthas
thread -b

# 4. 分析死锁原因
# - 检查 synchronized 嵌套
# - 检查 Lock 使用顺序
# - 检查数据库锁等待
```

## 5. 监控指标

```
关键监控指标:

┌──────────────────────┬───────────────┬───────────────────────┐
│ 指标                 │ 告警阈值      │ 说明                  │
├──────────────────────┼───────────────┼───────────────────────┤
│ 堆使用率             │ > 85%         │ 接近 OOM 风险         │
│ 老年代使用率         │ > 80%         │ 可能触发 Full GC      │
│ Full GC 频率         │ > 1次/小时    │ 需要调优              │
│ Full GC 耗时         │ > 1s          │ 影响响应时间          │
│ Young GC 频率        │ > 10次/分钟   │ 新生代可能太小        │
│ 线程数               │ > 500         │ 可能线程泄漏          │
│ 类加载数             │ 持续增长      │ 可能类加载泄漏        │
└──────────────────────┴───────────────┴───────────────────────┘
```

## 6. Prometheus + Grafana 监控

```yaml
# JVM 监控配置
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    tags:
      application: ${spring.application.name}

# Prometheus 配置
scrape_configs:
  - job_name: 'jvm'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']
```

```
Grafana JVM 面板:
- JVM Memory Pool Usage
- GC Pause Time
- GC Frequency
- Thread Count
- Class Loading
```

## 7. 排查检查清单

```
故障排查 Checklist:

□ CPU 飙高:
  - top -Hp 查看线程
  - jstack 分析堆栈
  - 检查死循环/频繁GC

□ 内存问题:
  - jstat 监控 GC
  - jmap 生成堆转储
  - MAT 分析泄漏点

□ 线程问题:
  - jstack 生成线程转储
  - 检查死锁/阻塞
  - 检查线程池配置

□ GC 问题:
  - 分析 GC 日志
  - 检查堆内存配置
  - 选择合适收集器

□ 应用卡顿:
  - 检查 GC 停顿
  - 检查锁竞争
  - 检查外部依赖
```
