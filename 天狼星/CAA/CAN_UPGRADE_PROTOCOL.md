# CAA Bootloader CAN 固件升级协议

## 一、概述

本协议定义了通过 CAN 总线对 CAA 设备进行固件升级的完整流程。协议采用命令-响应模式，支持固件数据传输与 CRC 校验。

上位机通过串口连接 OnStep 适配器板，由适配器将串口 XC 指令翻译为 CAN 帧发送至 CAA 设备；CAA 设备的响应帧经适配器转发回上位机串口。

### 1.1 通信参数

| 参数 | 值 | 说明 |
|------|-----|------|
| **上位机→设备 CAN ID** | 0x14* | 上位机发送ID |
| **设备→上位机 CAN ID** | 0x18* | 设备响应ID |
| **CAN 波特率** | 1000 kbps | CAN 经典模式 |
| **帧类型** | 标准帧 | 11位标识符 |
| **数据长度** | 8 字节 | 固定长度 |
| **CAN 模式** | Classic CAN | 非 CAN-FD 模式 |
| **串口波特率** | 115200 bps | 上位机↔适配器串口 |

### 1.2 链路拓扑

```
上位机(Python脚本)
    │  串口 115200bps
    │  XC 协议指令
OnStep 适配器板
    │  CAN 1Mbps
    │  Classic CAN
CAA 设备(Bootloader)
```

---

## 二、存储器布局

设备 Flash 总容量 128KB，分区如下：

| 区域 | 起始地址 | 大小 | Sector 范围 | 说明 |
|------|---------|------|------------|------|
| **Bootloader** | 0x0800 0000 | 24 KB | Sector 0-2 | 引导程序（只读） |
| **App Param** | 0x0800 6000 | 8 KB | Sector 3 | 配置参数区（升级标志、CRC等） |
| **Application** | 0x0800 8000 | 96 KB | Sector 4-15 | 应用程序区 |

> **注意**: 应用程序区从 0x08008000 开始，最大支持 96KB 固件。Bootloader 在上电时读取 App Param 区的 `update_flag`，若为 0x01 则进入升级模式，否则直接跳转应用。

---

## 三、协议命令定义

### 3.1 命令字

| 命令名称 | 命令字 | 方向 | 说明 |
|---------|--------|------|------|
| **CMD_UPDATE_REQ** | 0x31 | 上位机→设备 | 升级请求（握手） |
| **CMD_DATA_PACK** | 0x32 | 上位机→设备 | 固件数据包 |
| **CMD_CHECK_CRC** | 0x33 | 上位机→设备 | 校验并完成升级 |
| **CMD_RESPONSE** | 0xB0 | 设备→上位机 | 响应包 |

### 3.2 状态码

| 状态码 | 值 | 说明 |
|--------|-----|------|
| **STATUS_OK** | 0x00 | 操作成功 |
| **STATUS_ERROR** | 0x01 | 通用错误 |
| **STATUS_CRC_ERROR** | 0x02 | CRC 校验失败 |
| **STATUS_SEQ_ERROR** | 0x03 | 包序号错误 |

---

## 四、数据包格式

### 4.1 升级请求包（CMD_UPDATE_REQ = 0x31）

**功能**: 启动固件升级流程，传递固件元数据。设备收到后擦除应用区 Flash，完成后响应。

**数据包格式**（8字节）:

| 字节位置 | 字段 | 类型 | 说明 |
|---------|------|------|------|
| Byte[0] | cmd | uint8 | 0x31 |
| Byte[1] | version | uint8 | 固件版本号（当前填 0x01） |
| Byte[2:3] | total_packets | uint16 | 固件总包数（小端序） |
| Byte[4:7] | total_bytes | uint32 | 固件总字节数（小端序） |

**示例**（100包，4096字节）:
```
0x31 0x01 0x64 0x00 0x00 0x10 0x00 0x00
          └─┬─┘     └──────┬──────┘
          100包          4096字节
```

**注意事项**:
- 每包携带 **7字节** 有效数据，`total_packets = ceil(total_bytes / 7)`
- CAN 模式下握手包**不含** CRC32，CRC32 在最后校验阶段由 0x33 命令单独传递
- 设备收到后会**立即擦除**应用区 Flash（约 2-3 秒），期间持续广播 B0 心跳（每 500ms 一次）直到收到第一个 0x32 数据包
- 握手超时时间：**3000ms**，最多重试 5 次

---

### 4.2 数据包（CMD_DATA_PACK = 0x32）

**功能**: 传输固件数据，每包携带 7 字节有效数据。

**数据包格式**（8字节）:

| 字节位置 | 字段 | 类型 | 说明 |
|---------|------|------|------|
| Byte[0] | cmd | uint8 | 0x32 |
| Byte[1:7] | payload | uint8[7] | 固件数据（7字节，不足补 0x00） |

**示例**:
```
0x32 0x00 0x01 0x02 0x03 0x04 0x05 0x06
     └──────────────┬──────────────────┘
               7字节固件数据
```

**8对1 回复机制**（关键）:

设备每收到 **8 包**才回复一次 B0 响应，上位机发包策略如下：

```
块内包序（0-based）   上位机行为             适配器串口指令格式
─────────────────────────────────────────────────────────────
0 ~ 6  (前7包)        fire-and-forget        短格式 XC（无需等回复）
7      (第8包)        发送并等待 B0          长格式 XC（等待匹配回复）
```

- **短格式 XC**（前7包）：`:XC,143,32,xx,...#` — 适配器立即返回，不等 CAN 回复
- **长格式 XC**（第8包）：`:XC,143,32,xx,...,183,6000,B0,32,**,...#` — 等待设备 B0 帧

B0 响应中的 `packet_index` 为**当前块最后一个包的绝对序号**（1-based）。

**Flash 写入时机**:
- 缓冲区满 8KB（1个 Sector）时写入一次
- 收到全部数据（最后一包）时写入剩余缓冲

**数据包响应超时**: `750ms × 8 = 6000ms`，最多重试 5 次

---

### 4.3 校验包（CMD_CHECK_CRC = 0x33）

**功能**: 提交 CRC32 校验值，触发设备校验并跳转应用程序。

**数据包格式**（8字节）:

| 字节位置 | 字段 | 类型 | 说明 |
|---------|------|------|------|
| Byte[0] | cmd | uint8 | 0x33 |
| Byte[1:2] | reserved | uint16 | 保留（填 0x00） |
| Byte[3:6] | fw_crc32 | uint32 | 固件 CRC32（小端序） |
| Byte[7] | reserved | uint8 | 保留（填 0x00） |

**示例**（CRC32 = 0x78563412）:
```
0x33 0x00 0x00 0x12 0x34 0x56 0x78 0x00
               └──────┬──────────┘
                CRC32（小端序）
```

**校验超时**: **3000ms**，最多重试 3 次

---

### 4.4 响应包（CMD_RESPONSE = 0xB0）

**功能**: 设备对上位机命令的响应。

**数据包格式**（8字节）:

| 字节位置 | 字段 | 类型 | 说明 |
|---------|------|------|------|
| Byte[0] | cmd | uint8 | 0xB0 |
| Byte[1] | target_cmd | uint8 | 响应的目标命令（0x31/0x32/0x33） |
| Byte[2] | status | uint8 | 状态码（见 3.2） |
| Byte[3:6] | current_offset | uint32 | 0x32响应时为当前块末尾包序号；其他命令为已写入字节数 |
| Byte[7] | reserved | uint8 | 保留 |

**示例**（响应第8包数据，OK）:
```
0xB0 0x32 0x00 0x08 0x00 0x00 0x00 0x00
          └─┬─┘ └──────┬──────┘
           OK        包序号=8
```

**响应时机**:
- `CMD_UPDATE_REQ`：Flash 擦除完成后响应；擦除期间每 500ms 广播一次 B0 心跳
- `CMD_DATA_PACK`：每收满 **8 包**响应一次；最后一块不足 8 包时收到最后一包即响应
- `CMD_CHECK_CRC`：CRC 校验完成后响应，之后跳转应用程序

---

## 五、升级流程

### 5.1 完整流程图

```
上位机                              设备(Bootloader)
  │                                      │
  │────── CMD_UPDATE_REQ (0x31) ────────>│
  │                                      │ 1. 解析元数据，保存配置
  │<───── B0(0x31, OK) 心跳 ─────────────│ 2. 擦除应用区 Flash（约2-3秒）
  │<───── B0(0x31, OK) 心跳 ─────────────│    擦除期间每500ms广播一次心跳
  │<───── B0(0x31, OK) 心跳 ─────────────│ 3. 等待第一个0x32包
  │                                      │
  │── CMD_DATA_PACK(包1, fire-and-forget)>│
  │── CMD_DATA_PACK(包2, fire-and-forget)>│
  │── CMD_DATA_PACK(包3, fire-and-forget)>│
  │── CMD_DATA_PACK(包4, fire-and-forget)>│
  │── CMD_DATA_PACK(包5, fire-and-forget)>│
  │── CMD_DATA_PACK(包6, fire-and-forget)>│
  │── CMD_DATA_PACK(包7, fire-and-forget)>│
  │──── CMD_DATA_PACK(包8, 等待B0) ──────>│ 收满8包
  │<────── B0(0x32, OK, idx=8) ──────────│
  │                                      │
  │              ... (每8包一块) ...      │ (每满8KB写入一次Flash)
  │                                      │
  │── CMD_DATA_PACK(最后一包，等待B0) ───>│ 收到最后一包
  │<────── B0(0x32, OK, idx=N) ──────────│
  │                                      │
  │────── CMD_CHECK_CRC (0x33) ─────────>│ 1. 写入剩余缓冲数据
  │                                      │ 2. 验证接收字节数
  │                                      │ 3. 读Flash计算CRC32
  │                                      │ 4. 比对CRC32
  │<────── B0(0x33, OK) ─────────────────│ 5. 清除升级标志，响应OK
  │                                      │ 6. 跳转到应用程序
```

### 5.2 详细步骤说明

#### 第一阶段: 握手（Handshake）

**上位机发送升级请求**
```python
handshake = [
    0x31,                              # CMD_UPDATE_REQ
    0x01,                              # version
    total_packets & 0xFF,              # total_packets 低字节
    (total_packets >> 8) & 0xFF,       # total_packets 高字节
    total_bytes & 0xFF,                # total_bytes[0]
    (total_bytes >> 8) & 0xFF,         # total_bytes[1]
    (total_bytes >> 16) & 0xFF,        # total_bytes[2]
    (total_bytes >> 24) & 0xFF,        # total_bytes[3]
]
# total_packets = ceil(total_bytes / 7)
```

**设备处理流程**
1. 保存元数据（total_packets、total_bytes）到 App Param 区
2. 擦除应用区 Flash（Sector 4-15，96KB）
3. 擦除完成，进入 `STATE_RECEIVING`
4. 每 500ms 广播一次 B0(0x31, OK) 心跳，直到收到第一个 0x32 包

**超时处理**
- 上位机等待超时：**3000ms**，最多重试 5 次
- 握手成功后上位机立即开始发送数据包

---

#### 第二阶段: 数据传输（Transmission）

**包结构**
```python
DATA_PACK_PAYLOAD = 7  # 每包7字节有效数据
total_packets = (total_bytes + 6) // 7

pkg = [0x32] + firmware_chunk_7bytes
```

**8对1 发包机制**

每 8 包为一块，前 7 包使用短格式 XC 指令（fire-and-forget），第 8 包使用长格式 XC 指令等待 B0：

```
块内包号  串口指令格式                                  是否等B0
───────────────────────────────────────────────────────────────
  1~7    :XC,143,32,xx,xx,xx,xx,xx,xx,xx#              否（5ms超时丢弃）
   8     :XC,143,32,xx,xx,xx,xx,xx,xx,xx,183,6000,     是
              B0,32,**,**,**,**,**,**#
```

**上位机校验**
- 从 B0 响应中读取 `current_offset`（包序号），与预期值对比
- 若不一致，触发重发机制（最多重试 5 次）

**Flash 写入策略**
- 缓冲区积累满 **8192 字节**（1 个 Sector）时自动写入 Flash
- 收到全部数据后写入剩余缓冲区

---

#### 第三阶段: 校验与完成（Verification）

**计算固件 CRC32**
```python
import zlib
fw_crc32 = zlib.crc32(firmware_data) & 0xFFFFFFFF
# 等价于 CRC-32/ISO-HDLC，与设备端查表法一致
```

**发送校验包**
```python
check_crc_pkg = [
    0x33, 0x00, 0x00,
    fw_crc32 & 0xFF,
    (fw_crc32 >> 8) & 0xFF,
    (fw_crc32 >> 16) & 0xFF,
    (fw_crc32 >> 24) & 0xFF,
    0x00,
]
```

**设备校验流程**
1. 写入缓冲区剩余数据到 Flash
2. 验证 `bytes_received == total_bytes`，不一致则响应 `STATUS_ERROR`
3. 读取 Flash 从 `APP_START_ADDR` 起计算 CRC32
4. 与 `fw_crc32` 比对：
   - **通过**：清零 App Param 区（清除升级标志），响应 `STATUS_OK`，跳转应用
   - **失败**：擦除应用区，响应 `STATUS_CRC_ERROR`，保持升级模式等待重新升级

**响应等待超时**：**3000ms**，最多重试 3 次

---

## 六、错误处理

### 6.1 通信超时

| 场景 | 超时时间 | 最大重试 | 处理方法 |
|------|---------|---------|---------|
| 握手响应 | 3000ms | 5次 | 重发握手请求 |
| 数据块响应（8包一块） | 6000ms（750×8） | 5次 | 重发整块8包 |
| 校验响应 | 3000ms | 3次 | 重发校验包 |

### 6.2 状态码处理

| 状态码 | 场景 | 上位机处理 |
|--------|------|-----------|
| `STATUS_OK` | 正常 | 继续下一步 |
| `STATUS_SEQ_ERROR` | 包序号不匹配 | 重发当前块 |
| `STATUS_CRC_ERROR` | CRC 校验失败 | 重新开始整个升级流程 |
| `STATUS_ERROR` | 通用错误（Flash写入失败等） | 重试，超限后报错终止 |

### 6.3 CAN 跳转说明

设备在 CAN 中断回调（`HAL_FDCAN_RxFifo0Callback`）中**不直接处理协议**，而是将数据复制到缓冲区并设置 `pending` 标志，由主循环（`while(1)`）在线程上下文中调用 `Bootloader_ProcessRxData()`。

这是因为 `RUN_To_Application()` 中调用了 `__set_MSP()` 修改主栈指针，若在中断上下文执行会触发 HardFault 导致无法跳转。

---

## 七、时序要求

### 7.1 发送间隔

| 阶段 | 发送策略 |
|------|---------|
| 握手请求 | 收到心跳或超时后立即重试 |
| 数据包（前7包/块） | fire-and-forget，串口超时 5ms 后直接发下一包 |
| 数据包（第8包/块） | 发送后等待 B0，超时 6000ms |
| 校验包 | 收到最后一个数据块 B0 后立即发送 |

### 7.2 设备响应时间

| 命令 | 响应时间 | 说明 |
|------|---------|------|
| 握手响应 | 2~3 秒 | 包含 Flash 擦除时间，擦除期间每 500ms 发心跳 |
| 数据块响应 | < 50ms | 不涉及 Flash 写入时；涉及写入约 10~30ms |
| 校验响应 | < 2 秒 | 包含读 Flash 计算 CRC32 的时间 |

### 7.3 性能参考

以 ~61KB 固件（约 8710 包，1089 块）为例：

| 方案 | 每块耗时 | 总时间 |
|------|---------|-------|
| 1对1（每包等B0） | ~88ms | ~128s |
| 8对1（每块等B0，长格式nowait） | ~65ms | ~71s |
| 8对1（短格式XC + 5ms超时） | ~30ms | **~33s** |

---

### B. CRC32 查表法（设备端 C 语言实现）

```c
static const uint32_t crc32_table[16] = {
    0x00000000, 0x1DB71064, 0x3B6E20C8, 0x26D930AC,
    0x76DC4190, 0x6B6B51F4, 0x4DB26158, 0x5005713C,
    0xEDB88320, 0xF00F9344, 0xD6D6A3E8, 0xCB61B38C,
    0x9B64C2B0, 0x86D3D2D4, 0xA00AE278, 0xBDBDF21C
};

uint32_t CRC32_Calculate(uint8_t* data, uint32_t len) {
    uint32_t crc = 0xFFFFFFFF;
    uint32_t* data32 = (uint32_t*)data;
    uint32_t len32 = len / 4;

    for (uint32_t i = 0; i < len32; i++) {
        uint32_t val = data32[i];
        crc ^= val;
        // 每个字节处理两个4位nibble
        for (int n = 0; n < 8; n++) {
            crc = crc32_table[crc & 0x0F] ^ (crc >> 4);
        }
    }
    // 处理剩余字节
    uint8_t* remaining = (uint8_t*)(&data32[len32]);
    for (uint32_t i = 0; i < (len % 4); i++) {
        crc = crc32_table[(crc ^ remaining[i]) & 0x0F] ^ (crc >> 4);
        crc = crc32_table[(crc ^ (remaining[i] >> 4)) & 0x0F] ^ (crc >> 4);
    }
    return ~crc;
}
```

上位机 Python 端等价实现：
```python
import zlib
fw_crc32 = zlib.crc32(firmware_data) & 0xFFFFFFFF
```

**文档结束**
