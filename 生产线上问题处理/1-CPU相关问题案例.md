# 生产问题案例 - CPU 相关（案例 1-8）

## 案例1: 正则表达式导致 CPU 飙高

### 问题现象
- 生产环境 CPU 持续 100%
- 接口响应超时
- 重启后几分钟再次出现

### 排查过程
```bash
# 1. 查看 CPU 使用率高的进程
top -c

# 2. 查看 CPU 使用率高的线程
top -Hp <pid>

# 3. 转换线程 ID 为十六进制
printf "%x\n" <tid>

# 4. 获取线程堆栈
jstack <pid> | grep <tid_hex> -A 30

# 堆栈信息:
# "http-nio-8080-exec-1" #15 daemon prio=5 os_prio=0 tid=0x00007f8b4c012800 nid=0x1234 runnable [0x00007f8b52000000]
#    java.lang.Thread.State: RUNNABLE
#         at java.util.regex.Pattern$Loop.match(Pattern.java:4785)
#         at java.util.regex.Pattern$GroupTail.match(Pattern.java:4717)
#         at java.util.regex.Pattern$Curly.match0(Pattern.java:4271)
#         at java.util.regex.Pattern$Curly.match(Pattern.java:4234)
```

### 根因分析
```java
// 问题代码: 使用了灾难性回溯的正则表达式
String regex = "^(a+)+$";
String input = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaab";
boolean matches = input.matches(regex);  // CPU 100%

// 原因: (a+)+$ 存在指数级回溯
// 当输入不匹配时，正则引擎会尝试所有可能的分组组合
```

### 解决方案
```java
// 方案1: 优化正则表达式
String regex = "^a+$";  // 简化正则

// 方案2: 使用预编译的 Pattern
private static final Pattern PATTERN = Pattern.compile("^a+$");
boolean matches = PATTERN.matcher(input).matches();

// 方案3: 使用非贪婪匹配
String regex = "^(a+?)++$";

// 方案4: 设置超时（Java 9+）
import java.util.regex.Matcher;
import java.util.regex.Pattern;

Pattern pattern = Pattern.compile(regex);
Matcher matcher = pattern.matcher(input);
// 使用 find() 代替 matches()，并限制匹配时间
```

### 预防措施
- 避免使用嵌套量词 `(a+)+`
- 使用正则测试工具验证
- 对用户输入的正则进行校验
- 监控 CPU 使用率，设置告警

---

## 案例2: 死循环导致 CPU 100%

### 问题现象
- 服务启动后 CPU 快速飙升到 100%
- 接口全部超时
- 重启后问题依旧

### 排查过程
```bash
# jstack 获取线程堆栈
jstack <pid> > thread.dump

# 查找 RUNNABLE 状态的线程
grep -A 20 "RUNNABLE" thread.dump

# 堆栈信息:
# "Thread-0" #10 daemon prio=5 os_prio=0 tid=0x00007f8b4c012800 nid=0x1234 runnable [0x00007f8b52000000]
#    java.lang.Thread.State: RUNNABLE
#         at com.example.service.DataProcessor.process(DataProcessor.java:45)
#         at com.example.service.DataProcessor.run(DataProcessor.java:30)
```

### 根因分析
```java
// 问题代码: while 循环条件永远为真
public void process() {
    List<Task> tasks = getTasks();
    Iterator<Task> iterator = tasks.iterator();
    
    while (iterator.hasNext()) {  // 错误: iterator 没有 next()
        Task task = iterator.next();
        if (task.isValid()) {
            processTask(task);
        }
        // 忘记调用 iterator.next()，导致死循环
    }
}

// 另一个常见场景: 集合在遍历时被修改
public void process() {
    List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
    for (String item : list) {
        if ("b".equals(item)) {
            list.remove(item);  // ConcurrentModificationException 或死循环
        }
    }
}
```

### 解决方案
```java
// 方案1: 修复循环逻辑
public void process() {
    List<Task> tasks = getTasks();
    for (Task task : tasks) {
        if (task.isValid()) {
            processTask(task);
        }
    }
}

// 方案2: 使用安全的删除方式
public void process() {
    List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {
        String item = iterator.next();
        if ("b".equals(item)) {
            iterator.remove();  // 使用 iterator.remove()
        }
    }
}

// 方案3: 使用 removeIf
public void process() {
    List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
    list.removeIf(item -> "b".equals(item));
}
```

---

## 案例3: 频繁 Full GC 导致 CPU 飙高

### 问题现象
- CPU 使用率周期性飙升
- 接口响应时间波动大
- GC 日志显示频繁 Full GC

### 排查过程
```bash
# 查看 GC 情况
jstat -gcutil <pid> 1000

# 输出:
#   S0     S1     E      O      M     CCS    YGC   YGCT    FGC  FGCT     GCT
#   0.00 100.00 100.00  99.98  95.00  92.00   500  15.000  200  80.000  95.000

# 查看 GC 日志
tail -f gc.log | grep "Full GC"

# 使用 arthas 查看
dashboard
```

### 根因分析
```
问题原因:
1. 老年代空间不足
2. 对象晋升过快
3. 内存泄漏导致老年代持续增长

# 生成堆转储
jmap -dump:format=b,file=heap.hprof <pid>

# 使用 MAT 分析
# 找到占用内存最多的对象
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
# 找到 GC Roots 引用链
# 修复泄漏代码
```

---

## 案例4: 序列化/反序列化 CPU 飙高

### 问题现象
- 大量 JSON 数据处理时 CPU 飙高
- 接口处理大对象时超时

### 排查过程
```bash
# arthas 分析方法耗时
trace com.fasterxml.jackson.databind.ObjectMapper readValue -n 500

# 输出:
# `---ts=2024-01-01 12:00:00;thread_name=http-nio-8080-exec-1;cost=1500ms`
#     `---com.fasterxml.jackson.databind.ObjectMapper.readValue`
#         `---com.fasterxml.jackson.databind.deser.BeanDeserializer.deserialize`
```

### 根因分析
```java
// 问题代码: 每次请求都创建 ObjectMapper
@PostMapping("/process")
public Result process(@RequestBody String json) {
    ObjectMapper mapper = new ObjectMapper();  // 每次创建新实例
    Data data = mapper.readValue(json, Data.class);
    return process(data);
}

// ObjectMapper 创建成本高，且线程不安全
```

### 解决方案
```java
// 方案1: 单例复用 ObjectMapper
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
}

// 方案2: 使用 Spring 注入
@Autowired
private ObjectMapper objectMapper;

// 方案3: 对于大对象，使用流式 API
public void processLargeJson(InputStream is) {
    JsonFactory factory = new JsonFactory();
    try (JsonParser parser = factory.createParser(is)) {
        // 流式处理
    }
}
```

---

## 案例5: 加密解密 CPU 飙高

### 问题现象
- 接口加密解密时 CPU 飙高
- 大量请求时服务响应慢

### 排查过程
```bash
# 查看线程堆栈
jstack <pid> | grep -A 10 "RUNNABLE" | head -50

# 堆栈信息:
# at javax.crypto.Cipher.doFinal(Cipher.java:2069)
# at com.example.util.EncryptUtil.encrypt(EncryptUtil.java:25)
```

### 根因分析
```java
// 问题代码: 每次加密都初始化 Cipher
public String encrypt(String data) {
    Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");  // 每次创建
    SecretKeySpec keySpec = new SecretKeySpec(key.getBytes(), "AES");
    cipher.init(Cipher.ENCRYPT_MODE, keySpec);
    byte[] encrypted = cipher.doFinal(data.getBytes());
    return Base64.getEncoder().encodeToString(encrypted);
}
```

### 解决方案
```java
// 方案1: 使用 ThreadLocal 缓存 Cipher
private static final ThreadLocal<Cipher> ENCRYPT_CIPHER = ThreadLocal.withInitial(() -> {
    try {
        Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
        cipher.init(Cipher.ENCRYPT_MODE, keySpec, ivSpec);
        return cipher;
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
});

public String encrypt(String data) {
    Cipher cipher = ENCRYPT_CIPHER.get();
    byte[] encrypted = cipher.doFinal(data.getBytes());
    return Base64.getEncoder().encodeToString(encrypted);
}

// 方案2: 使用加密池
private final BlockingQueue<Cipher> cipherPool = new LinkedBlockingQueue<>(10);
```

---

## 案例6: JSON 解析 CPU 飙高

### 问题现象
- 大量 JSON 数据解析时 CPU 飙高
- FastJSON 解析复杂对象时特别慢

### 排查过程
```bash
# arthas trace
trace com.alibaba.fastjson.JSON parseObject -n 100

# 堆栈信息:
# `---cost=800ms`
#     `---com.alibaba.fastjson.JSON.parseObject`
#         `---com.alibaba.fastjson.parser.DefaultJSONParser.parse`
```

### 根因分析
```java
// 问题代码: 使用 FastJSON 的 autoType 功能
String json = "{\"@type\":\"com.example.User\",\"name\":\"张三\"}";
Object obj = JSON.parse(json);  // autoType 开启，安全风险且性能差

// FastJSON 1.x 的 autoType 有安全漏洞和性能问题
```

### 解决方案
```java
// 方案1: 升级到 FastJSON2
import com.alibaba.fastjson2.JSON;
Object obj = JSON.parse(json);

// 方案2: 关闭 autoType
// JVM 参数
-Dfastjson.parser.autoTypeSupport=false

// 方案3: 使用 Jackson
ObjectMapper mapper = new ObjectMapper();
User user = mapper.readValue(json, User.class);

// 方案4: 预编译 TypeReference
private static final TypeReference<List<User>> USER_LIST_TYPE = new TypeReference<List<User>>() {};
List<User> users = JSON.parseArray(json, User.class);
```

---

## 案例7: 日志框架锁竞争 CPU 飙高

### 问题现象
- 高并发时 CPU 飙高
- 线程堆栈显示大量 BLOCKED 状态
- 日志输出时特别慢

### 排查过程
```bash
# jstack 查看线程状态
jstack <pid> | grep "BLOCKED" | wc -l

# 堆栈信息:
# "http-nio-8080-exec-1" #15 daemon prio=5 os_prio=0 tid=0x00007f8b4c012800 nid=0x1234 waiting for monitor entry [0x00007f8b52000000]
#    java.lang.Thread.State: BLOCKED (on object monitor)
#         at ch.qos.logback.core.OutputStreamAppender.subAppend(OutputStreamAppender.java:218)
#         - waiting to lock <0x000000076ab2b780> (a ch.qos.logback.core.OutputStreamAppender)
```

### 根因分析
```
问题原因:
1. 所有线程竞争同一个日志 Appender 的锁
2. 日志输出到文件时，需要同步写入
3. 高并发时锁竞争激烈

# Logback 默认使用同步 Appender
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>app.log</file>
    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
</appender>
```

### 解决方案
```xml
<!-- 方案1: 使用异步 Appender -->
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>1024</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <appender-ref ref="FILE" />
</appender>

<!-- 方案2: 使用 Log4j2 异步日志 -->
<!-- 依赖 LMAX Disruptor -->
<dependency>
    <groupId>com.lmax</groupId>
    <artifactId>disruptor</artifactId>
    <version>3.4.4</version>
</dependency>

<!-- 方案3: 减少日志级别 -->
<logger name="com.example" level="INFO" />
```

---

## 案例8: ThreadLocal 内存泄漏导致 CPU 飙高

### 问题现象
- 服务运行一段时间后 CPU 飙高
- 内存使用率持续上升
- Full GC 频繁

### 排查过程
```bash
# 生成堆转储
jmap -dump:format=b,file=heap.hprof <pid>

# 使用 MAT 分析
# 查找 ThreadLocal 实例数量
# 查找 ThreadLocalMap 中的 Entry
```

### 根因分析
```java
// 问题代码: ThreadLocal 使用后未清理
public class UserContext {
    private static final ThreadLocal<User> USER_HOLDER = new ThreadLocal<>();
    
    public static void setUser(User user) {
        USER_HOLDER.set(user);
    }
    
    public static User getUser() {
        return USER_HOLDER.get();
    }
    
    // 缺少 remove() 方法
}

// 在 Web 应用中，线程池复用线程
// ThreadLocal 中的对象不会被 GC 回收
// 导致内存泄漏
```

### 解决方案
```java
// 方案1: 使用后清理
public class UserContext {
    private static final ThreadLocal<User> USER_HOLDER = new ThreadLocal<>();
    
    public static void setUser(User user) {
        USER_HOLDER.set(user);
    }
    
    public static User getUser() {
        return USER_HOLDER.get();
    }
    
    public static void clear() {
        USER_HOLDER.remove();  // 使用后清理
    }
}

// 在 Filter 或 Interceptor 中清理
@Override
public void afterCompletion(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler, Exception ex) {
    UserContext.clear();
}

// 方案2: 使用 InheritableThreadLocal（注意线程池问题）
// 方案3: 使用 TransmittableThreadLocal（阿里开源）
```

---

## 问题预防 Checklist

```
CPU 问题预防:

□ 代码审查:
  - 检查正则表达式是否有灾难性回溯
  - 检查循环是否有正确的退出条件
  - 检查序列化是否有性能问题

□ 监控告警:
  - CPU 使用率 > 80% 告警
  - GC 频率超过阈值告警
  - 线程数超过阈值告警

□ 性能测试:
  - 压测验证 CPU 使用率
  - 验证高并发下的性能
  - 验证大对象处理性能

□ 代码规范:
  - ThreadLocal 必须在 finally 中清理
  - ObjectMapper 单例复用
  - 使用异步日志
```
