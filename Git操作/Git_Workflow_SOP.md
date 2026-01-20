# Git/GitHub 版本管理 SOP

## 📋 分支策略

```
main (生产环境/稳定版本)
  ↑
develop (开发主线)
  ↑
feature/* (功能分支)
bugfix/*  (Bug修复分支)
hotfix/*  (紧急修复分支)
```

### 分支说明
- **main**: 生产环境代码，始终保持稳定可发布状态
- **develop**: 开发主线，集成所有完成的功能
- **feature/***: 新功能开发分支
- **bugfix/***: Bug 修复分支
- **hotfix/***: 生产环境紧急修复分支

---

## 🚀 完整工作流程

### 1️⃣ 开始新功能

```bash
# 确保 develop 是最新的
git checkout develop
git pull origin develop

# 从 develop 创建功能分支
git checkout -b feature/your-feature-name
```

**分支命名规范：**
- `feature/功能名` - 新功能开发
- `bugfix/问题描述` - Bug 修复
- `hotfix/紧急问题` - 生产环境紧急修复

**示例：**
```bash
git checkout -b feature/mcu2-wifi-config
git checkout -b bugfix/can-send-error
git checkout -b hotfix/critical-motor-stop
```

---

### 2️⃣ 开发过程中

#### 提交代码（小步快跑）
```bash
# 查看修改
git status
git diff

# 添加修改
git add .                    # 添加所有修改
git add specific-file.cpp    # 添加特定文件

# 提交
git commit -m "描述性的提交信息"

# 推送到远程（首次）
git push -u origin feature/your-feature-name

# 后续推送
git push
```

#### 提交信息规范
使用前缀清晰说明提交类型：

```
feat: 添加 MCU2 WiFi 配置功能
fix: 修复 CAN 总线发送问题
docs: 更新 README 文档
style: 格式化代码，无逻辑变更
refactor: 重构电机控制逻辑
perf: 优化通信性能
test: 添加单元测试
chore: 更新 .gitignore
build: 修改构建配置
ci: 更新 CI/CD 配置
```

#### 定期同步 develop 更新
```bash
# 切换到 develop 并更新
git checkout develop
git pull origin develop

# 回到功能分支
git checkout feature/your-feature-name

# 合并 develop 的更新
git merge develop

# 或使用 rebase（保持线性历史）
git rebase develop
```

---

### 3️⃣ 功能完成 - Pull Request 流程

#### 本地准备
```bash
# 1. 确保与 develop 同步
git checkout develop
git pull origin develop
git checkout feature/your-feature-name
git merge develop

# 2. 解决冲突（如有）
# 编辑冲突文件 → git add → git commit

# 3. 推送功能分支
git push origin feature/your-feature-name
```

#### 在 GitHub 创建 Pull Request

1. **打开仓库页面**，点击 "Pull requests" → "New pull request"

2. **选择分支**：
   - Base: `develop`
   - Compare: `feature/your-feature-name`

3. **填写 PR 描述**：
   ```markdown
   ## 功能描述
   实现了 MCU2 的 WiFi 配置功能
   
   ## 修改内容
   - 添加 WiFi 配置接口
   - 实现 Web 服务器控制
   - 修复 CAN 通信问题
   
   ## 测试情况
   - [x] 本地编译通过
   - [x] MCU1-MCU2 通信测试通过
   - [x] Web 控制界面正常
   
   ## 相关 Issue
   Closes #123
   ```

4. **请求代码审查**（Code Review）

5. **PR 审查通过后**，在 GitHub 上合并：
   - 点击 "Merge pull request"
   - 选择合并方式：
     - **Merge commit**: 保留完整提交历史
     - **Squash and merge**: 压缩为一个提交（推荐）
     - **Rebase and merge**: 线性历史
   - 勾选 "Delete branch" 删除远程分支

#### 清理本地分支
```bash
# 切换回 develop 并更新
git checkout develop
git pull origin develop

# 删除本地功能分支
git branch -d feature/your-feature-name

# 如果未合并但确定要删除
git branch -D feature/your-feature-name
```

---

### 4️⃣ 发布到生产环境

```bash
# 切换到 main 并更新
git checkout main
git pull origin main

# 合并 develop 到 main
git merge develop

# 打标签（版本号）
git tag -a v1.2.0 -m "Release version 1.2.0"

# 推送到远程
git push origin main
git push origin --tags

# 部署到生产环境（根据实际流程）
```

**语义化版本号规范（Semantic Versioning）：**
```
v主版本号.次版本号.修订号

v1.2.3
- 主版本号：不兼容的 API 修改
- 次版本号：向下兼容的功能新增
- 修订号：向下兼容的问题修复
```

---

## 🔥 紧急修复流程（Hotfix）

用于修复生产环境的严重 Bug，直接基于 main 分支创建。

```bash
# 1. 从 main 创建 hotfix 分支
git checkout main
git pull origin main
git checkout -b hotfix/critical-bug-description

# 2. 修复 Bug
# ... 编辑代码 ...
git add .
git commit -m "hotfix: 修复生产环境严重bug描述"

# 3. 合并到 main（生产环境）
git checkout main
git merge hotfix/critical-bug-description
git tag -a v1.2.1 -m "Hotfix: 修复xxx问题"
git push origin main --tags

# 4. 同步到 develop（避免下次发布丢失修复）
git checkout develop
git merge hotfix/critical-bug-description
git push origin develop

# 5. 删除 hotfix 分支
git branch -d hotfix/critical-bug-description
git push origin --delete hotfix/critical-bug-description
```

---

## 🛠️ 常用命令速查

### 分支操作
```bash
# 查看所有分支
git branch -a

# 创建并切换分支
git checkout -b branch-name
git switch -c branch-name    # 新语法

# 切换分支
git checkout branch-name
git switch branch-name       # 新语法

# 重命名分支
git branch -m old-name new-name

# 删除本地分支
git branch -d branch-name    # 安全删除（已合并）
git branch -D branch-name    # 强制删除

# 删除远程分支
git push origin --delete branch-name

# 查看已合并的分支
git branch --merged develop
git branch --no-merged develop
```

### 同步操作
```bash
# 拉取远程更新
git fetch origin             # 仅获取，不合并
git pull origin develop      # 获取并合并

# 推送到远程
git push origin branch-name
git push -u origin branch-name  # 首次推送并设置上游

# 强制推送（谨慎使用）
git push -f origin branch-name
```

### 提交操作
```bash
# 修改最后一次提交信息
git commit --amend -m "新的提交信息"

# 查看提交历史
git log --oneline
git log --oneline --graph --all

# 查看差异
git diff                     # 工作区 vs 暂存区
git diff HEAD                # 工作区 vs 最新提交
git diff branch1 branch2     # 两个分支的差异
git diff --name-only HEAD~1 HEAD  # 只显示文件名
```

### 撤销操作
```bash
# 撤销工作区修改
git checkout -- filename
git restore filename         # 新语法

# 撤销暂存
git reset HEAD filename
git restore --staged filename  # 新语法

# 回退到上一次提交（未推送时）
git reset --hard HEAD~1

# 回退到指定提交
git reset --hard commit-hash
```

### 清理操作
```bash
# 清理已删除的远程分支引用
git fetch --prune

# 查看远程仓库信息
git remote -v

# 查看分支跟踪关系
git branch -vv
```

---

## ✅ 最佳实践

### 1. 分支管理
- ❌ Never commit directly to `main`
- ✅ 所有功能通过 PR 合并到 develop
- ✅ 功能完成后立即删除分支
- ✅ 定期同步 develop，避免大规模冲突

### 2. 提交规范
- ✅ 小而频繁的提交（每个提交做一件事）
- ✅ 清晰的提交信息（使用规范前缀）
- ✅ 提交前检查 `git status` 和 `git diff`
- ❌ 避免提交编译产物、临时文件

### 3. 协作流程
- ✅ 使用 Pull Request 进行代码审查
- ✅ PR 描述清晰：做了什么、为什么、如何测试
- ✅ 解决所有 review 意见后再合并
- ✅ 合并后及时同步本地 develop

### 4. .gitignore 管理
```gitignore
# 编译和构建产物
.pio/
build/
*.elf
*.bin
*.hex
*.o

# IDE 配置
.vscode/
.idea/
*.iml

# 系统文件
.DS_Store
Thumbs.db

# 依赖
node_modules/
vendor/
```

### 5. 冲突处理
```bash
# 发生冲突时
git status                   # 查看冲突文件

# VS Code 会显示：
# <<<<<<< HEAD (当前分支)
# 你的代码
# =======
# 传入的代码
# >>>>>>> branch-name

# 选择保留的代码，删除标记
git add resolved-file.cpp
git commit
```

### 6. 安全措施
```bash
# 操作前创建备份分支
git branch backup-$(date +%Y%m%d)

# 查看将要合并的内容
git diff target-branch

# 使用 --dry-run 预览操作
git merge --no-commit --no-ff feature-branch
# 检查无误后
git merge --abort  # 取消
# 或
git commit         # 确认
```

---

## 📚 场景示例

### 场景 1: 开发新功能
```bash
git checkout develop
git pull origin develop
git checkout -b feature/new-sensor-driver

# ... 开发 ...
git add .
git commit -m "feat: 添加新传感器驱动"
git push -u origin feature/new-sensor-driver

# 在 GitHub 创建 PR: feature/new-sensor-driver → develop
# 审查通过后合并，然后：

git checkout develop
git pull origin develop
git branch -d feature/new-sensor-driver
```

### 场景 2: 修复 Bug
```bash
git checkout develop
git checkout -b bugfix/motor-stop-issue

git commit -m "fix: 修复电机意外停止问题"
git push -u origin bugfix/motor-stop-issue

# PR 合并后清理
git checkout develop
git pull origin develop
git branch -d bugfix/motor-stop-issue
```

### 场景 3: 紧急修复生产问题
```bash
git checkout main
git checkout -b hotfix/critical-memory-leak

git commit -m "hotfix: 修复内存泄漏导致的系统崩溃"

git checkout main
git merge hotfix/critical-memory-leak
git tag -a v1.2.1 -m "Hotfix: 修复内存泄漏"
git push origin main --tags

git checkout develop
git merge hotfix/critical-memory-leak
git push origin develop

git branch -d hotfix/critical-memory-leak
```

### 场景 4: 合并时发生冲突
```bash
git checkout develop
git pull origin develop
git checkout feature/my-feature
git merge develop

# 出现冲突：
# CONFLICT (content): Merge conflict in src/motor.cpp

# 打开 src/motor.cpp 解决冲突
# 编辑完成后：
git add src/motor.cpp
git commit -m "merge: 解决与 develop 的合并冲突"
git push
```

---

## 🎯 团队协作建议

### Code Review 检查清单
- [ ] 代码逻辑正确，无明显 bug
- [ ] 符合项目代码规范
- [ ] 有必要的注释和文档
- [ ] 没有调试代码（console.log, print 等）
- [ ] 没有提交敏感信息（密码、密钥）
- [ ] 测试通过
- [ ] 没有不必要的文件变更

### PR 模板示例
```markdown
## 变更类型
- [ ] 新功能
- [ ] Bug 修复
- [ ] 性能优化
- [ ] 文档更新
- [ ] 重构

## 变更描述
简要说明这个 PR 做了什么

## 测试情况
- [ ] 本地测试通过
- [ ] 单元测试通过
- [ ] 集成测试通过

## 相关 Issue
Closes #issue_number

## 截图/演示
（如果适用）

## 其他说明
需要特别注意的地方
```

---

## 🔍 故障排查

### 推送被拒绝
```bash
# 错误：remote: error: GH006: Protected branch update failed
# 原因：尝试直接推送到受保护分支（main）
# 解决：通过 PR 流程合并

# 错误：Updates were rejected (non-fast-forward)
# 原因：远程有新提交
git pull --rebase    # 拉取并变基
git push            # 重新推送
```

### 合并冲突
```bash
# 查看冲突文件
git status

# 取消合并
git merge --abort

# 或解决冲突后继续
git add .
git commit
```

### 误操作恢复
```bash
# 查看操作历史
git reflog

# 恢复到某个状态
git reset --hard HEAD@{2}

# 恢复误删的分支
git checkout -b recovered-branch commit-hash
```

---

## 📖 参考资源

- [Git 官方文档](https://git-scm.com/doc)
- [GitHub Flow](https://guides.github.com/introduction/flow/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)

---

**最后更新：** 2026-01-20
