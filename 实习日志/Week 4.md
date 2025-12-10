
---
# Linux 工作站

账户：`eds`
密码：`20210929`

## 工作站配置概览

| 组件               | 规格                                 | 解读与性能定位                                                                     |
| :--------------- | :--------------------------------- | :-------------------------------------------------------------------------- |
| **处理器 (CPU)**    | AMD Ryzen 9 7945HX<br>（16核 / 32线程） | **移动端旗舰级**。性能堪比台式机CPU，非常适合编译代码、运行虚拟机、视频渲染等重度多任务工作。                          |
| **内存 (RAM)**     | 32GB                               | **充裕且高效**。当前仅使用约6GB，有大量空闲内存保证大型软件和多任务流畅运行。                                  |
| **硬盘 (Storage)** | 1TB NVMe SSD<br>（系统盘实际约935GB）      | **高速大容量**。NVMe协议带来极快的读写速度，开机、加载软件和文件传输都会非常快。                                |
| **显卡 (GPU)**     | AMD Radeon 集成显卡<br>（Raphael 架构）    | **日常与轻量图形足够**。驱动良好，能流畅支持 **4K分辨率(3840x2160)** 桌面和视频播放。对于大型3A游戏或专业3D渲染，性能有限。 |
| **系统 (OS)**      | Ubuntu (使用LVM逻辑卷管理)                | 系统分配合理，运行状态健康。                                                              |

#  安装 `.deb` 格式的软件包
## apt 安装
先通过 `cd`  命令访问到软件包所在的目录，然后执行
```
sudo apt install ./包名.deb
```
- 它可以自动处理依赖关系
- 注意：必须加上`./` ，否则apt会去软件源里面找！

# 嵌入式 Linux 交叉编译与远程调试 SOP

**项目路径:** `~/code/seeing`
**目标板 IP:** `192.168.124.12`
**工具链:** `arm-buildroot-linux-gnueabihf-` (Linux x86_64 Host)

---

## 1. 编译 (Build)
> 在 **电脑 (Host)** 终端执行

进入项目根目录，指定工具链路径进行编译。

```bash
cd ~/code/seeing

# 清理旧文件（可选，仅在修改了头文件或 Make 配置时需要）
make clean

# 执行交叉编译
make CROSS_COMPILE=$(pwd)/toolchain/bin/arm-buildroot-linux-gnueabihf-
````

- **成功标志:** `output/usr/local/bin/` 下生成绿色的 `indigo_server`。
    

---

## 2. 部署 (Deploy)

> 在 **电脑 (Host)** 终端执行

为了防止文件占用错误 (`Text file busy`) 和协议报错，先杀进程，再使用兼容模式 (`-O`) 传输。

Bash

```
# 1. 杀掉板子上的旧进程
ssh root@192.168.124.248 "killall -9 indigo_server"

# 2. 传输可执行文件 (注意 -O 参数用于兼容旧版 SSH)
scp -O ./output/usr/local/bin/indigo_server root@192.168.124.248:/usr/local/bin/

# 3. (可选) 如果修改了库文件，需同步传输 Lib
scp -O -r ./output/usr/local/lib/* root@192.168.124.248:/usr/local/lib/
```

---

## 3. 启动调试端 (Target Launch)

> 在 **板子 (Target)** SSH 终端执行

设置库搜索路径，并启动 GDB Server 监听端口。

Bash

```
# 设置环境变量并启动监听 (端口 2333)
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib && gdbserver :2333 /usr/local/bin/indigo_server
```

- **成功标志:** 终端显示 `Listening on port 2333`。
    

---

## 4. 开始调试 (Start Debugging)

> 在 **VS Code** 操作

1. 打开 C 代码文件（如 `indigo_server.c`）。
    
2. 在代码行号左侧点击，打上 **断点 (Red Dot)**。
    
3. 按键盘 **`F5`** 启动调试。
    

---

## 💡 常见问题备忘

- **Exec format error:** 检查工具链是否为 Linux x86_64 版本（用 `file` 命令查看），以及是否误用了 Mac 版工具链。
    
- **Permission denied (Build):** 检查 `toolchain/bin` 是否有执行权限 (`chmod -R +x toolchain/bin`).
    
- **Text file busy:** 这是因为板子上程序还在运行，必须先 `killall` 再 `scp`。
    
- **SCP 报错 Connection closed:** 必须加 `-O` 参数以支持旧版 SCP 协议。
    
- **GDB 缺库 (libpython2.7):** 在 `launch.json` 中将 `miDebuggerPath` 改为 `/usr/bin/gdb-multiarch`。

# 在Linux上 优雅地部署基于 clash/mihomo 的代理环境
详细完整版请查看：[Gitee开源地址](https://gitee.com/tools-repo/clash-for-linux-install)
