
---


# SSH 免密码登录 SOP（Ubuntu → Linux / 嵌入式设备）

## 0. 前提检查

确保本机已安装 SSH 客户端：

```bash
ssh -V
````

确保可以正常密码登录：

```bash
ssh root@192.168.188.6
```

---

## 1. 生成 SSH 密钥对（本机执行）

```bash
ssh-keygen -t ed25519 -C "your_comment"
```

说明：

* 一路回车即可
* 默认路径：`~/.ssh/id_ed25519`
* passphrase 可以留空（否则每次仍需输入）

生成文件：

```text
~/.ssh/id_ed25519        # 私钥（必须保密）
~/.ssh/id_ed25519.pub    # 公钥
```

---

## 2. 拷贝公钥到远程设备

### 方法 A（推荐）

```bash
ssh-copy-id root@192.168.188.6
```

---

### 方法 B（手动）

```bash
cat ~/.ssh/id_ed25519.pub | ssh root@192.168.188.6 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

## 3. 修正远程权限（必须）

登录远程设备：

```bash
ssh root@192.168.188.6
```

执行：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## 4. 测试免密码登录

```bash
ssh root@192.168.188.6
```

* 不需要密码 → 成功
* 仍需密码 → 进入排查

---

## 5. 配置 SSH 别名（推荐）

编辑：

```bash
vim ~/.ssh/config
```

添加：

```text
Host mydevice
    HostName 192.168.188.6
    User root
    IdentityFile ~/.ssh/id_ed25519
```

使用：

```bash
ssh mydevice
```

---

## 6. （可选）禁用密码登录（安全强化）

编辑远程配置：

```bash
vim /etc/ssh/sshd_config
```

修改：

```text
PasswordAuthentication no
PermitRootLogin prohibit-password
```

重启：

```bash
systemctl restart ssh
```

⚠️ 注意：确保密钥登录已经成功，否则会锁死

---

# 故障排查

## 1. 强制使用指定私钥

```bash
ssh -i ~/.ssh/id_ed25519 root@192.168.188.6
```

---

## 2. 查看调试信息

```bash
ssh -v root@192.168.188.6
```

关注：

```text
Offering public key
Authentication refused
```

---

## 3. 检查远程权限

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## 4. 检查 sshd 配置

```bash
cat /etc/ssh/sshd_config | grep -E "PubkeyAuthentication|PermitRootLogin"
```

应为：

```text
PubkeyAuthentication yes
PermitRootLogin yes
```

---

## 5. root 登录策略说明

```text
PermitRootLogin prohibit-password
```

含义：

* 允许密钥登录
* 禁止密码登录

---

## 6. 嵌入式 / Dropbear 特殊路径

某些系统使用：

```bash
/etc/dropbear/authorized_keys
```

而不是：

```bash
/root/.ssh/authorized_keys
```

---

# 最简流程（快速版）

```bash
ssh-keygen -t ed25519
ssh-copy-id root@192.168.188.6
ssh root@192.168.188.6
```

---

# 结论

* `ssh-copy-id` 提示 key 已存在 → 属于正常现象
* 真正关键在于 SSH 认证阶段（权限 / 配置 / key 匹配）


---


