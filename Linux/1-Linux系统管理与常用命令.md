# Linux 系统管理与常用命令

## 1. 文件系统

### 1.1 目录结构

```
Linux 目录结构:

/
├── bin/        基础命令（ls, cp, mv）
├── boot/       启动文件（内核、引导程序）
├── dev/        设备文件
├── etc/        系统配置文件
├── home/       用户主目录
├── lib/        共享库
├── media/      可移动媒体挂载点
├── mnt/        临时挂载点
├── opt/        可选软件包
├── proc/       进程信息（虚拟文件系统）
├── root/       root 用户主目录
├── sbin/       系统管理命令
├── srv/        服务数据
├── sys/        系统信息（虚拟文件系统）
├── tmp/        临时文件
├── usr/        用户程序
│   ├── bin/    用户命令
│   ├── lib/    共享库
│   ├── local/  本地安装软件
│   └── sbin/   系统管理命令
└── var/        可变数据
    ├── log/    日志文件
    ├── tmp/    临时文件
    └── spool/  队列数据
```

### 1.2 文件操作命令

```bash
# 文件查看
ls -la                    # 列出文件（详细、隐藏）
cat file.txt              # 查看文件内容
less file.txt             # 分页查看
head -n 10 file.txt       # 查看前10行
tail -n 10 file.txt       # 查看后10行
tail -f file.txt          # 实时查看

# 文件操作
cp source dest            # 复制文件
mv source dest            # 移动/重命名
rm file.txt               # 删除文件
rm -rf directory          # 强制删除目录
mkdir -p /path/to/dir     # 创建目录
rmdir directory           # 删除空目录
ln -s target link_name    # 创建软链接

# 文件搜索
find / -name "*.log"      # 按名称查找
find / -type f -size +100M  # 查找大文件
find / -mtime -7          # 查找7天内修改的文件
locate filename           # 快速查找（需要updatedb）
which command             # 查找命令位置
whereis command           # 查找命令相关文件

# 文件权限
chmod 755 file.txt        # 设置权限
chmod u+x file.txt        # 添加执行权限
chown user:group file     # 修改所有者
chgrp group file          # 修改所属组

# 权限说明:
# r(4) 读  w(2) 写  x(1) 执行
# 755: rwxr-xr-x (所有者rwx，组r-x，其他r-x)
# 644: rw-r--r-- (所有者rw-，组r--，其他r--)
```

## 2. 用户管理

```bash
# 用户操作
useradd -m -s /bin/bash username  # 创建用户
passwd username           # 设置密码
userdel -r username       # 删除用户
usermod -aG group user    # 添加用户到组
id username               # 查看用户信息
whoami                    # 当前用户
who                       # 登录用户
w                         # 登录用户详情

# 组操作
groupadd groupname        # 创建组
groupdel groupname        # 删除组
groups username           # 查看用户所属组

# sudo 配置
visudo                    # 编辑 sudoers 文件
# username ALL=(ALL:ALL) ALL
# %groupname ALL=(ALL:ALL) ALL
```

## 3. 进程管理

```bash
# 进程查看
ps aux                    # 所有进程
ps -ef                    # 进程树
ps -p <pid>               # 指定进程
top                       # 实时进程
htop                      # 增强版 top

# 进程操作
kill <pid>                # 发送 SIGTERM
kill -9 <pid>             # 强制终止
killall name              # 按名称杀进程
pkill pattern             # 按模式杀进程

# 后台运行
command &                 # 后台运行
nohup command &           # 后台运行，不挂断
jobs                      # 查看后台任务
fg %n                     # 切换到前台
bg %n                     # 继续后台运行

# 系统服务
systemctl start service   # 启动服务
systemctl stop service    # 停止服务
systemctl restart service # 重启服务
systemctl status service  # 查看状态
systemctl enable service  # 开机启动
systemctl disable service # 禁止开机启动
systemctl list-units      # 列出所有服务
```

## 4. 网络管理

```bash
# 网络配置
ip addr                   # 查看 IP 地址
ip route                  # 查看路由表
ip link                   # 查看网络接口
ifconfig                  # 网络接口配置（旧）

# 网络测试
ping host                 # 测试连通性
traceroute host           # 路由追踪
nslookup domain           # DNS 查询
dig domain                # DNS 查询（详细）
host domain               # DNS 查询

# 端口和连接
netstat -tuln             # 查看监听端口
netstat -an               # 所有连接
ss -tuln                  # 查看监听端口（新版）
lsof -i :80               # 查看端口占用
lsof -p <pid>             # 查看进程打开的文件

# 防火墙
iptables -L               # 查看规则
iptables -A INPUT -p tcp --dport 80 -j ACCEPT  # 允许80端口
firewall-cmd --list-all   # firewalld 规则
ufw status                # Ubuntu 防火墙

# 网络工具
curl url                  # HTTP 请求
wget url                  # 下载文件
scp file user@host:path   # 远程复制
ssh user@host             # 远程登录
```

## 5. 磁盘管理

```bash
# 磁盘查看
df -h                     # 磁盘空间
du -sh *                  # 目录大小
du -sh /path              # 指定目录大小
lsblk                     # 块设备列表
fdisk -l                  # 分区信息

# 磁盘操作
mount /dev/sdb1 /mnt      # 挂载
umount /mnt               # 卸载
mkfs.ext4 /dev/sdb1       # 格式化

# LVM 管理
pvdisplay                 # 物理卷
vgdisplay                 # 卷组
lvdisplay                 # 逻辑卷

# RAID 管理
mdadm --detail /dev/md0   # RAID 状态
cat /proc/mdstat          # RAID 状态
```

## 6. 包管理

```bash
# Debian/Ubuntu (apt)
apt update                # 更新源
apt upgrade               # 升级包
apt install package       # 安装包
apt remove package        # 删除包
apt search keyword        # 搜索包
apt list --installed      # 已安装包

# CentOS/RHEL (yum/dnf)
yum update                # 更新
yum install package       # 安装
yum remove package        # 删除
yum search keyword        # 搜索
yum list installed        # 已安装包
dnf install package       # Fedora 新版

# 源码安装
./configure               # 配置
make                      # 编译
make install              # 安装
```

## 7. 系统信息

```bash
# 系统信息
uname -a                  # 内核信息
cat /etc/os-release       # 发行版信息
hostname                  # 主机名
uptime                    # 运行时间和负载
date                      # 系统时间
timedatectl               # 时间设置

# 硬件信息
lscpu                     # CPU 信息
free -h                   # 内存信息
lsblk                     # 块设备
lspci                     # PCI 设备
lsusb                     # USB 设备
dmidecode                 # 硬件详情

# 性能监控
vmstat 1                  # 虚拟内存统计
iostat 1                  # IO 统计
sar -u 1                  # CPU 使用率
```

## 8. 常用工具

```bash
# 文本处理
grep pattern file         # 搜索文本
grep -r pattern dir       # 递归搜索
sed 's/old/new/g' file    # 替换文本
awk '{print $1}' file     # 文本处理
cut -d: -f1 file          # 提取列
sort file                 # 排序
uniq file                 # 去重
wc -l file                # 统计行数

# 压缩解压
tar -czf archive.tar.gz dir   # 创建压缩包
tar -xzf archive.tar.gz       # 解压
tar -xjf archive.tar.bz2      # 解压 bz2
zip -r archive.zip dir         # 创建 zip
unzip archive.zip              # 解压 zip

# 系统工具
crontab -e                # 编辑定时任务
crontab -l                # 查看定时任务
at time                   # 一次性任务
watch -n 1 command        # 定时执行
screen -S name            # 创建会话
tmux                      # 终端复用
```
