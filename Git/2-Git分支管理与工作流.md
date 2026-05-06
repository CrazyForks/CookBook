# Git 分支管理与工作流

## 1. 分支概述

```
Git 分支模型:

┌─────────────────────────────────────────────────────────────┐
│                    Git 分支本质                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  分支是指向提交的可变指针                                   │
│                                                             │
│  main ──→ C1 ──→ C2 ──→ C3                                 │
│                         ↑                                   │
│                         │                                   │
│  feature ──────────────┘                                    │
│                                                             │
│  创建分支 = 创建一个 41 字节的指针文件                      │
│  切换分支 = 移动 HEAD 指针 + 更新工作区                     │
│                                                             │
│  分支类型:                                                  │
│  ├── main/master: 主分支，生产代码                         │
│  ├── develop:     开发分支，集成分支                       │
│  ├── feature/*:   功能分支                                 │
│  ├── release/*:   发布分支                                 │
│  ├── hotfix/*:    热修复分支                               │
│  └── support/*:   支持分支                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 分支基础操作

```bash
# 查看分支
git branch                    # 本地分支
git branch -r                 # 远程分支
git branch -a                 # 所有分支
git branch -v                 # 分支及最后提交

# 创建分支
git branch feature/login      # 创建分支
git checkout -b feature/login # 创建并切换
git switch -c feature/login   # Git 2.23+ 推荐

# 切换分支
git checkout main
git switch main               # Git 2.23+ 推荐

# 重命名分支
git branch -m old-name new-name
git branch -m new-name        # 重命名当前分支

# 删除分支
git branch -d feature/login   # 删除已合并分支
git branch -D feature/login   # 强制删除

# 远程分支操作
git push origin feature/login # 推送分支到远程
git push origin --delete feature/login  # 删除远程分支

# 设置上游分支
git branch --set-upstream-to=origin/main main
git branch -u origin/main     # 简写
```

## 3. 分支合并策略

### 3.1 Fast-Forward 合并

```
Fast-Forward 合并（默认）:

合并前:
main:    C1 ──→ C2
                    ↑
feature:            C3 ──→ C4

执行: git checkout main && git merge feature

合并后（fast-forward）:
main:    C1 ──→ C2 ──→ C3 ──→ C4
                              ↑
feature: ────────────────────┘

特点:
- 不创建新的合并提交
- 保持线性历史
- 适合简单的分支合并
```

```bash
# Fast-Forward 合并
git checkout main
git merge feature/login

# 禁止 Fast-Forward（总是创建合并提交）
git merge --no-ff feature/login
```

### 3.2 三方合并

```
三方合并（Three-Way Merge）:

合并前:
main:    C1 ──→ C2 ──→ C5
                    ↑
feature:            C3 ──→ C4

执行: git checkout main && git merge feature

合并后（创建合并提交）:
main:    C1 ──→ C2 ──→ C5 ──→ M
                    ↑         ↑
feature:            C3 ──→ C4 ┘

特点:
- 创建新的合并提交 M
- 保留分支历史
- 适合团队协作
```

### 3.3 Rebase 变基

```
Rebase 变基:

合并前:
main:    C1 ──→ C2 ──→ C5
                    ↑
feature:            C3 ──→ C4

执行: git checkout feature && git rebase main

变基后:
main:    C1 ──→ C2 ──→ C5
                          ↑
feature:                  C3' ──→ C4'

特点:
- 重写提交历史
- 保持线性历史
- 个人分支使用，公共分支避免
```

```bash
# Rebase 操作
git checkout feature/login
git rebase main

# 交互式 Rebase（修改/合并/删除提交）
git rebase -i HEAD~3

# Rebase 冲突解决
git rebase --continue
git rebase --abort
git rebase --skip
```

## 4. Git Flow 工作流

```
Git Flow 分支模型:

┌─────────────────────────────────────────────────────────────┐
│                      Git Flow                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  main ─────────────────●─────────────────────●─────────────
│                        │                     ↑             │
│                        │ release/1.0         │             │
│                        ↓                     │             │
│  develop ──●─────●─────●─────●─────●─────────●─────────────
│            ↑     ↑           ↑     ↑                       │
│            │     │           │     │                       │
│  feature/  │     │ feature/  │     │ feature/              │
│  login ────┘     │ payment ──┘     │ order ──              │
│                  │                 │                       │
│  hotfix/1.0.1 ───┼─────────────────┘                       │
│                  ↓                                         │
│  main ───────────●─────────────────────────────────────────│
│                                                             │
│  分支说明:                                                  │
│  main:     生产分支，只接受 release 和 hotfix 合入         │
│  develop:  开发主分支，功能分支的集成点                     │
│  feature/*: 功能分支，从 develop 创建，完成后合回 develop   │
│  release/*: 发布分支，从 develop 创建，完成后合入 main      │
│  hotfix/*:  热修复分支，从 main 创建，修复后合入 main       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```bash
# Git Flow 命令

# 安装 git-flow
brew install git-flow  # macOS
apt-get install git-flow  # Ubuntu

# 初始化
git flow init

# 功能分支
git flow feature start login
git flow feature finish login
git flow feature publish login

# 发布分支
git flow release start 1.0.0
git flow release finish 1.0.0

# 热修复分支
git flow hotfix start 1.0.1
git flow hotfix finish 1.0.1
```

## 5. GitHub Flow

```
GitHub Flow（简化工作流）:

┌─────────────────────────────────────────────────────────────┐
│                    GitHub Flow                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 从 main 创建功能分支                                    │
│     git checkout -b feature/login main                      │
│                                                             │
│  2. 开发并提交代码                                          │
│     git add . && git commit -m "feat: login"                │
│                                                             │
│  3. 推送分支并创建 Pull Request                             │
│     git push origin feature/login                           │
│                                                             │
│  4. 代码审查和讨论                                          │
│     - 同事审查代码                                          │
│     - 自动化测试运行                                        │
│                                                             │
│  5. 合并到 main                                             │
│     - Squash and merge（推荐）                              │
│     - Rebase and merge                                      │
│     - Create a merge commit                                 │
│                                                             │
│  6. 部署 main 分支                                          │
│                                                             │
│  特点:                                                      │
│  - 只有一个长期分支 main                                    │
│  - 所有开发在功能分支进行                                   │
│  - 通过 PR 合并代码                                         │
│  - 适合持续部署                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 6. Trunk-Based Development

```
Trunk-Based Development（主干开发）:

┌─────────────────────────────────────────────────────────────┐
│                 Trunk-Based Development                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  main ──●──●──●──●──●──●──●──●──●──●──●──●──               │
│         ↑  ↑  ↑  ↑  ↑  ↑                                   │
│         │  │  │  │  │  │                                    │
│         └──┴──┴──┴──┴──┘                                    │
│            短命分支（< 2天）                                │
│                                                             │
│  核心原则:                                                  │
│  - 所有开发在主干进行                                       │
│  - 分支生命周期极短（1-2天）                                │
│  - 频繁集成，减少合并冲突                                   │
│  - 依赖特性开关（Feature Toggle）                           │
│  - 依赖自动化测试保证质量                                   │
│                                                             │
│  适用场景:                                                  │
│  - 高频发布（每天多次）                                     │
│  - 成熟的 CI/CD 流水线                                      │
│  - 完善的自动化测试                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 7. 分支命名规范

```
分支命名规范:

格式: <type>/<ticket-id>-<description>

类型:
├── feature/   新功能
├── bugfix/    Bug 修复
├── hotfix/    紧急修复
├── release/   发布准备
├── support/   支持维护
└── docs/      文档更新

示例:
feature/PROJ-123-user-login
bugfix/PROJ-456-fix-null-pointer
hotfix/PROJ-789-security-patch
release/1.2.0
docs/update-readme

命名规则:
- 使用小写字母和连字符
- 包含工单号（JIRA/Trello）
- 简洁描述分支目的
- 避免使用特殊字符
```

## 8. 最佳实践

```
分支管理 Checklist:

□ 选择合适的工作流:
  - 小团队: GitHub Flow
  - 大团队: Git Flow
  - 高频发布: Trunk-Based

□ 分支命名规范:
  - 统一命名格式
  - 包含工单号
  - 简洁描述目的

□ 分支生命周期:
  - 及时删除已合并分支
  - 避免长期存在的功能分支
  - 定期清理远程分支

□ 合并策略:
  - 功能分支: Squash and merge
  - 发布分支: Merge commit
  - 热修复: Cherry-pick

□ 代码审查:
  - 所有代码通过 PR 合并
  - 至少一人审查
  - 自动化测试通过

□ 提交规范:
  - 使用 Conventional Commits
  - 每次提交完成一个逻辑单元
  - 提交信息清晰描述变更
```
