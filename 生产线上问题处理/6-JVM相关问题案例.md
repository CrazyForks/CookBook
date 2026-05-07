# 生产问题案例 - JVM 相关（案例 37-42）

## 案例37: 频繁 Full GC

### 问题现象
- GC 日志显示频繁 Full GC
- 应用响应时间波动大
- CPU 使用率周期性飙升

### 排查过程
```bash
# 查看 GC 情况
jstat -gcutil <pid> 1000

# 输出:
#   S0     S1     E      O      M     CCS    YGC   YGCT    FGC  FGCT     GCT
#   0.00 100.00 100.00  99.98  95.00  92.00   500  15.000  200  80.000  95.000

# 查看 GC 日志
tail -f gc.log | grep "Full GC"

# 生成堆转储
jmap -dump:format=b,file=heap.hprof <pid>
```

### 根因分析
```
问题原因:
1. 老年代空间不足
2. 对象晋升过快
3. 内存泄漏

# 使用 MAT 分析 heap dump
# 查找占用内存最多的对象
# 查看 GC Roots 引用链
```

### 解决方案
```bash
# 方案1: 增大堆内存
-Xms4g -Xmx4g -Xmn2g

# 方案2: 调整 GC 策略
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45

# 方案3: 排查内存泄漏
# 使用 MAT 分析 heap dump
# 找到泄漏代码并修复

# 方案4: 调整新生代比例
-XX:NewRatio=2  # 老年代:新生代 = 2:1
-XX:SurvivorRatio=8  # Eden:S0:S1 = 8:1:1
```

---

## 案例38: GC 停顿过长

### 问题现象
- 接口偶发超时
- GC 停顿时间超过 1 秒
- 用户体验差

### 排查过程
```bash
# 查看 GC 停顿时间
jstat -gcutil <pid> 1000

# GC 日志分析
# [GC pause (G1 Evacuation Pause) (young) 1024M->512M(4096M), 1.500 secs]

# 使用 GCViewer 分析
# 或使用 GCEasy 在线分析
```

### 根因分析
```
问题原因:
1. 堆太大，GC 扫描时间长
2. 使用低效的 GC 收集器
3. 对象存活率高

# G1 GC 停顿时间与堆大小相关
# 堆越大，停顿时间越长
```

### 解决方案
```bash
# 方案1: 使用低延迟 GC
# ZGC（JDK 15+）
-XX:+UseZGC

# Shenandoah
-XX:+UseShenandoahGC

# 方案2: G1 调优
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100  # 降低目标停顿
-XX:G1HeapRegionSize=8m   # 减小 Region 大小
-XX:ConcGCThreads=4       # 增加并发线程

# 方案3: 减小堆大小
-Xms2g -Xmx2g

# 方案4: 使用 ZGC（推荐）
-XX:+UseZGC
-XX:+ZGenerational  # JDK 21+
```

---

## 案例39: 类加载冲突

### 问题现象
- 报错 ClassCastException
- 类版本不一致
- NoSuchMethodError

### 排查过程
```bash
# 查看类加载路径
-verbose:class

# 查看类来源
jmap -clstats <pid>

# 查找冲突的 jar
mvn dependency:tree | grep "conflict"
```

### 根因分析
```
问题原因:
1. 依赖冲突，同一个类有多个版本
2. 类加载器隔离问题
3. 容器与应用类冲突

# 常见冲突:
# - Spring 版本冲突
# - Jackson 版本冲突
# - 日志框架冲突
```

### 解决方案
```xml
<!-- 方案1: 排除冲突依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 方案2: 统一版本管理 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson</groupId>
            <artifactId>jackson-bom</artifactId>
            <version>2.17.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

```bash
# 方案3: 使用 Maven 插件分析冲突
mvn enforcer:enforce

# 方案4: 使用 shade 插件重定位
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <relocations>
                <relocation>
                    <pattern>com.google.common</pattern>
                    <shadedPattern>shaded.com.google.common</shadedPattern>
                </relocation>
            </relocations>
        </execution>
    </executions>
</plugin>
```

---

## 案例40: 线程死锁

### 问题现象
- 应用无响应
- 线程状态全是 BLOCKED
- CPU 使用率低但无法处理请求

### 排查过程
```bash
# 获取线程堆栈
jstack <pid> > thread.dump

# 查找死锁
jstack -l <pid> | grep -A 20 "Found.*deadlock"

# 使用 arthas
thread -b

# 输出:
# "http-nio-8080-exec-1" BLOCKED on lock held by "http-nio-8080-exec-2"
# "http-nio-8080-exec-2" BLOCKED on lock held by "http-nio-8080-exec-1"
```

### 根因分析
```java
// 问题代码: 嵌套锁
public class DeadlockExample {
    private final Object lockA = new Object();
    private final Object lockB = new Object();
    
    public void method1() {
        synchronized (lockA) {
            // 操作 A
            synchronized (lockB) {
                // 操作 B
            }
        }
    }
    
    public void method2() {
        synchronized (lockB) {
            // 操作 B
            synchronized (lockA) {
                // 操作 A
            }
        }
    }
}
```

### 解决方案
```java
// 方案1: 统一加锁顺序
public void method1() {
    synchronized (lockA) {
        synchronized (lockB) {
            // 操作
        }
    }
}

public void method2() {
    synchronized (lockA) {  // 也先锁 A
        synchronized (lockB) {
            // 操作
        }
    }
}

// 方案2: 使用 tryLock（超时释放）
private final Lock lockA = new ReentrantLock();
private final Lock lockB = new ReentrantLock();

public void method1() {
    try {
        if (lockA.tryLock(1, TimeUnit.SECONDS)) {
            try {
                if (lockB.tryLock(1, TimeUnit.SECONDS)) {
                    // 操作
                }
            } finally {
                lockB.unlock();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        lockA.unlock();
    }
}

// 方案3: 使用无锁数据结构
// ConcurrentHashMap, AtomicInteger 等
```

---

## 案例41: 线程池耗尽

### 问题现象
- 报错 "RejectedExecutionException"
- 任务堆积
- 接口超时

### 排查过程
```bash
# 查看线程池状态
# 使用 actuator
curl http://localhost:8080/actuator/metrics/executors

# 查看线程堆栈
jstack <pid> | grep "pool-"
```

### 核因分析
```java
// 问题代码: 线程池配置不当
ExecutorService executor = new ThreadPoolExecutor(
    5, 5, 0, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(10)  // 队列太小
);

// 任务提交速度 > 处理速度
// 队列满后触发拒绝策略
```

### 解决方案
```java
// 方案1: 合理配置线程池
ExecutorService executor = new ThreadPoolExecutor(
    10,                                      // 核心线程数
    50,                                      // 最大线程数
    60, TimeUnit.SECONDS,                    // 空闲线程存活时间
    new LinkedBlockingQueue<>(1000),         // 有界队列
    new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略
);

// 方案2: 监控线程池
@Bean("taskExecutor")
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(1000);
    executor.setThreadNamePrefix("task-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.setWaitForTasksToCompleteOnShutdown(true);
    executor.setAwaitTerminationSeconds(60);
    return executor;
}

// 方案3: 使用信号量限流
private final Semaphore semaphore = new Semaphore(100);

public void submitTask(Runnable task) {
    if (semaphore.tryAcquire()) {
        executor.submit(() -> {
            try {
                task.run();
            } finally {
                semaphore.release();
            }
        });
    } else {
        // 降级处理
    }
}
```

---

## 案例42: 栈溢出

### 问题现象
- 报错 StackOverflowError
- 递归调用过深
- 应用崩溃

### 排查过程
```bash
# 查看线程堆栈
jstack <pid> | grep -A 100 "StackOverflowError"

# 输出:
# java.lang.StackOverflowError
#     at com.example.service.RecursiveService.process(RecursiveService.java:25)
#     at com.example.service.RecursiveService.process(RecursiveService.java:25)
#     at com.example.service.RecursiveService.process(RecursiveService.java:25)
#     ... (重复多次)
```

### 根因分析
```java
// 问题代码: 递归没有终止条件
public void process(Node node) {
    if (node == null) {
        return;
    }
    process(node.getChild());  // 可能无限递归
}

// 或者递归深度太大
public void deepRecursion(int n) {
    if (n <= 0) return;
    deepRecursion(n - 1);  // n 很大时栈溢出
}
```

### 解决方案
```java
// 方案1: 使用循环代替递归
public void process(Node node) {
    while (node != null) {
        // 处理节点
        node = node.getChild();
    }
}

// 方案2: 使用栈数据结构
public void process(Node root) {
    Deque<Node> stack = new ArrayDeque<>();
    stack.push(root);
    
    while (!stack.isEmpty()) {
        Node node = stack.pop();
        // 处理节点
        if (node.getChild() != null) {
            stack.push(node.getChild());
        }
    }
}

// 方案3: 增加栈大小
-Xss2m  # 默认 512k-1m

// 方案4: 设置递归深度限制
private static final int MAX_DEPTH = 1000;

public void process(Node node, int depth) {
    if (depth > MAX_DEPTH) {
        throw new RuntimeException("递归深度超限");
    }
    // 递归逻辑
}
```

---

## JVM 问题预防 Checklist

```
JVM 问题预防:

□ 内存配置:
  - 设置合理的堆大小
  - 选择合适的 GC 收集器
  - 配置 OOM 时 dump

□ 监控告警:
  - GC 频率监控
  - GC 停顿时间监控
  - 线程数监控
  - 类加载监控

□ 代码规范:
  - 避免内存泄漏
  - 避免死锁
  - 合理使用线程池
  - 避免无限递归

□ 运维规范:
  - 定期分析 GC 日志
  - 定期分析线程堆栈
  - 定期分析堆转储
  - JVM 参数调优
```
