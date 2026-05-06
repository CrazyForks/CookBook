# Git 多人协作与代码审查

## 1. Pull Request / Merge Request

```
PR/MR 工作流程:

┌─────────────────────────────────────────────────────────────┐
│                   Pull Request 流程                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 创建功能分支                                            │
│     git checkout -b feature/login main                      │
│                                                             │
│  2. 开发并提交                                              │
│     git add . && git commit -m "feat: login"                │
│                                                             │
│  3. 推送分支                                                │
│     git push origin feature/login                           │
│                                                             │
│  4. 创建 Pull Request                                       │
│     - 标题: 简洁描述变更                                    │
│     - 描述: 详细说明变更内容                                │
│     - 关联 Issue: Closes #123                               │
│     - 指定 Reviewer                                         │
│                                                             │
│  5. 代码审查                                                │
│     - Reviewer 审查代码                                     │
│     - 自动化测试运行                                        │
│     - 讨论和修改                                            │
│                                                             │
│  6. 合并代码                                                │
│     - Squash and merge（推荐）                              │
│     - Rebase and merge                                      │
│     - Create a merge commit                                 │
│                                                             │
│  7. 清理分支                                                │
│     git checkout main && git pull                           │
│     git branch -d feature/login                             │
│     git push origin --delete feature/login                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.1 PR 描述模板

```markdown
## 变更描述

简要描述本次变更的内容和目的。

## 变更类型

- [ ] 新功能 (feature)
- [ ] Bug 修复 (fix)
- [ ] 重构 (refactor)
- [ ] 文档更新 (docs)
- [ ] 其他 (请说明)

## 关联 Issue

Closes #123

## 测试说明

- [ ] 单元测试通过
- [ ] 集成测试通过
- [ ] 手动测试通过

## 截图（如有 UI 变更）

## 审查要点

请重点关注以下方面：
1. xxx
2. xxx
```

## 2. 代码审查最佳实践

```
代码审查 Checklist:

□ 代码质量:
  - 代码风格一致
  - 命名清晰有意义
  - 无重复代码
  - 适当的注释

□ 功能正确性:
  - 实现符合需求
  - 边界条件处理
  - 错误处理完善
  - 无明显 Bug

□ 性能考虑:
  - 无性能问题
  - 合理使用缓存
  - 避免 N+1 查询

□ 安全性:
  - 输入验证
  - SQL 注入防护
  - XSS 防护
  - 敏感信息保护

□ 测试覆盖:
  - 关键路径有测试
  - 边界条件有测试
  - 测试用例清晰
```

### 2.1 GitHub Code Owners

```yaml
# .github/CODEOWNERS

# 全局所有者
*       @team-lead

# 前端代码
/src/frontend/   @frontend-team

# 后端代码
/src/backend/    @backend-team

# 数据库迁移
/db/migrations/  @dba-team

# CI/CD 配置
/.github/        @devops-team
/Jenkinsfile     @devops-team

# 文档
/docs/           @tech-writer
*.md             @tech-writer
```

## 3. 冲突解决

```
冲突类型与解决:

1. 内容冲突（Content Conflict）
   同一行被不同分支修改

2. 删除冲突（Delete Conflict）
   一个分支修改，另一个分支删除

3. 添加冲突（Add Conflict）
   两个分支添加同名文件

解决步骤:
1. git merge feature/login
2. 编辑冲突文件
3. git add <resolved-file>
4. git commit

冲突标记:
<<<<<<< HEAD
当前分支的内容
=======
要合并的分支内容
>>>>>>> feature/login
```

```bash
# 使用工具解决冲突
git mergetool

# 配置合并工具
git config --global merge.tool vimdiff
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait --merge $REMOTE $LOCAL $BASE $MERGED'

# 放弃合并
git merge --abort
```

## 4. Fork 工作流

```
Fork 工作流（开源项目贡献）:

┌─────────────────────────────────────────────────────────────┐
│                    Fork 工作流                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Fork 仓库                                               │
│     GitHub 页面点击 Fork 按钮                               │
│                                                             │
│  2. 克隆 Fork 的仓库                                        │
│     git clone https://github.com/your/repo.git              │
│                                                             │
│  3. 添加上游仓库                                            │
│     git remote add upstream https://github.com/original/repo.git │
│                                                             │
│  4. 同步上游更新                                            │
│     git fetch upstream                                      │
│     git checkout main                                       │
│     git merge upstream/main                                 │
│                                                             │
│  5. 创建功能分支                                            │
│     git checkout -b feature/fix-typo                        │
│                                                             │
│  6. 提交并推送                                              │
│     git add . && git commit -m "fix: typo"                  │
│     git push origin feature/fix-typo                        │
│                                                             │
│  7. 创建 Pull Request（到上游仓库）                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```bash
# Fork 后同步上游更新
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

## 5. Issue 与 PR 关联

```bash
# 在提交信息中关联 Issue
git commit -m "fix: 修复登录bug

Closes #123
Fixes #456
Resolves #789"

# 在 PR 描述中关联
Closes #123
Fixes #123
Resolves #123

# 关键字:
# Closes, Fixes, Resolves - 合并后自动关闭 Issue
# Relates to - 关联但不关闭
```

## 6. GitHub Actions 自动化

```yaml
# .github/workflows/pr-check.yml
name: PR Check

on:
  pull_request:
    branches: [ main, develop ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Build with Maven
      run: mvn -B clean verify
    
    - name: Run Tests
      run: mvn test
    
    - name: Code Coverage
      run: mvn jacoco:report
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
```

## 7. 协作规范

```
团队协作规范:

□ 提交规范:
  - 使用 Conventional Commits
  - 每次提交完成一个逻辑单元
  - 提交信息清晰描述变更

□ 分支规范:
  - 统一命名格式
  - 及时删除已合并分支
  - 避免直接提交到 main

□ PR 规范:
  - PR 标题简洁明了
  - PR 描述详细说明
  - 关联 Issue
  - 指定 Reviewer

□ 审查规范:
  - 至少一人审查
  - 自动化测试通过
  - 代码风格一致
  - 无明显问题

□ 合并规范:
  - Squash and merge（功能分支）
  - Rebase and merge（小修复）
  - 合并后删除分支
```
