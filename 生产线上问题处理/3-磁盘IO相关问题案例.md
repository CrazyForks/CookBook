# 生产问题案例 - 磁盘/IO 相关（案例 17-22）

## 案例17: 日志文件打满磁盘

### 问题现象
- 服务突然无法写入日志
- 应用报错 "No space left on device"
- 磁盘使用率 100%

### 排查过程
```bash
# 查看磁盘使用率
df -h

# 输出:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   50G    0 100% /

# 查找大文件
du -sh /var/log/* | sort -rh | head -10

# 输出:
# 45G    /var/log/application
# 5G     /var/log/nginx

# 查找具体大文件
find /var/log -type f -size +100M -exec ls -lh {} \;
```

### 根因分析
```
问题原因:
1. 日志级别设置为 DEBUG，输出大量日志
2. 日志轮转配置不当，历史日志未清理
3. 错误日志异常频繁（如循环打印异常）

# 查看日志配置
cat logback.xml
# <root level="DEBUG">
#   <appender-ref ref="FILE" />
# </root>
```

### 解决方案
```xml
<!-- 方案1: 配置日志轮转 -->
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>30</maxHistory>
        <totalSizeCap>5GB</totalSizeCap>
    </rollingPolicy>
</appender>

<!-- 方案2: 生产环境使用 INFO 级别 -->
<root level="INFO">
    <appender-ref ref="FILE" />
</root>
```

```bash
# 方案3: 定时清理脚本
# cleanup-logs.sh
find /var/log -name "*.log.*" -mtime +7 -exec rm -f {} \;
find /var/log -name "*.gz" -mtime +30 -exec rm -f {} \;

# crontab
0 2 * * * /opt/scripts/cleanup-logs.sh
```

---

## 案例18: 临时文件未清理导致磁盘满

### 问题现象
- 服务运行一段时间后磁盘满
- /tmp 目录占用大量空间
- 重启后恢复，过段时间再次出现

### 排查过程
```bash
# 查看 /tmp 目录大小
du -sh /tmp

# 查找临时文件
ls -lh /tmp | head -20

# 查找未删除的临时文件
lsof +L1 | grep deleted
```

### 根因分析
```java
// 问题代码: 创建临时文件未删除
public void processFile(MultipartFile file) {
    File tempFile = File.createTempFile("upload-", ".tmp");
    file.transferTo(tempFile);
    
    // 处理文件
    processTempFile(tempFile);
    
    // 忘记删除临时文件
    // tempFile.delete();
}
```

### 解决方案
```java
// 方案1: 使用 try-with-resources 或 finally
public void processFile(MultipartFile file) {
    File tempFile = File.createTempFile("upload-", ".tmp");
    try {
        file.transferTo(tempFile);
        processTempFile(tempFile);
    } finally {
        tempFile.delete();  // 确保删除
    }
}

// 方案2: 使用 Spring 的临时文件清理
@Bean
public MultipartConfigElement multipartConfigElement() {
    MultipartConfigFactory factory = new MultipartConfigFactory();
    factory.setLocation("/tmp");
    factory.setMaxFileSize(DataSize.ofMegabytes(10));
    return factory.createMultipartConfig();
}

// 方案3: 定时清理临时文件
@Scheduled(cron = "0 0 3 * * ?")  // 每天凌晨3点
public void cleanupTempFiles() {
    File tempDir = new File(System.getProperty("java.io.tmpdir"));
    File[] tempFiles = tempDir.listFiles((dir, name) -> name.startsWith("upload-"));
    if (tempFiles != null) {
        for (File file : tempFiles) {
            if (file.lastModified() < System.currentTimeMillis() - 86400000) {
                file.delete();
            }
        }
    }
}
```

---

## 案例19: inode 耗尽导致无法写入

### 问题现象
- 磁盘空间充足，但无法创建新文件
- 报错 "No space left on device"
- df -h 显示有空间

### 排查过程
```bash
# 查看 inode 使用率
df -i

# 输出:
# Filesystem      Inodes  IUsed   IFree IUse% Mounted on
# /dev/sda1      3276800 3276800      0  100% /

# 查找小文件多的目录
find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -20
```

### 根因分析
```
问题原因:
1. 大量小文件占用了所有 inode
2. 邮件队列、临时文件、日志文件等

# 常见场景:
# - /var/spool/postfix/maildrop 大量邮件队列文件
# - /var/spool/clientmqueue 大量邮件队列
# - Session 文件过多
```

### 解决方案
```bash
# 方案1: 删除大量小文件
# 删除 30 天前的文件
find /var/spool/postfix/maildrop -type f -mtime +30 -delete

# 方案2: 重新格式化磁盘（增加 inode）
# 备份数据后
mkfs.ext4 -i 4096 /dev/sda1  # 每 4KB 分配一个 inode

# 方案3: 使用 XFS 文件系统（动态 inode）
mkfs.xfs /dev/sda1

# 方案4: 定时清理
# crontab
0 4 * * * find /var/spool/postfix/maildrop -type f -mtime +7 -delete
```

---

## 案例20: IO 等待导致服务响应慢

### 问题现象
- 服务响应时间变长
- CPU 使用率不高，但 iowait 高
- 接口超时

### 排查过程
```bash
# 查看 IO 状态
iostat -x 1

# 输出:
# Device  rrqm/s wrqm/s  r/s   w/s  rMB/s  wMB/s avgrq-sz avgqu-sz await r_await w_await svctm %util
# sda       0.00  100.00  0.00 500.00   0.00  10.00    40.00    10.00 20.00    0.00   20.00   2.00 100.00

# 查看 IO 高的进程
iotop -o

# 查看文件系统缓存
free -h
```

### 根因分析
```
问题原因:
1. 频繁读写小文件
2. 数据库查询产生大量磁盘 IO
3. 日志同步写入

# 查看是否使用了 O_DIRECT
# 查看文件系统是否使用了 noatime
mount | grep sda
```

### 解决方案
```bash
# 方案1: 使用 SSD
# 将热数据放在 SSD 上

# 方案2: 增加文件系统缓存
echo 3 > /proc/sys/vm/drop_caches  # 清理缓存（临时）
# 增加内存

# 方案3: 使用 noatime 挂载
mount -o remount,noatime /

# 方案4: 数据库配置优化
# innodb_flush_method = O_DIRECT
# innodb_io_capacity = 2000
```

```java
// 方案5: 使用缓冲写入
try (BufferedWriter writer = new BufferedWriter(new FileWriter(file))) {
    writer.write(data);
}

// 方案6: 使用 NIO
try (FileChannel channel = FileChannel.open(path, StandardOpenOption.WRITE)) {
    ByteBuffer buffer = ByteBuffer.wrap(data.getBytes());
    channel.write(buffer);
}
```

---

## 案例21: 文件句柄泄漏

### 问题现象
- 服务运行一段时间后报错 "Too many open files"
- 无法打开新文件或建立新连接
- 重启后恢复

### 排查过程
```bash
# 查看进程打开的文件数
lsof -p <pid> | wc -l

# 查看系统限制
ulimit -n

# 查看进程限制
cat /proc/<pid>/limits | grep "Max open files"

# 查看打开的文件类型
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn
```

### 根因分析
```java
// 问题代码: 文件流未关闭
public void readFiles(List<String> paths) {
    for (String path : paths) {
        FileInputStream fis = new FileInputStream(path);
        // 处理文件
        // 未关闭 fis
    }
}

// 另一个问题: Socket 连接未关闭
public void connect(String host, int port) {
    Socket socket = new Socket(host, port);
    // 使用 socket
    // 未关闭 socket
}
```

### 解决方案
```java
// 方案1: 使用 try-with-resources
public void readFiles(List<String> paths) {
    for (String path : paths) {
        try (FileInputStream fis = new FileInputStream(path)) {
            // 处理文件
        }
    }
}

// 方案2: 使用连接池
// 数据库连接池、HTTP 连接池
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
cm.setMaxTotal(200);
cm.setDefaultMaxPerRoute(20);

// 方案3: 增加文件句柄限制
# /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535

# /etc/sysctl.conf
fs.file-max = 2097152
```

---

## 案例22: 磁盘阵列故障

### 问题现象
- 磁盘 IO 性能突然下降
- RAID 降级警告
- 磁盘错误日志

### 排查过程
```bash
# 查看 RAID 状态
cat /proc/mdstat

# 查看磁盘健康状态
smartctl -a /dev/sda

# 查看磁盘错误
dmesg | grep -i error

# 查看磁盘 IO 统计
iostat -x 1
```

### 根因分析
```
问题原因:
1. 磁盘硬件故障
2. RAID 阵列降级
3. 磁盘坏道

# RAID 5 允许一块磁盘故障
# RAID 1 允许一块磁盘故障
# RAID 0 无冗余
```

### 解决方案
```bash
# 方案1: 更换故障磁盘
mdadm /dev/md0 --remove /dev/sdb1
mdadm /dev/md0 --add /dev/sdc1

# 方案2: 监控磁盘健康
# smartctl 定期检查
smartctl -H /dev/sda

# 方案3: 使用 RAID 10
# 兼顾性能和冗余

# 方案4: 使用分布式存储
# Ceph、GlusterFS 等
```

---

## 磁盘问题预防 Checklist

```
磁盘问题预防:

□ 监控告警:
  - 磁盘使用率 > 80% 告警
  - inode 使用率 > 80% 告警
  - IO 等待 > 30% 告警
  - 磁盘健康状态检查

□ 配置优化:
  - 配置日志轮转
  - 设置临时文件清理
  - 使用 noatime 挂载
  - 选择合适的文件系统

□ 代码规范:
  - 文件流必须关闭
  - 临时文件及时删除
  - 使用缓冲写入
  - 避免频繁小文件 IO

□ 运维规范:
  - 定期清理日志
  - 监控磁盘健康
  - RAID 状态检查
  - 备份策略
```
