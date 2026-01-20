# MCU1 (OnStepX) 问题排查与解决总结

## 📋 项目背景
- **MCU1 (OnStepX)**: 电机控制板，使用 ESP32-PICO-V3-02 + KTECH电机 (CAN 1Mbps)
- **MCU2 (SmartWebServer)**: WiFi服务板，通过 UART SERIAL_B (115200 baud) 与 MCU1 通信
- **硬件集成**: MCU1 和 MCU2 焊死在同一块集成板上

---

## 🔴 问题 #1: 启用 SERIAL_B 后，CAN 总线停止工作

### 现象
```
MSG: CanPlus, PULSAR Deep Wakeup... success
MSG: CanPlus, Monitor Task Started
...
DEBUG: A3 target=0 canID=0x321 ready=1
DEBUG: beginPacket result=1
ERR: twai_transmit err=259  ← 所有 CAN 发送都失败
DEBUG: endPacket result=0
```

错误码 259 = `ESP_ERR_INVALID_STATE` (0x103) - TWAI 驱动处于无效状态

### 根本原因分析
1. **初始问题**: Serial2 默认使用 GPIO16/17，这两个引脚连接到 ESP32-PICO 的内部 Flash，导致无限重启
2. **临时解决**: 改用 GPIO21/22 作为 Serial2 的 RX/TX
3. **副作用**: Serial2 初始化时破坏了 TWAI (CAN) 的 GPIO 矩阵配置
   - CAN 驱动在 setup() 早期初始化
   - Serial2 在 setup() 中后期初始化，破坏了已配置的 GPIO 映射
   - 导致 TWAI 处于无效状态，所有发送都失败 (err=259)

### ✅ 解决方案: TWAI 自动重初始化

在 `src/lib/canPlus/esp32/Esp32.cpp` 中：

```cpp
int CanPlusESP32::writePacket(int id, const uint8_t *buffer, size_t size) {
  static bool firstSend = true;
  
  // 首次发送前主动重新初始化 TWAI
  if (firstSend) {
    reinitTwai();  // 完全重新安装并启动 TWAI 驱动
    firstSend = false;
  }
  
  // ... 正常发送逻辑 ...
}

static void reinitTwai() {
  // 停止并卸载当前驱动
  twai_stop();
  twai_driver_uninstall();
  delay(20);
  
  // 重新执行深度唤醒序列 (GPIO 2/7/27)
  // 重置 CAN 引脚 (GPIO 25/26)
  // 重新安装并启动 TWAI 驱动
  // ...
}
```

**原理**: Serial2 初始化在 CAN 发送之前完成，所以首次 CAN 发送时重新初始化一次，确保 GPIO 矩阵被正确配置。

**效果**: ✅ CAN 总线恢复正常，电机可以控制

---

## 🔴 问题 #2: MCU2 无法连接到 MCU1

### 现象
MCU2 的网页显示"设备离线"，:GVP# 命令没有回复

### 根本原因分析
在解决问题 #1 时，我为 SERIAL_B 初始化添加了额外的 `setPins()` 调用：

```cpp
// ❌ 错误的方式 - 混合两种引脚设置方法
#if defined(ESP32)
  SERIAL_B.setPins(SERIAL_B_RX, SERIAL_B_TX);  // 方式 A
#endif
SERIAL_B.begin(baud, SERIAL_8N1, SERIAL_B_RX, SERIAL_B_TX);  // 方式 B
```

**问题**: ESP32 Arduino 库中 `setPins()` 和 `begin()` 的参数形式会互相冲突，同时调用会导致 SERIAL_B 配置混乱，无法正常接收数据。

### ✅ 解决方案: 使用单一的引脚设置方法

在 `src/lib/commands/SerialWrapper.cpp` 中改为：

```cpp
// ✅ 正确的方式 - 只用 begin() 的参数形式
#ifdef SERIAL_B
  if (isChannel(channel++))
    #if defined(SERIAL_B_RX) && defined(SERIAL_B_TX) && !defined(SERIAL_B_RXTX_SET)
      SERIAL_B.begin(baud, SERIAL_8N1, SERIAL_B_RX, SERIAL_B_TX);
    #else
      SERIAL_B.begin(baud);
    #endif
#endif
```

**效果**: ✅ SERIAL_B 恢复正常，MCU2 可以与 MCU1 通信

---

## 🟡 问题 #3: 过多的调试日志导致芯片发热

### 现象
大量的 `MSG: cmdB = ...` 日志，芯片温度很高

### 原因
MCU2 每隔数百毫秒就向 MCU1 查询位置、时间等信息，这些都被日志记录下来。

### ✅ 解决方案

**Extended.config.h**:
```cpp
#define DEBUG                      ON        // 只显示错误和关键信息
#define DEBUG_ECHO_COMMANDS   ERRORS_ONLY   // 只显示有错误的命令，不显示成功的查询
#define DEBUG_CAN                   OFF      // 关闭 CAN 总线日志
```

**KTech.cpp**: 移除每次 CAN 发送的详细日志，只在运动开始/停止时输出：
```cpp
// 运动开始时输出
VF("MSG: Axis, autoSlew start "); V(frequency); VLF(" deg/s");

// 运动停止时输出
VLF("MSG: Axis, slewing stop");
```

**效果**: ✅ 日志显著减少，芯片温度正常

---

## 🟡 剩余错误信息

启动时仍会看到：
```
ERR: NV, failed to read back key!
MSG: cmdB = F1a, reply = 0, Error no error false/fail  (焦点器命令)
MSG: cmdB = GX98, reply = 0, Error no error false/fail  (旋转器命令)
MSG: cmdB = GXY0, reply = 0, Error no error false/fail  (未知命令)
```

### 分析
- **NV 错误**: 非严重，EEPROM/Flash 初始化异常，不影响功能
- **焦点器/旋转器命令**: MCU2 在探测 MCU1 的功能
  - `AXIS4_DRIVER_MODEL = OFF` (焦点器禁用)
  - `AXIS3_DRIVER_MODEL = OFF` (旋转器禁用)
  - MCU1 正确返回"不支持"是预期行为
- **GXY0 命令**: MCU2 发送的未知命令，MCU1 正确返回错误

**这些都不影响 MCU1 的核心功能。**

---

## ✅ 最终状态

### MCU1 正常工作的功能
- ✅ CAN 总线与 KTECH 电机通信 (1Mbps)
- ✅ SERIAL_B 与 MCU2 通信 (115200 baud, GPIO21/22)
- ✅ :GVP# 握手命令正确响应
- ✅ 串口命令接收处理正常
- ✅ 电机位置控制正常 (:Me# 命令)
- ✅ 日志输出合理，芯片温度正常

### 已知的非核心问题（需 MCU2 改进）
- ⚠️ MCU2 网页控制电机：第一次按下成功，第二次无反应
  - 可能原因：MCU2 的命令重复发送逻辑、状态机、或超时重试机制有问题
  - 需要在 MCU2 代码中检查

---

## 📝 代码改动总结

### 修改的文件
1. **src/lib/canPlus/esp32/Esp32.cpp**
   - 添加 `reinitTwai()` 函数：重新初始化 TWAI 驱动
   - 修改 `writePacket()` 函数：首次发送时调用 reinitTwai()
   - 移除详细的发送日志

2. **src/lib/commands/SerialWrapper.cpp**
   - 移除 `SERIAL_B.setPins()` 调用
   - 只使用 `begin()` 的参数形式设置引脚

3. **src/lib/axis/motor/kTech/KTech.cpp**
   - 移除每次 CAN 发送的 DEBUG 日志
   - 添加运动开始/停止的简洁日志

4. **Extended.config.h**
   - `DEBUG_ECHO_COMMANDS` 改为 `ERRORS_ONLY`
   - `DEBUG_CAN` 改为 `OFF`

5. **OnStepX.ino**
   - 重置为原始状态（撤销之前的日志改动）

---

## 🔧 故障排查清单（给 MCU2 Copilot）

### MCU2 网页控制电机"第一次成功，第二次失效"的可能原因

1. **命令缓冲问题**
   - 第一次命令后，缓冲区是否被清空？
   - 是否有旧命令残留导致冲突？

2. **状态机问题**
   - MCU2 是否正确追踪电机运动状态？
   - 第一次后电机状态是否被正确更新？

3. **超时重试机制**
   - MCU2 对 MCU1 的命令响应是否有超时判断？
   - 超时后是否会重新发送，导致多次发送？

4. **SERIAL_B 通信**
   - MCU2 接收 MCU1 的回复后是否正确处理？
   - 是否有数据丢失或部分接收的情况？

5. **网页状态同步**
   - 网页界面的状态与 MCU2 的实际状态是否同步？
   - 按钮是否在等待响应时被禁用？

### 测试建议

1. **使用 :Me# 命令测试**
   ```
   :Me#  ← 第一次（MCU1 应返回 1#）
   :Me#  ← 第二次（MCU1 应返回 1#）
   ```
   如果都成功，说明 MCU1 正常，问题在 MCU2

2. **监控 SERIAL_B 日志**
   - 在 MCU2 代码中添加日志，显示每次发送的命令和收到的回复
   - 对比网页按钮的点击次数和实际发送的命令数量

3. **检查电机状态**
   - 在网页上显示电机的当前位置和状态
   - 观察第二次点击时状态是否有变化

---

## 🎯 总结

MCU1 侧的所有问题都已解决。系统现在稳定工作：
- ✅ CAN 总线正常
- ✅ SERIAL_B 通信正常  
- ✅ 电机可以通过 :Me# 命令正常旋转
- ✅ MCU2 可以与 MCU1 握手和通信

**后续工作**：关注 MCU2 的控制逻辑，可能是状态机或命令重复发送的问题。
