# Git 高级技巧与实战

## 1. Stash 暂存工作区

```bash
# 暂存当前工作区
git stash
git stash save "正在开发的功能"

# 查看暂存列表
git stash list

# 恢复暂存
git stash pop                 # 恢复并删除
git stash apply               # 恢复不删除
git stash apply stash@{1}     # 恢复指定暂存

# 删除暂存
git stash drop stash@{0}
git stash clear               # 清空所有

# 从暂存创建分支
git stash branch new-feature stash@{0}

# 暂存未跟踪文件
git stash -u
git stash --include-untracked

# 暂存指定文件
git stash push -m "message" -- file1.txt file2.txt
```

## 2. Reset 与 Revert

```bash
# Reset 三种模式
git reset --soft HEAD~1   # 保留工作区和暂存区
git reset --mixed HEAD~1  # 保留工作区，清空暂存区（默认）
git reset --hard HEAD~1   # 清空工作区和暂存区

# Revert 撤销提交（保留历史）
git revert HEAD           # 撤销最近一次提交
git revert abc1234        # 撤销指定提交
git revert HEAD~3..HEAD   # 撤销最近 3 次提交

# 对比:
# reset: 重写历史，适合本地未推送的提交
# revert: 创建新提交撤销，适合已推送的提交
```

## 3. Cherry-Pick 摘取提交

```bash
# 摘取单个提交
git cherry-pick abc1234

# 摘取多个提交
git cherry-pick abc1234 def5678

# 摘取提交范围
git cherry-pick abc1234..def5678

# 只应用变更，不自动提交
git cherry-pick --no-commit abc1234

# 处理冲突后继续
git cherry-pick --continue

# 放弃 cherry-pick
git cherry-pick --abort
```

## 4. 交互式 Rebase

```bash
# 交互式 rebase（修改最近 N 次提交）
git rebase -i HEAD~3

# 编辑器显示:
pick abc1234 feat: 添加用户登录
pick def5678 feat: 添加登录验证
pick ghi9012 fix: 修复登录bug

# 操作命令:
# pick:   保留提交
# reword: 修改提交信息
# edit:   修改提交内容
# squash: 合并到上一个提交
# fixup:  合并到上一个提交，丢弃提交信息
# drop:   删除提交

# 示例：合并多个提交为一个
pick abc1234 feat: 添加用户登录
squash def5678 feat: 添加登录验证
squash ghi9012 fix: 修复登录bug
# 结果：合并为一个提交 "feat: 添加用户登录"
```

## 5. Reflog 引用日志

```bash
# 查看引用日志
git reflog
git reflog --all

# 恢复误删的分支
git reflog
# 找到分支删除前的 commit hash
git checkout -b recovered-branch abc1234

# 恢复误 reset 的提交
git reflog
# 找到 reset 前的 commit hash
git reset --hard abc1234

# 引用日志过期时间
git gc --reflog-expire=90.days.ago
```

## 6. Bisect 二分查找

```bash
# 开始二分查找
git bisect start

# 标记当前版本为坏
git bisect bad

# 标记某个版本为好
git bisect good v1.0.0

# Git 自动检出中间版本，测试后标记
git bisect good  # 或 git bisect bad

# 自动查找（需要测试脚本）
git bisect run ./test-script.sh

# 结束二分查找
git bisect reset
```

## 7. Submodule 子模块

```bash
# 添加子模块
git submodule add https://github.com/user/lib.git libs/lib

# 克隆包含子模块的仓库
git clone --recursive https://github.com/user/project.git
# 或
git clone https://github.com/user/project.git
git submodule init
git submodule update

# 更新子模块
git submodule update --remote

# 删除子模块
git submodule deinit libs/lib
git rm libs/lib
rm -rf .git/modules/libs/lib
```

## 8. 大文件处理（Git LFS）

```bash
# 安装 Git LFS
git lfs install

# 跟踪大文件类型
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.jar"

# 查看跟踪规则
git lfs track

# 迁移已有大文件到 LFS
git lfs migrate import --include="*.psd"

# 查看 LFS 文件
git lfs ls-files

# 拉取 LFS 文件
git lfs pull
```

## 9. 提交规范（Conventional Commits）

```
Conventional Commits 格式:

<type>(<scope>): <subject>

<body>

<footer>

type 类型:
├── feat:     新功能
├── fix:      Bug 修复
├── docs:     文档变更
├── style:    代码格式（不影响功能）
├── refactor: 重构（非新功能、非修复）
├── perf:     性能优化
├── test:     测试相关
├── build:    构建系统或外部依赖
├── ci:       CI 配置
├── chore:    其他变更
└── revert:   撤销提交

示例:
feat(auth): 添加 OAuth2 登录支持

- 集成 GitHub OAuth
- 添加 OAuth2 配置类
- 实现回调处理

Closes #123

BREAKING CHANGE: 移除旧的登录接口
```

## 10. 常用别名配置

```bash
# 配置常用别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.whoami 'config user.name'
git config --global alias.today "log --since=midnight --oneline --no-merges"
git config --global alias.amend 'commit --amend --no-edit'
```

## 11. 清理与维护

```bash
# 查看未跟踪文件
git clean -n  # 预览
git clean -f  # 删除
git clean -fd # 删除文件和目录

# 清理已合并分支
git branch --merged main | grep -v main | xargs git branch -d

# 垃圾回收
git gc
git gc --aggressive

# 验证仓库完整性
git fsck

# 优化仓库
git repack -a -d --depth=250 --window=250
```
