# MCU 硬件存储规格与 OTA 传输策略分析

**硬件型号**: ESP32-PICO-MINI-02 (ESP32-PICO-V3-02 SiP)
**参考文档**: ESP32-PICO-MINI-02 Datasheet v0.5

---

## 1. MCU 核心存储资源 (基于 Datasheet)

根据 Datasheet 第 3 页（Section 1.1 Features & 1.2 Description）的明确记载，本模组的存储资源如下：

| 参数项                  | 规格值        | 来源章节      | 说明                                                           |
| :------------------- | :--------- | :-------- | :----------------------------------------------------------- |
| **SPI Flash (闪存)**   | **8 MB**   | 1.1 / 1.2 | 用于存储固件、文件系统 (SPIFFS/LittleFS) 及 NVS 数据。容量充足，支持双分区 (A/B) OTA。 |
| **PSRAM (伪静态随机存储器)** | **2 MB**   | 1.1 / 1.2 | 外部扩展内存，可用于 OTA 过程中大数据包的临时缓存。                                 |
| **SRAM (静态随机存储器)**   | **520 KB** | 1.1       | 芯片内部内存，用于数据和指令。                                              |
| **ROM**              | **448 KB** | 1.1       | 用于启动引导 (Booting) 和核心功能。                                      |
| **RTC SRAM**         | **16 KB**  | 1.1       | 低功耗模式下的数据保持。                                                 |

---

## 2. Flash 写入特性分析

OTA 的核心是将数据写入 SPI Flash。为了保证写入效率和稳定性，传输分片大小需参考 Flash 的物理特性。

### 2.1 物理页大小 (Page Size)
*   **规格**: **256 Bytes** `[基于通用 SPI NOR Flash 标准，Datasheet 未明确]`
*   **说明**: Datasheet 仅给出了 Flash 总容量 (8 MB)，未详细列出 Flash 芯片的内部架构参数（Page/Sector/Block Size）。
    *   *注：ESP32 系列集成的 Flash 通常遵循 JEDEC 标准，编程（Program）的最小原子单位为 1 页 (256 字节)。*

### 2.2 擦除扇区 (Sector Size)
*   **规格**: **4 KB (4096 Bytes)** `[基于通用 SPI NOR Flash 标准，Datasheet 未明确]`
*   **说明**: Flash 在写入前必须按扇区擦除。

---

## 3. UART 串口通信缓冲能力

OTA 数据通过 UART 接口传输。硬件缓冲区的容量决定了单次传输的抗溢出能力。

### 3.1 硬件 FIFO 大小
*   **规格**: **128 Bytes** `[基于 ESP32 架构标准，Datasheet 未明确]`
*   **说明**: 本 Datasheet (Section 1.1 Hardware) 仅列出了支持 UART 接口，未给出 FIFO 深度。
    *   *注：根据《ESP32 技术参考手册 (Technical Reference Manual)》第 13 章 UART 控制器章节，ESP32 的 UART TX/RX FIFO 均为 128 字节。*

### 3.2 软件缓冲机制
由于 OTA 分片 (512 Bytes) 大于硬件 FIFO (128 Bytes)，系统的稳定性依赖于：
1.  **中断服务**: ESP32 利用中断实时搬运 FIFO 数据至 RAM。
2.  **RAM 空间**: 520 KB SRAM + 2 MB PSRAM 提供了巨大的软件缓冲池，足以应对 512 Bytes 的瞬时数据流。

---

## 4. OTA 传输策略论证

基于上述硬件参数，我们实施了 **512 Bytes 分片传输 + 流控** 的 OTA 策略。

### 4.1 为什么选择 512 Bytes？
该数值是基于 Flash 物理特性的整数倍设计，而非随意选取：

$$ \text{OTA Chunk Size (512 B)} = 2 \times \text{Flash Page Size (256 B)} $$

*   **写入效率**: 每次接收的数据正好填满 **2 个物理页**。这避免了非对齐写入（例如传 100 字节，却占用 1 个页的编程周期），最大化了 Flash 的编程效率。
*   **传输开销**: 相比于“一页一页传”（256 Bytes），512 Bytes 方案减少了 50% 的握手（ACK/NACK）交互次数，提升了整体吞吐量。

### 4.2 为什么不会溢出？
虽然单次传输 (512 B) > 硬件 FIFO (128 B)，但通过以下机制保证可靠性：
1.  **分片流控**: 每发送 512 字节后，MCU2 会暂停发送，等待 MCU1 处理（写入 Flash）并回复 ACK。
2.  **软件搬运**: ESP32 的驱动层处理速度（240 MHz 主频）远高于串口传输速度（1 Mbps），在数据填满硬件 FIFO 前即可将其移入 RAM。

---

## 5. 遗留确认事项 (To-Do)

为了确保文档的绝对严谨，建议针对 Datasheet 未列出的参数进行二次确认：

- [ ] **确认 Flash Page Size**: 需查找 *ESP32-PICO-V3-02* 内部封装的 Flash 具体型号手册，确认其 Page Size 是否严格为 256 Bytes。
- [ ] **确认 UART FIFO**: 引用 *ESP32 Technical Reference Manual* 对应章节作为数据来源。

---
