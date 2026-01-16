
## 问题根源

  

1. **串口命令不会触发CAN发送** - `:Me#` 等命令通过USB/UART处理，与CAN无关

2. **CAN只用于电机通信** - 只有在控制KTech电机时才会发送CAN帧

  

## TX引脚状态说明

  

- **空闲时保持高电平** - CAN协议的隐性（Recessive）状态，正常现象

- **发送数据时才会变化** - 只有当电机控制命令发送时才能在示波器上看到波形

- **需要电机初始化** - 电机控制器必须正确初始化并响应CAN通信

  

## 如何测试CAN通信

  

### 方法1：发送电机命令（需要真实KTech电机）

  

通过串口发送以下命令（这些命令会触发内部的CAN通信）：

  

```

:hF#      // 寻找原点（会发送CAN指令到电机）

:Q#       // 停止移动

:MS#      // 开始跟踪（会持续发送CAN指令）

:Me#      // 移动到东方（会发送CAN运动指令）

```

  

**注意**：这些串口命令会被OnStepX解析，然后**内部**转换为CAN命令发送给电机。

  

### 方法2：使用CAN_TX_Debug.ino（推荐）

  

已创建的 `CAN_TX_Debug.ino` 直接发送CAN帧，可以独立测试硬件：

- 每秒发送一个测试CAN帧

- 示波器应能看到TX引脚的CAN波形

- 不依赖电机存在

  

### 方法3：强制发送测试帧

  

在 Esp32.cpp 的 init() 函数末尾添加测试代码：

  

```cpp

// 测试：发送一个CAN帧

uint8_t testData[8] = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08};

writePacket(0x123, testData, 8);

VLF("Test CAN frame sent!");

```

  

## 检查电机是否连接

  

如果有KTech电机连接在CAN总线上：

1. 上传原始OnStepX固件

2. 发送 `:hF#` 命令启动寻找原点

3. 这时示波器上应该能看到TX引脚有CAN波形

4. 串口监视器应该有电机响应日志

  

## 调试输出

  

在 Esp32.cpp 中已经有调试输出：

```cpp

Serial.print("[CAN DEBUG] TX called, id=0x");

Serial.print(id, HEX);

```

  

如果电机命令被触发，串口监视器会显示 `[CAN DEBUG] TX called`。