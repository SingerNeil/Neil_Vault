# Day 6：流水线

对应资料：

- 学校 PPT：`Computer Organization Chapter04 for 24H.pdf`
- Lab：`Lab2 Pipelining for 24H.pdf`
- 视频：p125-p130、p132

## 1. 为什么要流水线

非流水线：

一条指令完整做完，下一条才开始。

流水线：

当前指令进入下一阶段后，下一条指令可以开始前一阶段。

流水线主要提升：

```text
throughput
```

不一定减少：

```text
single instruction latency
```

## 2. MIPS 五级流水线

五个阶段：

| 阶段 | 全称 | 作用 |
|---|---|---|
| IF | Instruction Fetch | 取指令，更新 PC |
| ID | Instruction Decode | 译码，读寄存器 |
| EX | Execute | ALU 运算，地址计算，比较 |
| MEM | Memory Access | 读/写数据存储器 |
| WB | Write Back | 写回寄存器 |

必须背：

```text
IF -> ID -> EX -> MEM -> WB
```

## 3. Pipeline Registers

流水线寄存器保存阶段之间的信息。

| 流水线寄存器 | 位置 | 常见内容 |
|---|---|---|
| IF/ID | IF 和 ID 之间 | instruction、PC+4 |
| ID/EX | ID 和 EX 之间 | 读出的寄存器值、立即数、控制信号 |
| EX/MEM | EX 和 MEM 之间 | ALU result、写存储器数据、控制信号 |
| MEM/WB | MEM 和 WB 之间 | memory data、ALU result、写回控制信号 |

Lab2 里问 pipeline register，就按这个思路答。

## 4. 理想流水线时序

假设没有 hazard：

```text
Cycle:  1   2   3   4   5   6   7
I1:     IF  ID  EX  MEM WB
I2:         IF  ID  EX  MEM WB
I3:             IF  ID  EX  MEM WB
```

第 n 条指令每个阶段依次向右移动。

## 5. 流水线加速比

理想情况下：

```text
Speedup ≈ number of pipeline stages
```

但现实中会被这些因素降低：

- pipeline register overhead。
- 不同阶段耗时不完全相等。
- structural hazard。
- data hazard。
- control hazard。
- stall 和 flush。

## 6. 三类 Hazard

### 6.1 Structural Hazard

硬件资源冲突。

例：

同一个 cycle 中，一条指令要取指，另一条指令要访问数据存储器，如果 instruction memory 和 data memory 是同一个资源，就冲突。

解决：

- 增加硬件资源。
- 分离 instruction memory 和 data memory。
- stall。

### 6.2 Data Hazard

后一条指令需要前一条指令尚未写回的结果。

例：

```text
add $t0, $t1, $t2
sub $t3, $t0, $t4
```

`sub` 需要 `$t0`，但 `add` 还没 WB。

解决：

- forwarding / bypassing。
- stall。

### 6.3 Control Hazard

branch/jump 改变 PC，导致已经取到的指令可能不该执行。

解决：

- stall。
- flush。
- branch prediction。
- 提前计算 branch decision。

## 7. Forwarding

Forwarding 的意思：

结果不用等到 WB 写回寄存器，而是从后面的流水线寄存器直接送到 ALU 输入。

典型来源：

- EX/MEM -> EX
- MEM/WB -> EX

典型用途：

```text
add $t0, $t1, $t2
sub $t3, $t0, $t4
```

`add` 的结果可以 forward 给 `sub` 的 EX 阶段。

## 8. Load-use Hazard

最经典需要 stall 的情况：

```text
lw  $t0, 0($t1)
add $t2, $t0, $t3
```

原因：

`lw` 的数据到 MEM 阶段末尾才拿到，下一条 `add` 在 EX 阶段就需要它。

即使用 forwarding，通常也要插入 1 个 bubble。

## 9. Stall、Bubble、Flush

Stall：

暂停流水线某些阶段，让后面等等。

Bubble：

插入一条空操作，什么也不做。

Flush：

清除错误路径上已经取到或部分执行的指令。

区别：

- data hazard 常用 stall/bubble。
- control hazard 常用 flush。

## 10. Lab2 做题模板

如果题目问某个 cycle 各阶段是什么：

1. 先画标准五级流水线表。
2. 每条指令从 IF 开始依次填 ID、EX、MEM、WB。
3. 遇到 hazard，插入 bubble。
4. 再读指定 cycle 的 IF/ID/EX/MEM/WB。

标准表：

```text
Cycle:  1   2   3   4   5   6   7   8
I1:     IF  ID  EX  MEM WB
I2:         IF  ID  EX  MEM WB
I3:             IF  ID  EX  MEM WB
I4:                 IF  ID  EX  MEM WB
```

如果第 13 个 cycle 问各阶段，就把表一直画到 cycle 13。

## 11. Day 6 自测

闭卷写出：

1. 五级流水线名称和作用。
2. 四个 pipeline registers。
3. 三类 hazard。
4. forwarding 是什么。
5. load-use hazard 为什么通常要 stall。
6. stall、bubble、flush 的区别。

