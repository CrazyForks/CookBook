# Shell 脚本编程实战

## 1. Shell 脚本基础

### 1.1 脚本结构

```bash
#!/bin/bash
# 脚本描述: 示例脚本
# 作者: Author
# 日期: 2024-01-01

set -e  # 遇到错误立即退出
set -u  # 使用未定义变量报错
set -o pipefail  # 管道命令失败时报错

# 主逻辑
main() {
    echo "Hello, World!"
}

main "$@"
```

### 1.2 变量

```bash
# 变量定义（等号两边不能有空格）
name="John"
age=25
readonly PI=3.14  # 只读变量

# 变量使用
echo "Name: $name"
echo "Name: ${name}"  # 推荐使用花括号

# 字符串操作
str="Hello World"
echo ${#str}          # 长度: 11
echo ${str:0:5}       # 截取: Hello
echo ${str/World/Lua} # 替换: Hello Lua

# 特殊变量
$0    # 脚本名称
$1    # 第一个参数
$#    # 参数个数
$@    # 所有参数（数组）
$*    # 所有参数（字符串）
$?    # 上一个命令返回值
$$    # 当前进程 ID
```

### 1.3 数据类型

```bash
# 字符串
str1="Hello"
str2='World'
str3="${str1} ${str2}"  # 字符串拼接

# 数组
arr=(1 2 3 4 5)
echo ${arr[0]}       # 访问元素: 1
echo ${arr[@]}       # 所有元素
echo ${#arr[@]}      # 数组长度

# 关联数组（Bash 4+）
declare -A map
map[name]="John"
map[age]=25
echo ${map[name]}
```

## 2. 流程控制

### 2.1 条件判断

```bash
# if 语句
if [ condition ]; then
    # code
elif [ condition2 ]; then
    # code
else
    # code
fi

# 条件表达式
# 文件测试
if [ -f file.txt ]; then echo "文件存在"; fi
if [ -d /path ]; then echo "目录存在"; fi
if [ -r file.txt ]; then echo "可读"; fi
if [ -w file.txt ]; then echo "可写"; fi
if [ -x file.txt ]; then echo "可执行"; fi
if [ -s file.txt ]; then echo "非空"; fi

# 字符串比较
if [ "$str" = "Hello" ]; then echo "相等"; fi
if [ "$str" != "World" ]; then echo "不相等"; fi
if [ -z "$str" ]; then echo "为空"; fi
if [ -n "$str" ]; then echo "非空"; fi

# 数值比较
if [ $a -eq $b ]; then echo "相等"; fi
if [ $a -ne $b ]; then echo "不相等"; fi
if [ $a -gt $b ]; then echo "大于"; fi
if [ $a -lt $b ]; then echo "小于"; fi
if [ $a -ge $b ]; then echo "大于等于"; fi
if [ $a -le $b ]; then echo "小于等于"; fi

# 逻辑运算
if [ $a -gt 0 ] && [ $a -lt 100 ]; then echo "0-100"; fi
if [ $a -eq 0 ] || [ $a -eq 1 ]; then echo "0或1"; fi
if [ ! -z "$str" ]; then echo "非空"; fi

# case 语句
case $1 in
    start)
        echo "Starting..."
        ;;
    stop)
        echo "Stopping..."
        ;;
    restart)
        echo "Restarting..."
        ;;
    *)
        echo "Usage: $0 {start|stop|restart}"
        exit 1
        ;;
esac
```

### 2.2 循环

```bash
# for 循环
for i in 1 2 3 4 5; do
    echo $i
done

for i in {1..5}; do
    echo $i
done

for ((i=0; i<5; i++)); do
    echo $i
done

for file in *.txt; do
    echo "Processing: $file"
done

# while 循环
count=0
while [ $count -lt 5 ]; do
    echo $count
    count=$((count + 1))
done

# 读取文件
while IFS= read -r line; do
    echo "$line"
done < file.txt

# until 循环
count=0
until [ $count -ge 5 ]; do
    echo $count
    count=$((count + 1))
done

# break 和 continue
for i in {1..10}; do
    if [ $i -eq 5 ]; then
        continue  # 跳过5
    fi
    if [ $i -eq 8 ]; then
        break     # 到8停止
    fi
    echo $i
done
```

## 3. 函数

```bash
# 函数定义
function greet() {
    local name=$1  # local 定义局部变量
    echo "Hello, $name!"
}

# 调用函数
greet "World"

# 带返回值的函数
add() {
    local result=$(( $1 + $2 ))
    echo $result  # 通过 echo 返回
}

result=$(add 1 2)
echo "Sum: $result"

# 检查返回值
check_file() {
    if [ -f "$1" ]; then
        return 0  # 成功
    else
        return 1  # 失败
    fi
}

if check_file "test.txt"; then
    echo "File exists"
else
    echo "File not found"
fi
```

## 4. 输入输出

```bash
# 读取输入
echo -n "Enter name: "
read name
echo "Hello, $name!"

# 带提示的读取
read -p "Enter age: " age
read -s -p "Enter password: " password  # 静默输入

# 格式化输出
printf "Name: %-10s Age: %d\n" "John" 25
printf "%05d\n" 42  # 00042

# 重定向
echo "Hello" > file.txt      # 覆盖写入
echo "World" >> file.txt     # 追加写入
command 2> error.log         # 错误重定向
command > out.log 2>&1       # 合并重定向
command &> all.log           # 合并重定向（简写）
command < input.txt          # 输入重定向

# Here Document
cat << EOF
Hello
World
EOF

# Here String
grep "pattern" <<< "Hello World"
```

## 5. 错误处理

```bash
# 错误处理最佳实践
set -e  # 遇到错误立即退出
set -u  # 未定义变量报错
set -o pipefail  # 管道失败时报错

# trap 捕获信号
trap 'echo "Ctrl+C pressed"; exit 1' INT
trap 'echo "Cleaning up..."; rm -f /tmp/lockfile' EXIT

# 错误处理函数
error_handler() {
    echo "Error occurred at line $1"
    exit 1
}

trap 'error_handler $LINENO' ERR

# 安全的命令执行
safe_command() {
    if ! command; then
        echo "Command failed"
        return 1
    fi
}
```

## 6. 实战脚本

### 6.1 日志清理脚本

```bash
#!/bin/bash
# 清理超过7天的日志文件

set -euo pipefail

LOG_DIR="/var/log/app"
DAYS=7

find "$LOG_DIR" -name "*.log" -type f -mtime +$DAYS -exec rm -f {} \;
echo "Cleaned logs older than $DAYS days from $LOG_DIR"
```

### 6.2 备份脚本

```bash
#!/bin/bash
# 数据库备份脚本

set -euo pipefail

BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="mydb"
RETENTION_DAYS=30

# 创建备份
mysqldump -u root "$DB_NAME" | gzip > "$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz"

# 清理旧备份
find "$BACKUP_DIR" -name "*.sql.gz" -type f -mtime +$RETENTION_DAYS -delete

echo "Backup completed: ${DB_NAME}_${DATE}.sql.gz"
```

### 6.3 服务监控脚本

```bash
#!/bin/bash
# 服务健康检查脚本

set -euo pipefail

SERVICE_NAME="myapp"
CHECK_URL="http://localhost:8080/health"
MAX_RETRIES=3
RETRY_INTERVAL=5

check_health() {
    local status_code
    status_code=$(curl -s -o /dev/null -w "%{http_code}" "$CHECK_URL")
    
    if [ "$status_code" -eq 200 ]; then
        return 0
    else
        return 1
    fi
}

restart_service() {
    systemctl restart "$SERVICE_NAME"
    sleep 10
}

for ((i=1; i<=MAX_RETRIES; i++)); do
    if check_health; then
        echo "Health check passed"
        exit 0
    fi
    
    echo "Health check failed (attempt $i/$MAX_RETRIES)"
    
    if [ $i -lt $MAX_RETRIES ]; then
        sleep $RETRY_INTERVAL
    fi
done

echo "Service unhealthy, restarting..."
restart_service

if check_health; then
    echo "Service restarted successfully"
else
    echo "Service restart failed"
    exit 1
fi
```

## 7. 调试技巧

```bash
# 启用调试
bash -x script.sh          # 执行并打印命令
set -x                     # 在脚本中启用调试
set +x                     # 禁用调试

# shellcheck 检查
shellcheck script.sh       # 静态分析

# 常见错误
# 1. 变量未加引号
rm -rf $DIR/*              # 危险！
rm -rf "$DIR"/*            # 正确

# 2. 等号两边有空格
var = "value"              # 错误！
var="value"                # 正确

# 3. 未处理空变量
echo $undefined_var        # 可能出错
echo "${undefined_var:-}"  # 安全
```
