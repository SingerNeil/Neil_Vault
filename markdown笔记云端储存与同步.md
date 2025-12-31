
---

# 基于Github的Obsidian库免费同步方案

## 1.下载Obsidian，














---
# Git 冲突处理与手动同步核心手册

## 1. 为什么会产生冲突？

当同时在**多台设备**上对**同一个文件**进行了修改，导致各个设备上Git插件自动执行的pull和push出现冲突，或存在本地修改，但在自动pull前在Github上修改了同个文件的内容，Git 就会因为无法判断保留哪个版本而产生 `Conflict`。

---

## 2. 方案一：无脑以云端为准（放弃本地修改）

```bash
# 1. 获取远程所有分支的最新状态
git fetch --all

# 2. 强制将本地分支重置为远程分支状态
# 注意：这会永久抹掉本地未同步的修改！
git reset --hard origin/main

# 3. 清理掉本地产生的临时冲突文件或未追踪文件
git clean -fd
```

---

## 3. 方案二：手动解决冲突（保留两边内容）

当你发现两台电脑改的内容都有用，需要合并时：

1. **定位冲突**：在 Obsidian 中找到出现 `<<<<<<< HEAD` 标记的文件。
    
2. **编辑文件**：手动删除 Git 的标记符，保留需要的内容：
    
    - `<<<<<<< HEAD` (本地内容开始)
        
    - `=======` (本地与云端的分隔线)
        
    - `>>>>>>> origin/main` (云端内容结束)
        
3. **完成手动同步三部曲**：
    
    ```bash
    git add .
    git commit -m "Resolve conflict manually"
    git push origin main
    ```
    
---

## 4. 标准手动同步流程

若Giit插件不正常工作（如因网络环境改变导致的Git端口错误），则执行以下步骤。
不过还是***建议先直接重新配置好Git插件和网络端口设置。***

### **A. 开始写笔记前：若自动Git-pull插件没有生效，手动 Pull**

```bash
# 确保本地代码/笔记是全球最新的
git pull origin main
```

### **B. 写完笔记后：若自动Git-push插件没有生效，手动 Push**

```bash
# 1. 暂存所有修改
git add .

# 2. 提交到本地仓库（备注尽量简洁明了）
git commit -m "Update: [你的笔记标题或实习日志]"

# 3. 推送到 GitHub 云端
git push origin main
```

---

## 5. 快速排查命令

- `git status`：查看当前哪些文件在打架。
    
- `git log --oneline`：查看最近几次同步的记录。
    
- `git remote -v`：确认你的云端地址是否正确。
    

---
