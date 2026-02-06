
---

# Pulsar-CN1 单口双芯固件烧录协议规范 (v1.0)

**适用范围**: Pulsar 主控板 (MCU1: OnStepX / MCU2: SmartWebServer)
**接口标准**: CN1 (USB-Serial, 115200 Baud, 8-N-1)

---

## 1. 方案概述

本方案通过单一物理接口 (CN1) 实现对板载两颗 MCU 的固件更新。系统基于 **应用层 OTA (Application OTA)** 机制，通过软件指令触发，无需物理操作 Boot 键。

*   **MCU1 烧录**: 上位机 直接与 MCU1 握手，MCU1 进入 OTA 接收模式。
*   **MCU2 烧录**: 上位机 命令 MCU1 进入**透传模式 (Passthrough)**，建立 上位机 与 MCU2 的虚拟直连通道，进而触发 MCU2 的 OTA 接收模式。

---

## 2. 指令层协议 (ASCII Handshake)

在传输二进制固件前，必须通过 ASCII 指令完成模式切换与握手。

### 2.1 握手指令集

| 目标芯片     | 发送指令 (上位机 -> MCU) | 预期响应 (MCU -> 上位机)    | 描述                                            |
| :------- | :---------------- | :------------------- | :-------------------------------------------- |
| **MCU1** | `:FLASH_M1#`      | `FLASH_M1_READY#`    | MCU1 挂起后台任务，进入 OTA Trap，准备接收数据。               |
| **MCU2** | `:FLASH_M2#`      | `PASSTHROUGH_READY#` | **步骤 1**: MCU1 停止业务，进入串口透传死循环。                |
| **MCU2** | `:UPDATE_MODE#`   | `OTA_READY#`         | **步骤 2**: 指令穿过 MCU1 到达 MCU2，MCU2 进入 OTA Trap。 |

---

## 3. 传输层协议 (Binary Transport)

握手成功后，通信切换为二进制帧格式。所有多字节字段均采用 **小端序 (Little-Endian)**。

### 3.1 帧结构 (Frame Structure)

| 偏移 | 字段 | 长度 | 值/类型 | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| 0 | Header | 2B | `0x55 0xAA` | 固定帧头 |
| 2 | Command | 1B | `uint8_t` | 命令字 (见 3.2) |
| 3 | Sequence | 2B | `uint16_t` | 包序号 (0 ~ 65535) |
| 5 | Length | 2B | `uint16_t` | **Payload** 的长度 |
| 7 | Payload | N | `uint8_t[]` | 数据载荷 (见 4.0) |
| 7+N | CRC16 | 2B | `uint16_t` | 校验码 |

*   **CRC 算法**: CRC16-CCITT-FALSE (Poly: `0x1021`, Init: `0xFFFF`)。
*   **校验范围**: 从 **Command** 字节开始，至 **Payload** 结束 (即跳过 `0x55 0xAA` 帧头)。

### 3.2 命令字定义

| 命令字    | 名称        | 方向         | 描述           |
| :----- | :-------- | :--------- | :----------- |
| `0x01` | **START** | 上位机 -> MCU | 传输开始，包含元数据   |
| `0x02` | **DATA**  | 上位机 -> MCU | 固件数据分片       |
| `0x03` | **END**   | 上位机 -> MCU | 传输结束，触发校验    |
| `0x10` | **ACK**   | MCU -> 上位机 | 成功确认         |
| `0x11` | **NACK**  | MCU -> 上位机 | 失败/错误 (含错误码) |

---

## 4. 数据载荷定义 (Payload Definitions)

### 4.1 START 帧载荷 (25 Bytes, Packed)
用于初始化 Flash 分区。

```cpp
struct OtaStartPayload {
    uint32_t totalSize;    // 固件总大小 (Bytes)
    uint16_t blockSize;    // 传输块大小 (固定 512)
    uint16_t totalBlocks;  // 总块数
    uint8_t  partition;    // 目标分区 (0 = APP)
    uint8_t  md5Hash[16];  // 完整固件的 MD5 指纹
};
```

### 4.2 DATA 帧载荷 (2 + N Bytes)
用于传输固件切片。

```cpp
struct OtaDataPayload {
    uint16_t blockSeq;     // 当前块序号 (用于防乱序)
    uint8_t  data[N];      // 固件切片 (通常 N=512)
};
```

### 4.3 END 帧载荷 (16 Bytes)
用于触发最终校验。

```cpp
struct OtaEndPayload {
    uint8_t  md5Hash[16];  // 再次发送 MD5 以供比对
};
```

### 4.4 ACK 响应载荷 (4 Bytes)
MCU 回复确认。

```cpp
struct OtaAckPayload {
    uint8_t  ackedCmd;     // 被确认的命令字 (如 0x01, 0x02)
    uint16_t ackedSeq;     // 被确认的序号
    uint8_t  status;       // 0x00 = Success
};
```

---

## 5. 交互流程与流控 (Flow Control)

为适应 Flash 写入特性及防止缓冲区溢出，上位机**必须**遵循以下流控逻辑：

1.  **分片策略**: 固件必须按 **512 Bytes** 切片。
    *   *原理*: 512B 对应 2 个 Flash 物理页，且适配 ESP32 内存缓冲。
2.  **停等协议 (Stop-and-Wait)**:
    *   上位机发送一帧 `DATA` 后，**必须阻塞等待** 收到 MCU 的 `ACK`。
    *   若收到 `NACK` 或超时 (建议 5s)，应重试发送当前帧。
3.  **写入延迟 (Pacing)**:
    *   收到 `ACK` 后，上位机应主动延时 **10ms ~ 30ms** 再发送下一帧，给予 MCU 足够的 Flash 编程时间。

---

## 6. 安全保障机制 (Safety & Anti-Brick)

本方案采用三级防护策略，确保设备在传输中断、断电或误操作情况下**不“变砖”**。

1.  **A/B 分区冗余**:
    *   采用 ESP32 原生 OTA 双分区架构。新固件写入闲置分区 (App1)，当前运行的固件 (App0) 保持只读状态，不受任何影响。
2.  **原子性提交 (Atomic Commit)**:
    *   仅当所有数据块传输完成，且 **全量 MD5 校验** 通过后，MCU 才会修改 Bootloader 引导指针。
    *   若传输中途断开，引导指针未修改，重启后系统自动回滚至旧版本。
3.  **超时看门狗**:
    *   MCU 进入 OTA 模式后启动活跃检测计时器 (300s)。若上位机失去响应，MCU 自动复位，恢复正常运行。

---

## 7. 使用方法 (Python Tool)

使用提供的 `pulsar_master_flasher.py` 脚本进行操作。

### 7.1 配置参数
在脚本头部修改配置：
```python
SERIAL_PORT = '/dev/ttyUSB0'  # 或 COMx
TARGET_CHIP = 1               # 1=MCU1, 2=MCU2
FILE_PATH_MCU1 = "..."        # 固件路径
```

### 7.2 烧录 MCU1
1.  设置 `TARGET_CHIP = 1`。
2.  运行脚本。
3.  观察终端握手成功 -> 进度条走完 -> 显示 "烧录成功"。

### 7.3 烧录 MCU2
1.  设置 `TARGET_CHIP = 2`。
2.  运行脚本。
3.  脚本将自动执行：开启 MCU1 透传 -> 唤醒 MCU2 -> 传输数据。
4.  等待 MCU2 重启完成。
    *   *注：MCU2 烧录完成后，建议手动复位主板以退出 MCU1 的透传模式。*