# CAA Bootloader CAN 固件升级协议

## 一、概述

本协议定义了通过 CAN 总线对 CAA 设备进行固件升级的完整流程。协议采用命令-响应模式，支持固件数据传输、CRC校验和断点续传。

### 1.1 通信参数

| 参数 | 值 | 说明 |
|------|-----|------|
| **CAN ID** | 0x143 | 上位机和设备共用同一ID，双向通信 |
| **波特率** | 1000 kbps | CAN经典模式 |
| **帧类型** | 标准帧 | 11位标识符 |
| **数据长度** | 8 字节 | 固定长度 |
| **CAN模式** | Classic CAN | 非CAN-FD模式 |

---

## 二、存储器布局

设备Flash总容量 128KB，分区如下：

| 区域 | 起始地址 | 大小 | Sector范围 | 说明 |
|------|---------|------|-----------|------|
| **Bootloader** | 0x0800 0000 | 24 KB | Sector 0-2 | 引导程序 |
| **App Param** | 0x0800 6000 | 8 KB | Sector 3 | 配置参数区 |
| **Application** | 0x0800 8000 | 96 KB | Sector 4-15 | 应用程序区 |

> **注意**: 应用程序区从 0x08008000 开始，最大支持 96KB 固件

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
| **STATUS_CRC_ERROR** | 0x02 | CRC校验失败 |
| **STATUS_SEQ_ERROR** | 0x03 | 包序号错误 |

---

## 四、数据包格式

### 4.1 升级请求包（CMD_UPDATE_REQ = 0x31）

**功能**: 启动固件升级流程，传递固件元数据

**数据包格式**（8字节）:

| 字节位置 | 字段 | 类型 | 说明 |
|---------|------|------|------|
| Byte[0] | cmd | uint8 | 0x31 |
| Byte[1] | version | uint8 | 固件版本号 |
| Byte[2:3] | total_packets | uint16 | 固件总包数（小端序） |
| Byte[4:7] | total_bytes | uint32 | 固件总字节数（小端序） |

**示例**:
```
0x31 0x01 0x00 0x64 0x00 0x10 0x00 0x00
         └─┬─┘ └────┬───┘ └─────┬──────┘
          版本    100包      4096字节
```

**注意事项**:
- `total_packets`: 固件数据包的总数量，每包最大5字节有效数据
- `total_bytes`: 固件二进制文件的实际字节数
- CAN模式下不传输CRC32，在最后校验阶段传递
- 设备收到后会擦除应用区Flash，可能耗时数秒

---

### 4.2 数据包（CMD_DATA_PACK = 0x32）

**功能**: 传输固件数据

**数据包格式**（8字节）:

| 字节位置 | 字段 | 类型 | 说明 |
|---------|------|------|------|
| Byte[0] | cmd | uint8 | 0x32 |
| Byte[1:2] | packet_index | uint16 | 包序号（小端序，从0开始） |
| Byte[3:7] | payload | uint8[5] | 固件数据（5字节） |

**示例**:
```
0x32 0x00 0x00 0x01 0x02 0x03 0x04 0x05
     └───┬──┘ └──────┬──────────────┘
       包序号0        5字节固件数据
```

**重要规则**:
- `packet_index` 必须严格递增，从 0 开始
- 如果包序号不连续，设备将返回 `STATUS_SEQ_ERROR`
- 每包携带 5 字节有效数据（固定长度）
- 最后一包不足5字节时，剩余字节填充任意值（设备按实际长度写入）
- 设备每收满 8KB（一个Sector）或收到所有数据后才写入Flash

---

### 4.3 校验包（CMD_CHECK_CRC = 0x33）

**功能**: 提交CRC32校验值，触发设备校验和完成升级

**数据包格式**（8字节）:

| 字节位置 | 字段 | 类型 | 说明 |
|---------|------|------|------|
| Byte[0] | cmd | uint8 | 0x33 |
| Byte[1:2] | reserved | uint16 | 保留（填0x00） |
| Byte[3:6] | fw_crc32 | uint32 | 固件CRC32（小端序） |
| Byte[7] | reserved | uint8 | 保留（填0x00） |

**示例**:
```
0x33 0x00 0x00 0x12 0x34 0x56 0x78 0x00
               └──────┬──────────┘
                  CRC32=0x78563412
```

**CRC32计算规则**:
- 算法: CRC-32/ISO-HDLC
- 多项式: 0xEDB88320
- 初值: 0xFFFFFFFF
- 输入反转: 是
- 输出反转: 是
- 结果异或值: 0xFFFFFFFF
- 计算范围: 完整的 .bin 文件所有字节

---

### 4.4 响应包（CMD_RESPONSE = 0xB0）

**功能**: 设备对上位机命令的响应

**数据包格式**（8字节）:

| 字节位置 | 字段 | 类型 | 说明 |
|---------|------|------|------|
| Byte[0] | cmd | uint8 | 0xB0 |
| Byte[1] | target_cmd | uint8 | 响应的目标命令 |
| Byte[2] | status | uint8 | 状态码 |
| Byte[3:6] | current_offset | uint32 | 当前写入偏移量（小端序） |
| Byte[7] | reserved | uint8 | 保留 |

**示例**:
```
0xB0  0x32 0x00  0x00 0x04 0x00 0x00 0x00
     └─┬─┘ └─┬─┘ └──────┬──────────┘
   响应0x32  OK    已写入1024字节
```

**响应时机**:
- 收到 `CMD_UPDATE_REQ` 后响应
- 收到每个 `CMD_DATA_PACK` 后立即响应
- 收到 `CMD_CHECK_CRC` 后，校验通过后响应

---

## 五、升级流程

### 5.1 完整流程图

```
上位机                              设备(Bootloader)
  │                                      │
  │────── CMD_UPDATE_REQ (0x31) ───────> │
  │                                      │ 1. 解析元数据
  │                                      │ 2. 擦除Flash (约2-3秒)
  │                                      │ 3. 进入接收状态
  │<────── CMD_RESPONSE (OK) ────────────│
  │                                      │
  │────── CMD_DATA_PACK (idx=0) ────────>│
  │<────── CMD_RESPONSE (OK) ────────────│
  │                                      │
  │────── CMD_DATA_PACK (idx=1) ────────>│
  │<────── CMD_RESPONSE (OK) ────────────│
  │                                      │
  │              ...                     │ (每8KB写入一次Flash)
  │                                      │
  │────── CMD_DATA_PACK (idx=N-1) ──────>│
  │<────── CMD_RESPONSE (OK) ────────────│
  │                                      │
  │────── CMD_CHECK_CRC (0x33) ─────────>│
  │                                      │ 1. 写入剩余缓冲数据
  │                                      │ 2. 验证数据长度
  │                                      │ 3. 计算并比对CRC32
  │                                      │ 4. 清除升级标志
  │<────── CMD_RESPONSE (OK) ────────────│
  │                                      │
  │                                      │ 5. 跳转到应用程序
  │                                      │ 6. 系统运行新固件
```

### 5.2 详细步骤说明

#### 第一阶段: 握手（Handshake）

**1. 上位机发送升级请求**
```c
// 数据包内容
uint8_t data[8] = {
    0x31,                    // CMD_UPDATE_REQ
    0x01,                    // version
    (total_packets & 0xFF),  // total_packets低字节
    (total_packets >> 8),    // total_packets高字节
    (total_bytes & 0xFF),    // total_bytes[0]
    (total_bytes >> 8),      // total_bytes[1]
    (total_bytes >> 16),     // total_bytes[2]
    (total_bytes >> 24)      // total_bytes[3]
};
```

**2. 设备处理流程**
- 检查固件大小是否超出 96KB
- 擦除应用区 Flash（Sector 4-15）
- 保存元数据到配置区
- 响应 `STATUS_OK`

**3. 超时处理**
- 上位机等待响应超时: **5秒**（Flash擦除需要时间）
- 如超时，重新发送握手请求

---

#### 第二阶段: 数据传输（Transmission）

**1. 计算总包数**
```python
packet_size = 5  # 每包5字节有效数据
total_packets = (total_bytes + packet_size - 1) // packet_size
```

**2. 循环发送数据包**
```python
for idx in range(total_packets):
    offset = idx * 5
    payload = firmware[offset:offset+5]
    
    # 构造数据包
    data = [
        0x32,                # CMD_DATA_PACK
        idx & 0xFF,          # packet_index低字节
        (idx >> 8) & 0xFF,   # packet_index高字节
        payload[0], payload[1], payload[2], 
        payload[3], payload[4]
    ]
    
    # 发送并等待响应
    send_can_message(0x601, data)
    wait_for_response(timeout=1000ms)
```

**3. 响应处理**
- 收到 `STATUS_OK`: 继续发送下一包
- 收到 `STATUS_SEQ_ERROR`: 重发当前包
- 收到 `STATUS_ERROR`: 中止升级，重新握手
- 超时（1秒无响应）: 重发当前包，最多重试3次

**4. 进度计算**
```python
progress = (idx + 1) / total_packets * 100
```

---

#### 第三阶段: 校验与完成（Verification）

**1. 计算固件CRC32**
```python
import zlib
with open('firmware.bin', 'rb') as f:
    firmware_data = f.read()
    crc32 = zlib.crc32(firmware_data) & 0xFFFFFFFF
```

**2. 发送校验包**
```python
data = [
    0x33,                # CMD_CHECK_CRC
    0x00, 0x00,          # reserved
    crc32 & 0xFF,        # crc32[0]
    (crc32 >> 8) & 0xFF, # crc32[1]
    (crc32 >> 16) & 0xFF,# crc32[2]
    (crc32 >> 24) & 0xFF,# crc32[3]
    0x00                 # reserved
]
send_can_message(0x601, data)
```

**3. 设备校验流程**
- 写入缓冲区剩余数据到Flash
- 验证接收字节数 == total_bytes
- 读取Flash计算CRC32
- 比对CRC32
- **校验通过**: 清除升级标志，响应 `STATUS_OK`，跳转应用
- **校验失败**: 擦除应用区，响应 `STATUS_CRC_ERROR`，保持升级模式

**4. 响应等待**
- 超时时间: **3秒**（设备需要读取Flash计算CRC）
- 收到 `STATUS_OK`: 升级成功，设备将自动重启
- 收到 `STATUS_CRC_ERROR`: 升级失败，需要重新开始

---

## 六、错误处理

### 6.1 通信超时

| 场景 | 超时时间 | 处理方法 |
|------|---------|---------|
| 握手响应 | 5秒 | 重发握手请求，最多3次 |
| 数据包响应 | 1秒 | 重发当前包，最多3次 |
| 校验响应 | 3秒 | 重发校验包，最多3次 |

### 6.2 状态码处理

| 状态码 | 场景 | 上位机处理 |
|--------|------|-----------|
| `STATUS_SEQ_ERROR` | 包序号不匹配 | 重发当前包 |
| `STATUS_CRC_ERROR` | CRC校验失败 | 重新开始整个升级流程 |
| `STATUS_ERROR` | 通用错误 | 重新握手，重试3次后报错 |

### 6.3 断点续传

设备响应包中的 `current_offset` 字段可用于断点续传：
- 如果升级中断，上位机可读取该字段
- 重新握手后，从对应包序号继续传输
- **注意**: 当前实现不支持跨次升级的断点续传

---

## 七、时序要求

### 7.1 发送间隔
- **握手请求**: 无需等待，立即发送
- **数据包**: 收到上一包响应后立即发送下一包
- **校验包**: 收到最后一个数据包响应后立即发送

### 7.2 设备响应时间
- **握手响应**: < 3秒（包括Flash擦除时间）
- **数据包响应**: < 100ms
- **校验响应**: < 2秒（包括CRC计算时间）

---

### B. CRC32查表法（C语言参考）
```c
static const uint32_t crc32_table[16] = {
    0x00000000, 0x1DB71064, 0x3B6E20C8, 0x26D930AC,
    0x76DC4190, 0x6B6B51F4, 0x4DB26158, 0x5005713C,
    0xEDB88320, 0xF00F9344, 0xD6D6A3E8, 0xCB61B38C,
    0x9B64C2B0, 0x86D3D2D4, 0xA00AE278, 0xBDBDF21C
};

uint32_t crc32_calculate(uint8_t* data, uint32_t len) {
    uint32_t crc = 0xFFFFFFFF;
    
    for (uint32_t i = 0; i < len; i++) {
        crc ^= data[i];
        crc = crc32_table[crc & 0x0F] ^ (crc >> 4);
        crc = crc32_table[crc & 0x0F] ^ (crc >> 4);
    }
    
    return ~crc;
}
```

**文档结束**
