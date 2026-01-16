# KTECH 电机操作指南

基于ESP32 MaxESP3 + TJA1050 CAN总线的KTECH电机控制

## 前提条件

- 固件已配置：`MOUNT_ENABLE_IN_STANDBY = ON` 
- 固件已配置：`DEBUG_CAN = ON`（用于调试CAN消息）
- Serial Monitor 连接参数：**115200 波特率**
- CAN 配置：1Mbps，GPIO13(RX)，GPIO15(TX)

---

## 初始化和验证

### 1. 连接 Serial Monitor

```
Arduino IDE → Tools → Serial Monitor (Ctrl+Shift+M)
设置波特率：115200
```

### 2. 观察启动日志

系统启动时应该看到：

```
MSG: CanPlus, CAN_ESP32 (TWAI) Start... success
MSG: CanPlus, start callback monitor task (rate 100ms priority 3)... success
MSG: TWAI, state=1 tx_err=0 rx_err=0 bus_err=0
MSG: Axis_KTech, 1 driver powered up
MSG: Axis_KTech, 2 driver powered up
```

**预期状态：**
- CanPlus 初始化成功
- TWAI 驱动启动成功
- 两个KTECH电机已启用（因为 MOUNT_ENABLE_IN_STANDBY=ON）

**如果看到错误：**
- `FAILED! twai_driver_install` → CAN引脚配置错误
- `No CAN interface!` → CAN总线未初始化
- `ERR: NV, failed to read back key!` → 执行NV清理（见下文）

---

## LX200 命令操作

### 命令格式
所有命令以 `:` 开始，以 `#` 结束

### 3. 获取当前位置

```
:GR#
回复: HH:MM:SS#  (赤经/方位角)

:GD#
回复: sDD*MM:SS#  (赤纬/高度角)

:GZ#
回复: DDD*MM:SS#  (方位角)
```

**例如：**
```
输入：:GR#
输出：12:30:45#
```

---

## 电机控制命令

### 4. 启用/禁用电机

**启用电机：**
```
:MS#
```
预期回复：`0#`（成功）

此时Serial Monitor应显示：
```
MSG: Axis_KTech, 1 driver powered up
MSG: Axis_KTech, 2 driver powered up
```

**禁用电机：**
```
:Qe#
```
预期回复：`0#`（成功）

### 5. Goto（转到指定位置）命令

**设置目标RA/Az：**
```
:Sr HH:MM:SS#
:Sd sDD*MM:SS#
```

**例如（转到北极星附近）：**
```
:Sr 02:31:49#
回复：1#

:Sd +89*15:51#
回复：1#
```

**执行 Goto：**
```
:MS#
```

**预期行为：**
- 回复：`0#`（成功开始Goto）
- Serial Monitor 显示 CAN 消息：
  ```
  MSG: TWAI, state=1 tx_err=0 rx_err=0 bus_err=0
  MSG: TWAI tx packet id=0x141  (AXIS1)
  MSG: TWAI tx packet id=0x142  (AXIS2)
  ```
- CAN分析仪显示CAN总线上有消息流动

**中止 Goto：**
```
:Q#
```
回复：`0#`

### 6. 家位置操作

**回到家位置（90°, 90°）：**
```
:Mh#
```

预期回复：`0#`

Serial Monitor 显示：
```
MSG: Axis_KTech, 1 driver powered up
MSG: Axis_KTech, 2 driver powered up
```

### 7. 跟踪控制

**启动跟踪：**
```
:Te#
```
回复：`0#`

**停止跟踪：**
```
:Td#
```
回复：`0#`

---

## 导轨引导速率命令

### 8. 设置导轨速率

**查询导轨速率：**
```
:Rg#
```
回复：`n#`（速率级别 0-9）

**设置导轨速率：**
```
:Rn#
```
（其中 n = 0 为最慢，9 为最快）

### 9. 手动移动（导轨）

**向北/上移动：**
```
:Mn#
```

**向南/下移动：**
```
:Ms#
```

**向东/右移动：**
```
:Me#
```

**向西/左移动：**
```
:Mw#
```

**停止移动：**
```
:Q#
```

**预期行为：**
- 每个方向命令应该发送 CAN 消息到对应电机
- Serial Monitor 显示：
  ```
  MSG: TWAI tx packet id=0x141  (AXIS1 - RA/Az)
  MSG: TWAI tx packet id=0x142  (AXIS2 - Dec/Alt)
  ```

---

## CAN 消息调试

### 10. 观察 CAN 消息

当 `DEBUG_CAN = ON` 时，每条 CAN 消息都会显示（每秒钟一次）：

```
MSG: TWAI, state=1 tx_err=0 rx_err=0 bus_err=0
```

**参数说明：**
- `state=1` → TWAI 驱动运行正常
- `tx_err=0` → 无传输错误
- `rx_err=0` → 无接收错误
- `bus_err=0` → 无总线错误

**如果出现错误：**
```
MSG: TWAI, state=2 tx_err=10 rx_err=5 bus_err=3
```
表示：CAN 总线有问题，检查物理连接和终端电阻

### 11. 使用 CAN 分析仪验证

**应该看到的 CAN 帧：**

| 命令 | CAN ID | 数据 | 说明 |
|------|--------|------|------|
| `:MS#` (Goto) | 0x141, 0x142 | `A3 xx xx xx xx xx xx xx` | 位置命令（AXIS1/2） |
| 启用电机 | 0x141, 0x142 | `88 00 00 00 00 00 00 00` | 电机启动 |
| 禁用电机 | 0x141, 0x142 | `80 00 00 00 00 00 00 00` | 电机停止 |
| 请求状态 | 0x141, 0x142 | `9A 00 00 00 00 00 00 00` | 状态查询 |

**期望的CAN总线活动：**
```
时间戳     CAN ID   DLC   数据
00.000    0x141     8    88 00 00 00 00 00 00 00  (启动AXIS1)
00.010    0x142     8    88 00 00 00 00 00 00 00  (启动AXIS2)
00.500    0x141     8    A3 10 27 00 00 F0 FF FF  (移动AXIS1)
00.510    0x142     8    A3 10 27 00 00 F0 FF FF  (移动AXIS2)
```

---

## 故障排除

### 问题1：CAN消息为0，Serial Monitor正常

**诊断：**
```
:MS#
```
如果得到回复 `6#`（already in goto）但CAN总线无消息

**解决方案：**

1. **检查电机是否启用：**
   ```
   :Mh#
   ```
   观察日志是否显示 `driver powered up`

2. **清理NV存储：**
   ```
   :NV#
   ```
   重启系统

3. **验证CAN引脚：**
   在 [Extended.config.h](Extended.config.h) 确认：
   ```cpp
   #define CAN_RX_PIN  13
   #define CAN_TX_PIN  15
   ```

### 问题2：编译错误 - "already in goto"

**原因：**系统被卡在goto状态（NV存储损坏）

**解决方案：**

1. **执行一次有效的Goto：**
   ```
   :Sr 12:00:00#
   :Sd +45:00:00#
   :MS#
   ```
   等待Goto完成

2. **或清除NV存储：**
   ```
   :NV#
   ```

3. **重启ESP32**：按板上的RESET按钮

### 问题3：TWAI 初始化失败

**Serial Monitor显示：**
```
MSG: CanPlus, CAN_ESP32 (TWAI) Start... FAILED! twai_driver_install err=265
```

**检查清单：**
- [ ] GPIO13（RX）是否被其他功能占用？
- [ ] GPIO15（TX）是否被其他功能占用？
- [ ] TJA1050模块是否连接正确？
- [ ] CAN终端电阻是否正确（120Ω在总线两端）？

---

## 快速参考 - 完整示例流程

```
① 启动系统，观察日志
② 查询位置
   :GR#
   :GD#

③ 启用电机
   :Ms#

④ 设置目标
   :Sr 14:00:00#
   :Sd +45:00:00#

⑤ 执行Goto
   :MS#
   （观察CAN分析仪，应该有消息）

⑥ 等待Goto完成
   :GR#    （查询，直到位置接近目标）

⑦ 停止跟踪
   :Td#

⑧ 禁用电机
   :Qe#
```

---

## 配置总结

**启用电机的关键配置：**

[Config.h](Config.h#L144)：
```cpp
#define MOUNT_ENABLE_IN_STANDBY  ON
```

**CAN调试的关键配置：**

[Extended.config.h](Extended.config.h#L43)：
```cpp
#define DEBUG_CAN  ON
```

**KTECH驱动配置：**

[Config.h](Config.h#L51)：
```cpp
#define AXIS1_DRIVER_MODEL  KTECH
#define AXIS2_DRIVER_MODEL  KTECH
```

[Extended.config.h](Extended.config.h#L46)：
```cpp
#define AXIS1_ENCODER  KTECH_IME
#define AXIS2_ENCODER  KTECH_IME
```

---
