## 💻 ThinkBook 14+ (Linux) 电池充电模式切换指南

### 1. 核心原理

在 Linux 内核中，联想笔记本通过 `ideapad_acpi` 驱动暴露硬件控制接口。通过修改 `/sys/bus/platform/drivers/ideapad_acpi/` 下的 `conservation_mode` 节点，可以直接控制电池的充电阈值。

- **0**: 关闭养护模式（允许充电至 **100%**，适合室外/差旅使用）。
    
- **1**: 开启养护模式（限制充电至 **60%-80%**，适合寝室/工位插电使用）。
    

---

### 2. 快捷指令 (Zsh 别名方案)

建议将以下代码添加到 `~/.zshrc` 中，实现一键切换。

Bash

```bash
# 编辑配置文件
nano ~/.zshrc
```

 *粘贴以下内容
 ThinkBook Battery Mode Switch*
 ```bash
alias bat-full='echo 0 | sudo tee /sys/bus/platform/drivers/ideapad_acpi/VPC2004:00/conservation_mode && echo "🚀 Mode: Full Charge (100%)"'
alias bat-save='echo 1 | sudo tee /sys/bus/platform/drivers/ideapad_acpi/VPC2004:00/conservation_mode && echo "🛡️ Mode: Conservation (80%)"'
```

```bash
# 生效配置
source ~/.zshrc
```

---

### 3. 操作手册

| **场景**        | **命令**                                                                    | **效果**                |
| ------------- | ------------------------------------------------------------------------- | --------------------- |
| **准备外带/出差**   | `bat-full`                                                                | 电池将充至 100%，确保最大续航。    |
| **回到寝室/插电办公** | `bat-save`                                                                | 限制充电上限，延长电池物理寿命，减少发热。 |
| **查询当前状态**    | `cat /sys/bus/platform/drivers/ideapad_acpi/VPC2004:00/conservation_mode` | `0` 为满充，`1` 为养护。      |

---

### 4. 注意事项 & 风险评估

- **路径检查**：若路径 `VPC2004:00` 报错，请使用 `ls /sys/bus/platform/drivers/ideapad_acpi/` 确认实际节点名称。
    
- **硬件寿命**：长期保持 100% 电量在高负载（如运行 AudioVisual 或 编译 C 项目）下会产生叠加热量，建议仅在必要时开启 `bat-full`。
    
- **系统权限**：此操作涉及内核文件系统修改，必须配合 `sudo` 使用。
    

---