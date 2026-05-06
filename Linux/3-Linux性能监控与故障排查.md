# Linux 性能监控与故障排查

## 1. 性能监控工具

```
Linux 性能工具体系:

┌─────────────────────────────────────────────────────────────┐
│                    Linux 性能工具                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CPU 监控:                                                  │
│  ├── top/htop:      实时进程监控                           │
│  ├── mpstat:        多核 CPU 统计                          │
│  ├── pidstat:       进程级统计                             │
│  └── perf:          性能分析工具                           │
│                                                             │
│  内存监控:                                                  │
│  ├── free:          内存使用统计                           │
│  ├── vmstat:        虚拟内存统计                           │
│  ├── slabtop:       内核 slab 缓存                         │
│  └── /proc/meminfo: 内存详细信息                           │
│                                                             │
│  磁盘 IO:                                                   │
│  ├── iostat:        IO 统计                                │
│  ├── iotop:         IO 进程监控                            │
│  ├── df:            磁盘空间                               │
│  └── du:            目录大小                               │
│                                                             │
│  网络监控:                                                  │
│  ├── netstat:       网络连接                               │
│  ├── ss:            套接字统计                             │
│  ├── iftop:         网络流量                               │
│  └── tcpdump:       网络抓包                               │
│                                                             │
│  综合工具:                                                  │
│  ├── dstat:         多维度统计                             │
│  ├── sar:           系统活动报告                           │
│  ├── nmon:          系统监控                               │
│  └── glances:       系统监控                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. CPU 监控

### 2.1 top 命令

```bash
# top 常用操作
top
# 按 1: 显示每个 CPU 核心
# 按 P: 按 CPU 排序
# 按 M: 按内存排序
# 按 k: 杀死进程
# 按 q: 退出

# top 输出说明
# %Cpu(s):  CPU 使用率
#   us: 用户空间
#   sy: 内核空间
#   ni: 调整优先级
#   id: 空闲
#   wa: IO 等待
#   hi: 硬中断
#   si: 软中断
#   st: 虚拟机偷取时间
```

### 2.2 mpstat 多核统计

```bash
# 每秒输出一次，共5次
mpstat 1 5

# 指定 CPU 核心
mpstat -P 0 1 5

# 输出说明:
# %usr:   用户空间
# %sys:   内核空间
# %iowait: IO 等待
# %idle:  空闲
```

### 2.3 pidstat 进程统计

```bash
# 进程 CPU 使用
pidstat -u 1 5

# 进程内存使用
pidstat -r 1 5

# 进程 IO 使用
pidstat -d 1 5

# 指定进程
pidstat -p <pid> 1 5
```

## 3. 内存监控

### 3.1 free 命令

```bash
# 内存使用统计
free -h

# 输出说明:
#               total        used        free      shared  buff/cache   available
# Mem:           15Gi       8.0Gi       2.0Gi       500Mi       5.0Gi       6.0Gi
# Swap:         2.0Gi       0.0Gi       2.0Gi

# total:    总内存
# used:     已使用
# free:     空闲
# shared:   共享
# buff/cache: 缓冲/缓存
# available: 可用内存
```

### 3.2 vmstat 虚拟内存

```bash
# 每秒输出一次
vmstat 1

# 输出说明:
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  1  0      0 2048000 512000 5120000    0    0     0     0  100  200  10  5 85  0  0

# r: 运行队列长度
# b: 阻塞进程数
# swpd: 使用的交换分区
# free: 空闲内存
# buff: 缓冲区
# cache: 缓存
# si/so: 交换分区读写
# bi/bo: 块设备读写
# in: 中断数
# cs: 上下文切换
# us/sy/id/wa: CPU 使用率
```

## 4. 磁盘 IO 监控

### 4.1 iostat 命令

```bash
# IO 统计
iostat -x 1

# 输出说明:
# Device  rrqm/s wrqm/s  r/s   w/s  rMB/s  wMB/s avgrq-sz avgqu-sz await r_await w_await svctm %util
# sda       0.00   0.00  0.00  0.00   0.00   0.00     0.00     0.00  0.00    0.00    0.00   0.00   0.00

# rrqm/s:   每秒合并读请求数
# wrqm/s:   每秒合并写请求数
# r/s:      每秒读请求数
# w/s:      每秒写请求数
# rMB/s:    每秒读取量
# wMB/s:    每秒写入量
# await:    平均 IO 等待时间
# %util:    IO 使用率
```

### 4.2 iotop 进程 IO

```bash
# 进程级 IO 监控
iotop

# 输出说明:
# Total DISK READ:       0.00 B/s | Total DISK WRITE:       0.00 B/s
# Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       0.00 B/s
#   TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
#   1234 be/4  root     0.00 B/s    0.00 B/s  0.00 %  0.00 % java
```

## 5. 网络监控

### 5.1 netstat 命令

```bash
# 查看所有连接
netstat -an

# 查看监听端口
netstat -tuln

# 查看 TCP 连接状态
netstat -an | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key, state[key]}'

# 查看进程使用的端口
netstat -tulnp
```

### 5.2 ss 命令

```bash
# 查看监听端口
ss -tuln

# 查看所有 TCP 连接
ss -tan

# 查看连接数统计
ss -s

# 按状态过滤
ss -tan state established
ss -tan state time-wait
```

### 5.3 tcpdump 抓包

```bash
# 捕获指定接口的包
tcpdump -i eth0

# 捕获指定端口的包
tcpdump -i eth0 port 80

# 捕获指定主机的包
tcpdump -i eth0 host 192.168.1.100

# 保存到文件
tcpdump -i eth0 -w capture.pcap

# 读取文件
tcpdump -r capture.pcap
```

## 6. 故障排查流程

### 6.1 CPU 飙高排查

```bash
# 1. 查看 CPU 使用率
top -bn1 | head -5

# 2. 找到 CPU 高的进程
ps aux --sort=-%cpu | head -10

# 3. 查看进程的线程
top -Hp <pid>

# 4. 获取线程堆栈（Java 进程）
printf "%x\n" <tid>
jstack <pid> | grep <tid_hex> -A 30

# 5. 分析堆栈
# 常见原因:
# - 死循环
# - 频繁 GC
# - 锁竞争
# - 计算密集型任务
```

### 6.2 内存问题排查

```bash
# 1. 查看内存使用
free -h

# 2. 查看进程内存
ps aux --sort=-%mem | head -10

# 3. 查看内存详细信息
cat /proc/meminfo

# 4. 查看 slab 缓存
slabtop

# 5. 查看 OOM 事件
dmesg | grep -i oom
journalctl -k | grep -i oom
```

### 6.3 IO 问题排查

```bash
# 1. 查看 IO 使用率
iostat -x 1

# 2. 找到 IO 高的进程
iotop -o

# 3. 查看磁盘空间
df -h

# 4. 查看大文件
du -sh /* | sort -rh | head -10

# 5. 查看 IO 等待
vmstat 1
```

### 6.4 网络问题排查

```bash
# 1. 检查网络连通性
ping host

# 2. 检查端口监听
netstat -tuln | grep port

# 3. 检查防火墙
iptables -L

# 4. 检查 DNS
nslookup domain
dig domain

# 5. 抓包分析
tcpdump -i eth0 port 80 -nn
```

## 7. 性能优化建议

```
性能优化 Checklist:

□ CPU 优化:
  - 减少上下文切换
  - 优化锁竞争
  - 使用多核并行
  - 避免 CPU 密集型在主线程

□ 内存优化:
  - 合理配置堆大小
  - 及时释放不用的对象
  - 使用内存池
  - 监控内存泄漏

□ IO 优化:
  - 使用 SSD
  - 批量读写
  - 异步 IO
  - 减少小文件操作

□ 网络优化:
  - 使用连接池
  - 减少网络往返
  - 压缩传输数据
  - 使用 CDN

□ 监控告警:
  - 设置 CPU/内存/IO 阈值
  - 配置自动告警
  - 定期性能分析
  - 建立性能基线
```
