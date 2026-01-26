# Pulsar 项目 OTA 功能设计与实现文档 (MCU1)

**日期**：2026.01.26  
**适用模块**：Pulsar - MCU1 (OnStepX / 电机控制核心)  
**前置条件**：MCU2 已实现 Web OTA 功能

---

## 1. 概述

本文档描述 Pulsar 板载 **MCU1 (ESP32-PICO-MINI-02U)** 的 OTA 固件升级实现方案。MCU1 作为核心控制器运行 OnStepX 固件，负责望远镜电机控制、CAN 总线通信等关键任务。由于 MCU1 **没有直接的 WiFi 连接能力**（WiFi 功能由 MCU2 独占），其 OTA 升级需通过 **MCU2 作为中继代理** 完成。

## 2. 硬件架构分析

### 2.1 双 MCU 拓扑结构

```
[用户浏览器] --WiFi--> [MCU2: SmartWebServer] --UART--> [MCU1: OnStepX]
                       (ESP32-PICO)                        (ESP32-PICO)
                       WiFi AP 192.168.0.1                 电机控制/CAN总线
```

**关键硬件连接**：

| 信号名 | MCU2 侧引脚 | MCU1 侧引脚 | 功能说明 |
|--------|------------|------------|----------|
| `ONSTEP_TXD2` | GPIO21 (TX) | GPIO22 (RX) | MCU2 → MCU1 数据传输 |
| `ONSTEP_RXD2` | GPIO22 (RX) | GPIO21 (TX) | MCU1 → MCU2 数据传输 |
| `GND` | - | - | 共地 |

- **物理连接方式**：MCU2 的 `Serial2` 与 MCU1 的 `Serial2` 交叉连接（TX-RX / RX-TX）。
- **通信协议**：默认 115200 波特率，8N1 格式。
- **现有用途**：已用于 LX200 指令透传（如 `:GR#` 查询赤经）和 ESP Flasher 模式切换（历史遗留功能）。

### 2.2 为什么必须通过 UART？

1. **MCU1 无独立网络接口**：  
   Pulsar 硬件设计中，**仅 MCU2 配备天线和 WiFi 功能**，MCU1 的天线引脚未接出，无法建立独立的 WiFi 连接。
   
2. **USB 接口已被占用**：  
   MCU1 的 USB-to-Serial (TXD0/RXD0) 主要用于开发调试和首次烧录，野外使用时用户设备（手机/平板）无法直接连接 USB。
   
3. **现有通信信道复用**：  
   MCU2 与 MCU1 之间的 UART 通道已建立，复用该信道进行固件传输是最经济、最可靠的方案。

### 2.3 MCU1 Flash 分区策略

**当前分区表**：（需确认）  
根据 ESP32 标准实践，MCU1 应采用与 MCU2 相同的 **"Minimal SPIFFS (1.9MB APP with OTA)"** 分区表：

```
Name       Type  SubType  Offset   Size      Flags
nvs        data  nvs      0x9000   0x5000    -
otadata    data  ota      0xE000   0x2000    -
app0       app   ota_0    0x10000  0x1E0000  -
app1       app   ota_1    0x1F0000 0x1E0000  -
spiffs     data  spiffs   0x3D0000 0x30000   -
```

- **App0 (1.9MB)**：当前运行的 OnStepX 固件。
- **App1 (1.9MB)**：OTA 升级目标分区（备份分区）。
- **otadata**：记录当前启动分区的元数据，支持 A/B 切换和回滚。

**⚠️ 重要验证项**：  
在实施前必须通过 `esptool.py` 或 Arduino IDE 确认 MCU1 当前固件已使用支持 OTA 的分区表。如果分区表不支持双 App 分区，需先通过 USB 刷入带 OTA 分区表的 Bootloader 和初始固件。

---

## 3. 技术实现方案

### 3.1 整体流程概览

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 用户上传 MCU1 固件 (.bin) 到 MCU2 的 /update_mcu1 接口          │
│    [浏览器] --HTTP POST--> [MCU2: /update_mcu1]                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. MCU2 缓冲固件数据并分片处理                                        │
│    • 计算总大小和 MD5 校验和                                          │
│    • 将固件分割为 2KB 的数据帧                                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. MCU2 通过 UART 发送 OTA 协议帧给 MCU1                            │
│    [MCU2] --Serial2 (115200)--> [MCU1]                          │
│    • 开始帧 (START): 总大小、MD5、目标分区                             │
│    • 数据帧 (DATA): 序号、长度、数据块、CRC16                           │
│    • 结束帧 (END): 触发校验和切换                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. MCU1 接收并写入 Flash                                            │
│    • 初始化 OTA 分区 (Update.begin)                                 │
│    • 逐帧写入数据到 App1 分区                                         │
│    • 完成后校验 MD5，标记分区有效                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. MCU1 重启并从新分区启动                                            │
│    ESP.restart() -> Bootloader 切换到 App1 -> OnStepX 新版本运行    │
│    • 若启动失败 3 次，Bootloader 自动回滚到 App0                        │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 UART OTA 协议设计

#### 3.2.1 帧格式定义

**通用帧结构**：
```
+------+------+--------+---------+--------+------+
| 帧头  | 命令  | 序号    | 数据长度  | 数据体  | CRC16 |
| 2B   | 1B   | 2B     | 2B      | 0-2048B| 2B   |
+------+------+--------+---------+--------+------+
```

- **帧头 (0xAA55)**：固定魔术字，用于帧同步。
- **命令类型**：
  - `0x01` START（开始）
  - `0x02` DATA（数据块）
  - `0x03` END（结束）
  - `0x04` ABORT（中止）
  - `0x10` ACK（确认）
  - `0x11` NACK（重传请求）
- **序号 (Seq)**：从 0 开始递增，DATA 帧用于检测丢包。
- **CRC16**：使用 CRC-16/CCITT-FALSE 算法，覆盖整个帧（不含 CRC 本身）。

#### 3.2.2 关键帧详细说明

**START 帧** (MCU2 → MCU1)：
```c
struct {
  uint32_t totalSize;        // 固件总大小（字节）
  uint8_t  md5Hash[16];      // 固件完整的 MD5 校验值
  uint8_t  targetPartition;  // 0x00=Auto, 0x01=App0, 0x02=App1
} StartFrame;
```

**DATA 帧** (MCU2 → MCU1)：
```c
struct {
  uint16_t seq;              // 数据块序号
  uint16_t dataLen;          // 数据长度 (1-2048)
  uint8_t  data[2048];       // 实际固件数据
} DataFrame;
```

**ACK/NACK 帧** (MCU1 → MCU2)：
```c
struct {
  uint16_t seq;              // 对应的序号
  uint8_t  errorCode;        // 0x00=Success, 其他见错误码表
} ResponseFrame;
```

#### 3.2.3 错误码定义

| 错误码 | 含义 | 处理方式 |
|-------|------|---------|
| `0x00` | 成功 | 继续 |
| `0x01` | CRC 校验失败 | 重传该帧 |
| `0x02` | 序号不连续 | 重传缺失帧 |
| `0x03` | Flash 写入失败 | 中止升级 |
| `0x04` | 空间不足 | 中止升级 |
| `0x05` | MD5 校验失败 | 中止升级 |
| `0x06` | 超时 | 重传或中止 |

### 3.3 可靠性保障机制

#### 3.3.1 分层校验策略

1. **帧级 CRC16**：每帧独立校验，检测传输错误。
2. **序号连续性检查**：MCU1 检测 `seq` 是否连续，防止丢帧。
3. **整体 MD5 校验**：所有数据写入完成后，计算整个分区的 MD5 与 START 帧中的值对比。

#### 3.3.2 超时与重传机制

- **MCU2 发送超时**：每发送一帧后，等待 MCU1 的 ACK/NACK，超时 `500ms` 则重传，最多重传 `3` 次。
- **MCU1 接收超时**：若 `5` 秒内未收到下一帧，则认为传输中断，发送 ABORT 帧并清理 OTA 状态。

#### 3.3.3 Bootloader 自动回滚

ESP32 的 Bootloader 具备内置回滚功能：
- 新固件启动后必须调用 `esp_ota_mark_app_valid_cancel_rollback()` 标记为有效。
- 若新固件在启动阶段崩溃（如 `setup()` 中死循环），Bootloader 检测到连续启动失败 3 次后，会自动切换回旧分区，防止变砖。

---

## 4. 代码架构设计

### 4.1 MCU1 端实现（OnStepX）

#### 4.1.1 新增模块结构

```
src/telescope/ota/
├── OtaReceiver.h          // OTA 接收管理类
├── OtaReceiver.cpp        // 核心逻辑
├── UartProtocol.h         // UART 协议定义（帧格式、CRC 等）
└── UartProtocol.cpp       // 协议解析与封装
```

#### 4.1.2 核心类接口设计

```cpp
class OtaReceiver {
public:
  void init();                       // 初始化，注册 UART 中断
  void poll();                       // 主循环轮询，解析帧并处理
  
private:
  enum State {
    IDLE,                            // 空闲等待
    RECEIVING,                       // 正在接收数据
    VERIFYING,                       // 校验 MD5
    COMPLETED,                       // 完成，等待重启
    FAILED                           // 失败
  };
  
  State state = IDLE;
  Update &updater;                   // Arduino OTA 库实例
  uint8_t rxBuffer[2048];            // 接收缓冲区
  uint32_t totalSize;
  uint32_t receivedSize;
  uint16_t expectedSeq;
  MD5Builder md5;
  
  void handleStartFrame(const uint8_t* data);
  void handleDataFrame(const uint8_t* data);
  void handleEndFrame();
  void sendAck(uint16_t seq, uint8_t errorCode);
  void abort(uint8_t errorCode);
};
```

#### 4.1.3 关键流程伪代码

```cpp
void OtaReceiver::poll() {
  if (Serial2.available()) {
    parseFrame();  // 解析 UART 帧
  }
  
  switch (state) {
    case IDLE:
      // 等待 START 帧
      break;
      
    case RECEIVING:
      // 处理 DATA 帧
      if (frame.cmd == DATA) {
        // 1. 检查序号连续性
        if (frame.seq != expectedSeq) {
          sendNack(frame.seq, ERR_SEQ_MISMATCH);
          return;
        }
        
        // 2. 写入 Flash
        if (Update.write(frame.data, frame.dataLen) != frame.dataLen) {
          abort(ERR_FLASH_WRITE);
          return;
        }
        
        // 3. 更新进度
        receivedSize += frame.dataLen;
        expectedSeq++;
        
        // 4. 发送 ACK
        sendAck(frame.seq, ERR_SUCCESS);
      }
      break;
      
    case VERIFYING:
      // 校验 MD5
      if (md5.calculate() == expectedMD5) {
        Update.end(true);  // 标记分区有效
        state = COMPLETED;
      } else {
        abort(ERR_MD5_MISMATCH);
      }
      break;
      
    case COMPLETED:
      // 延迟 2 秒后重启
      delay(2000);
      ESP.restart();
      break;
  }
}
```

### 4.2 MCU2 端实现（SmartWebServer）

#### 4.2.1 Web 路由扩展

在现有 SmartWebServer 基础上新增：

```cpp
// 新增路由处理函数
server.on("/update_mcu1", HTTP_POST, 
  []() { /* 返回结果页 */ },
  handleMCU1Upload  // 上传处理回调
);

void handleMCU1Upload() {
  HTTPUpload& upload = server.upload();
  
  if (upload.status == UPLOAD_FILE_START) {
    // 1. 计算固件总大小和 MD5
    otaForwarder.begin(upload.totalLen);
    
    // 2. 发送 START 帧到 MCU1
    otaForwarder.sendStartFrame(upload.totalLen, md5Hash);
  }
  else if (upload.status == UPLOAD_FILE_WRITE) {
    // 3. 分片并发送 DATA 帧
    otaForwarder.sendDataChunk(upload.buf, upload.currentSize);
  }
  else if (upload.status == UPLOAD_FILE_END) {
    // 4. 发送 END 帧，等待 MCU1 校验完成
    otaForwarder.sendEndFrame();
  }
}
```

#### 4.2.2 转发器类设计

```cpp
class OtaForwarder {
public:
  bool begin(uint32_t totalSize);
  bool sendStartFrame(uint32_t size, const uint8_t* md5);
  bool sendDataChunk(const uint8_t* data, size_t len);
  bool sendEndFrame();
  
private:
  uint16_t currentSeq = 0;
  uint8_t retryCount = 0;
  const uint8_t MAX_RETRY = 3;
  const uint16_t ACK_TIMEOUT = 500;  // ms
  
  bool sendFrameWithRetry(const uint8_t* frame, size_t len);
  bool waitForAck(uint16_t expectedSeq);
};
```

---

## 5. 前端交互设计

### 5.1 更新页面改进

在现有 `/update.htm` 基础上增加目标选择：

```html
<form method="POST" action="/update" enctype="multipart/form-data">
  <label>升级目标:</label>
  <select name="target" id="target">
    <option value="mcu2">MCU2 (WiFi模块 - 本机)</option>
    <option value="mcu1">MCU1 (OnStepX - 电机控制)</option>
  </select>
  
  <label>固件文件:</label>
  <input type="file" name="firmware" accept=".bin" required>
  
  <button type="submit">开始升级</button>
</form>
```

**JavaScript 动态路由切换**：
```javascript
document.getElementById('target').addEventListener('change', (e) => {
  const form = document.querySelector('form');
  if (e.target.value === 'mcu1') {
    form.action = '/update_mcu1';
  } else {
    form.action = '/update';
  }
});
```

### 5.2 进度反馈机制

MCU2 通过 Server-Sent Events (SSE) 或轮询方式向浏览器推送实时进度：

```javascript
let progress = 0;
const eventSource = new EventSource('/ota_progress');
eventSource.onmessage = (e) => {
  progress = parseInt(e.data);
  document.getElementById('progressBar').value = progress;
  
  if (progress === 100) {
    alert('MCU1 升级完成，设备即将重启！');
    eventSource.close();
  }
};
```

---

## 6. 风险评估与应对

### 6.1 潜在风险点

| 风险点 | 影响 | 概率 | 应对措施 |
|-------|------|------|---------|
| **UART 传输中断** | 升级失败，需重试 | 中等 | • 超时重传机制<br>• 断点续传支持（记录已完成序号） |
| **MCU1 分区表不支持 OTA** | 无法升级 | 低 | • 编译时检查分区表<br>• 提供 USB 刷写指导文档 |
| **固件过大超过 1.9MB** | 写入溢出 | 低 | • 编译时警告<br>• MCU1 拒绝接收过大固件 |
| **新固件启动崩溃** | 功能丧失，需回滚 | 中等 | • 依赖 Bootloader 自动回滚<br>• 新固件必须调用 `mark_valid()` |
| **UART 波特率不匹配** | 通信失败 | 低 | • 硬编码 115200 波特率<br>• MCU2 检测握手失败时降低波特率重试 |
| **同时升级 MCU1 和 MCU2** | 系统全失效 | 极低 | • **禁止同时操作**<br>• 前端互斥锁定升级按钮 |

### 6.2 失败恢复策略

1. **传输失败**：  
   - 用户可在 Web 界面点击"重试"按钮，MCU2 重新上传固件。
   - MCU1 的旧固件仍然运行，不影响正常使用。

2. **启动失败**：  
   - Bootloader 自动回滚后，用户需通过 Web 界面查看错误日志（MCU1 通过 UART 回传错误信息）。
   - 若持续失败，提供 USB 刷写的降级方案。

3. **两台 MCU 失联**：  
   - 若 MCU2 在传输过程中复位，MCU1 会检测超时并清理 OTA 状态，恢复正常运行。
   - MCU2 重启后可重新发起升级。

---

## 7. 实施步骤与里程碑

### 7.1 第一阶段：基础框架（1 周）

- [ ] **MCU1 侧**：
  - 实现 `UartProtocol` 模块（帧解析、CRC 计算）。
  - 实现 `OtaReceiver::init()` 和基本状态机。
  - 单元测试：模拟接收 START 帧并初始化 OTA 分区。

- [ ] **MCU2 侧**：
  - 实现 `OtaForwarder` 基础类。
  - 添加 `/update_mcu1` 路由和文件上传处理。
  - 单元测试：发送 START 帧并验证 MCU1 响应。

### 7.2 第二阶段：数据传输与校验（1 周）

- [ ] **MCU1 侧**：
  - 完成 DATA 帧接收和 Flash 写入逻辑。
  - 实现序号连续性检查和 CRC 校验。
  - 集成 MD5 校验和 `Update.end()` 调用。

- [ ] **MCU2 侧**：
  - 实现固件分片和 DATA 帧发送。
  - 实现 ACK/NACK 响应解析和重传逻辑。
  - 计算并传递固件 MD5 值。

### 7.3 第三阶段：前端集成与测试（3 天）

- [ ] 更新 Web 页面，增加 MCU1/MCU2 选择器。
- [ ] 实现进度条和错误提示 UI。
- [ ] 端到端测试：
  - 正常升级流程（小固件 < 500KB）。
  - 大固件传输（1.5MB）。
  - 故意中断（拔掉电源）验证回滚。
  - 刷入错误固件验证错误提示。

### 7.4 第四阶段：优化与文档（2 天）

- [ ] 日志完善：MCU1 和 MCU2 的详细日志输出。
- [ ] 性能优化：调整分片大小以平衡速度和稳定性。
- [ ] 用户文档：编写操作手册和故障排查指南。
- [ ] 代码注释和开发者文档。

---

## 8. 性能预估

### 8.1 升级时间预估

假设固件大小为 `1.5 MB`，UART 波特率为 `115200 bps`：

- **理论传输速率**：115200 / 10 ≈ `11.52 KB/s`（考虑起始位/停止位）
- **实际有效速率**：约 `10 KB/s`（考虑帧开销、ACK 延迟）
- **预计传输时间**：1.5 MB / 10 KB/s = `150 秒` ≈ **2.5 分钟**
- **加上校验和重启**：总耗时约 **3 分钟**

### 8.2 优化方向

1. **提高波特率**：  
   - 尝试使用 `230400` 或 `460800` bps，可将时间缩短至 1-1.5 分钟。
   - 需测试高波特率下的误码率。

2. **压缩固件**：  
   - 使用 gzip 压缩 .bin 文件，MCU1 解压后写入（需额外 RAM）。

3. **并行校验**：  
   - 边传输边计算 MD5，减少校验等待时间。

---

## 9. 测试计划

### 9.1 功能测试用例

| 用例编号 | 测试场景 | 预期结果 | 测试状态 |
|---------|---------|---------|---------|
| TC-01 | 正常升级 1.0 MB 固件 | 成功写入，重启运行新版本 | ⬜ 未测试 |
| TC-02 | 上传 2.0 MB 固件（超限） | MCU1 拒绝并返回错误码 `0x04` | ⬜ 未测试 |
| TC-03 | 传输中故意断电 MCU2 | MCU1 超时中止，旧固件继续运行 | ⬜ 未测试 |
| TC-04 | 发送错误 CRC 的 DATA 帧 | MCU1 返回 NACK，MCU2 重传 | ⬜ 未测试 |
| TC-05 | 新固件在 `setup()` 中死循环 | Bootloader 回滚到旧分区 | ⬜ 未测试 |
| TC-06 | 同时打开两个浏览器升级 MCU1 | 第二个请求被拒绝（互斥锁） | ⬜ 未测试 |

### 9.2 压力测试

- **连续升级 10 次**：验证 Flash 擦写寿命和稳定性。
- **弱信号环境**：在 WiFi 信号边缘测试传输可靠性。
- **低电压测试**：在电池电量不足情况下测试是否会变砖。

---

## 10. 后续演进方向

### 10.1 短期改进

1. **断点续传**：  
   - 记录已传输的最后序号，传输中断后从该点继续，避免重新上传。

2. **版本校验**：  
   - 在 START 帧中增加固件版本号字段，MCU1 拒绝降级或重复刷写相同版本。

3. **固件签名验证**：  
   - 使用 RSA/ECDSA 签名，防止恶意固件注入。

### 10.2 长期规划

1. **无线 OTA 中继链**：  
   - 支持通过 MCU2 同时升级多个下游设备（如未来增加的 MCU3）。

2. **云端固件仓库**：  
   - MCU2 定期从云服务器拉取最新固件，自动提示用户升级。

3. **A/B 无缝切换**：  
   - 类似 Android 系统，后台下载并写入备份分区，下次重启直接切换，用户无感知。

---

## 11. 附录

### 11.1 相关文档

- [Pulsar-OTA 功能设计与实现文档 (MCU2).md](./Pulsar-OTA%20功能设计与实现文档%20(MCU2).md)
- [U1_引脚定义.md](./U1_引脚定义.md)
- [Arduino OTA 官方文档](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/ota.html)

### 11.2 代码仓库结构（规划）

```
OnStepX/
├── src/
│   ├── telescope/
│   │   └── ota/                     # 新增 OTA 模块
│   │       ├── OtaReceiver.h
│   │       ├── OtaReceiver.cpp
│   │       ├── UartProtocol.h
│   │       └── UartProtocol.cpp
│   └── pinmaps/
│       └── Pins.MaxESP3.h           # 已确认 SERIAL_B 定义
└── tools/
    └── ota_test_scripts/            # 测试脚本
        ├── send_fake_firmware.py    # 模拟 MCU2 发送固件
        └── verify_partition.sh      # 验证分区表
```

### 11.3 CRC16 参考实现

```cpp
uint16_t crc16_ccitt(const uint8_t* data, size_t len) {
  uint16_t crc = 0xFFFF;
  for (size_t i = 0; i < len; i++) {
    crc ^= (uint16_t)data[i] << 8;
    for (uint8_t j = 0; j < 8; j++) {
      if (crc & 0x8000)
        crc = (crc << 1) ^ 0x1021;
      else
        crc <<= 1;
    }
  }
  return crc;
}
```

---

**文档版本**：v1.0  
**编写者**：Pulsar 开发团队  
**审阅状态**：待审阅  
**下次更新**：实施第一阶段完成后
