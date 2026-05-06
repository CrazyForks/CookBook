# Git 基础入门与核心概念

## 1. Git 概述

```
Git 是什么？

┌─────────────────────────────────────────────────────────────┐
│                      Git 版本控制系统                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  定义: 分布式版本控制系统（DVCS）                           │
│                                                             │
│  核心特点:                                                  │
│  ├── 分布式: 每个开发者都有完整仓库                        │
│  ├── 快照式: 保存文件快照而非差异                          │
│  ├── 分支廉价: 创建/切换分支几乎瞬间完成                   │
│  ├── 完整性: 使用 SHA-1 校验所有数据                       │
│  └── 离线工作: 大部分操作无需网络                          │
│                                                             │
│  vs 集中式版本控制（SVN/CVS）:                             │
│  ┌─────────────┬─────────────┬─────────────┐               │
│  │ 特性        │ Git         │ SVN         │               │
│  ├─────────────┼─────────────┼─────────────┤               │
│  │ 架构        │ 分布式      │ 集中式      │               │
│  │ 离线工作    │ ✓           │ ✗           │               │
│  │ 分支成本    │ 极低        │ 高          │               │
│  │ 速度        │ 快          │ 慢          │               │
│  │ 完整性      │ SHA-1 校验  │ 无          │               │
│  └─────────────┴─────────────┴─────────────┘               │
│                                                             │
│  核心概念:                                                  │
│  ├── 仓库（Repository）: 项目的所有版本数据                │
│  ├── 工作区（Working Directory）: 项目目录                 │
│  ├── 暂存区（Staging Area）: Git 目录下的 index 文件       │
│  ├── 提交（Commit）: 项目的一个版本快照                    │
│  └── 分支（Branch）: 开发的独立线路                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 安装与配置

### 2.1 安装 Git

```bash
# macOS
brew install git

# Ubuntu/Debian
sudo apt-get update
sudo apt-get install git

# CentOS/RHEL
sudo yum install git

# Windows
# 下载安装包: https://git-scm.com/download/win

# 验证安装
git --version
# 输出: git version 2.43.0
```

### 2.2 全局配置

```bash
# 用户信息配置（必须）
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# 编辑器配置
git config --global core.editor vim
git config --global core.editor "code --wait"  # VS Code

# 默认分支名
git config --global init.defaultBranch main

# 自动处理换行符
git config --global core.autocrlf input  # macOS/Linux
git config --global core.autocrlf true   # Windows

# 颜色输出
git config --global color.ui auto

# 设置别名（提升效率）
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"

# 查看所有配置
git config --list

# 查看特定配置
git config user.name
```

### 2.3 SSH 配置

```bash
# 1. 生成 SSH 密钥
ssh-keygen -t ed25519 -C "your.email@example.com"
# 或使用 RSA
ssh-keygen -t rsa -b 4096 -C "your.email@example.com"

# 2. 启动 ssh-agent
eval "$(ssh-agent -s)"

# 3. 添加密钥到 ssh-agent
ssh-add ~/.ssh/id_ed25519

# 4. 复制公钥
# macOS
pbcopy < ~/.ssh/id_ed25519.pub
# Linux
cat ~/.ssh/id_ed25519.pub | xclip -selection clipboard
# Windows (Git Bash)
clip < ~/.ssh/id_ed25519.pub

# 5. 添加到 GitHub/GitLab
# GitHub: Settings → SSH and GPG keys → New SSH key

# 6. 测试连接
ssh -T git@github.com
# 成功: Hi username! You've successfully authenticated...
```

## 3. 基础操作

### 3.1 仓库初始化

```bash
# 方式一：本地初始化
mkdir my-project
cd my-project
git init

# 方式二：克隆远程仓库
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git  # SSH 方式

# 克隆指定分支
git clone -b develop https://github.com/user/repo.git

# 浅克隆（只克隆最近 N 次提交）
git clone --depth 1 https://github.com/user/repo.git
```

### 3.2 文件状态流转

```
Git 文件状态:

                    git add           git commit
  工作区 (Untracked) ─────→ 暂存区 (Staged) ─────→ 仓库 (Committed)
       ↑                      │                        │
       │                      │                        │
       │    git checkout --   │    git reset HEAD      │
       └──────────────────────┘←───────────────────────┘

文件状态:
- Untracked:  新文件，未被 Git 跟踪
- Modified:   已跟踪文件被修改
- Staged:     修改已放入暂存区
- Committed:  已提交到本地仓库
```

### 3.3 基础命令

```bash
# 查看状态
git status
git status -s  # 简洁格式

# 添加到暂存区
git add file.txt          # 添加单个文件
git add .                 # 添加所有修改
git add *.java            # 添加所有 Java 文件
git add src/              # 添加 src 目录

# 提交
git commit -m "feat: 添加用户登录功能"
git commit -am "fix: 修复登录bug"  # 跳过 add，直接提交已跟踪文件

# 查看提交历史
git log
git log --oneline         # 单行显示
git log --graph           # 图形化显示
git log --author="张三"   # 按作者筛选
git log -5                # 最近 5 次提交
git log --since="2024-01-01" --until="2024-12-31"

# 查看差异
git diff                  # 工作区 vs 暂存区
git diff --staged         # 暂存区 vs 最新提交
git diff HEAD             # 工作区 vs 最新提交
git diff branch1..branch2 # 两个分支差异

# 撤销操作
git checkout -- file.txt  # 撤销工作区修改
git reset HEAD file.txt   # 撤销暂存区修改
git commit --amend        # 修改最近一次提交
```

## 4. 远程仓库操作

```bash
# 查看远程仓库
git remote -v

# 添加远程仓库
git remote add origin https://github.com/user/repo.git

# 推送到远程
git push origin main
git push -u origin main   # -u 设置上游分支，后续可直接 git push

# 拉取远程更新
git pull origin main      # 拉取并合并
git fetch origin          # 只拉取不合并

# 删除远程分支
git push origin --delete feature/old-branch

# 推送标签
git push origin v1.0.0
git push origin --tags    # 推送所有标签
```

## 5. .gitignore 配置

```gitignore
# .gitignore 示例

# 编译输出
/target/
*.class
*.jar
*.war

# IDE 文件
.idea/
*.iml
.vscode/
*.swp
*.swo

# 日志文件
*.log
logs/

# 操作系统文件
.DS_Store
Thumbs.db

# 依赖目录
/node_modules/
/vendor/

# 环境配置
.env
.env.local
*.env.production

# Maven/Gradle
.mvn/
gradle/

# 临时文件
*.tmp
*.bak
*.cache

# 敏感文件
*.pem
*.key
credentials.json
```

## 6. 标签管理

```bash
# 查看标签
git tag
git tag -l "v1.*"

# 创建标签
git tag v1.0.0                          # 轻量标签
git tag -a v1.0.0 -m "Release 1.0.0"   # 附注标签

# 给历史提交打标签
git tag -a v1.0.0 abc1234

# 查看标签信息
git show v1.0.0

# 删除标签
git tag -d v1.0.0

# 推送标签到远程
git push origin v1.0.0
git push origin --tags
```

## 7. 常用命令速查表

```
Git 命令速查表:

┌──────────────────────┬───────────────────────────────────────┐
│ 命令                 │ 说明                                  │
├──────────────────────┼───────────────────────────────────────┤
│ git init             │ 初始化本地仓库                        │
│ git clone <url>      │ 克隆远程仓库                          │
│ git add <file>       │ 添加到暂存区                          │
│ git commit -m <msg>  │ 提交到本地仓库                        │
│ git status           │ 查看状态                              │
│ git log              │ 查看提交历史                          │
│ git diff             │ 查看差异                              │
│ git push             │ 推送到远程                            │
│ git pull             │ 拉取远程更新                          │
│ git fetch            │ 获取远程更新（不合并）                │
│ git branch           │ 查看/创建分支                         │
│ git checkout         │ 切换分支/恢复文件                     │
│ git merge            │ 合并分支                              │
│ git stash            │ 暂存工作区                            │
│ git reset            │ 重置提交/暂存区                       │
│ git revert           │ 撤销提交（保留历史）                  │
│ git tag              │ 标签管理                              │
│ git remote           │ 远程仓库管理                          │
│ git rebase           │ 变基操作                              │
│ git cherry-pick      │ 摘取提交                              │
└──────────────────────┴───────────────────────────────────────┘
```
