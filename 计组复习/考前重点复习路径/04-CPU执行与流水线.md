# 04-CPU执行与流水线

优先级：高。流水线比复杂控制器更值得看，题型更集中。

## 对应视频

### CPU 执行过程

- `p98 6-1 中央处理器概述`
- `p99 6-2-1 指令的执行过程 — 指令执行的一般流程`
- `p100 6-2-2 指令的执行过程 — 指令周期`
- `p101 6-2-3 指令的执行过程 — 指令周期各阶段的数据流`

### 数据通路 / 控制器可选

- `p102 6-3-1 数据通路 — CPU内部单总线结构`
- `p103 6-3-2 数据通路 — 习题课（1）`
- `p104 6-3-2 数据通路 —习题课（2）`
- `p105 6-3-2 数据通路 — 习题课（3）`
- `p106 6-4-1 控制器 — CPU的时序及其控制方式`

### 流水线必看

- `p125 6-6-1 指令流水线概述`
- `p126 6-6-2 流水线数据通路`
- `p127 6-6-3 流水线冲突与处理（1）— 结构冲突与解决方案，控制冲突与解决方案`
- `p128 6-6-3 流水线冲突与处理（2）— 数据冲突`
- `p129 6-6-3 流水线冲突与处理（3）— 数据冲突的解决方案—插入气泡`
- `p130 6-6-3 流水线冲突与处理（4）— 数据冲突的解决方案—数据旁路`
- `p132 6-6-4 指令流水线—习题课`

跳过：

- `p131 6-6-3 流水线冲突与处理（5）— 进一步提升指令执行的并行性—超流水线技术和多发射技术`

## 对应 PPT 页码

### CPU / Datapath

- `Computer Organization Chapter04 for 24H.pdf` p.29-p.31：实现 MIPS 总览。
- `Computer Organization Chapter04 for 24H.pdf` p.32-p.34：instruction fetch / PC increment。
- `Computer Organization Chapter04 for 24H.pdf` p.35-p.36：decode / register file read。
- `Computer Organization Chapter04 for 24H.pdf` p.37-p.39、p.62：R-format datapath。
- `Computer Organization Chapter04 for 24H.pdf` p.40-p.46、p.63-p.64：`lw/sw` datapath。
- `Computer Organization Chapter04 for 24H.pdf` p.47-p.54、p.65：`beq` / branch / jump datapath。
- `Computer Organization Chapter04 for 24H.pdf` p.55-p.66：single-cycle complete datapath / control path。
- `Computer Organization Chapter04 for 24H.pdf` p.97-p.100、p.162：single-cycle vs multi-cycle。

### Pipeline

- `Computer Organization Chapter04 for 24H.pdf` p.101-p.110：pipeline 引入、执行时间对比。
- `Computer Organization Chapter04 for 24H.pdf` p.105-p.112、p.128：五级流水线。
- `Computer Organization Chapter04 for 24H.pdf` p.114-p.116：pipeline registers。
- `Computer Organization Chapter04 for 24H.pdf` p.117-p.129：cycle-by-cycle pipeline 图。
- `Computer Organization Chapter04 for 24H.pdf` p.131-p.135：structural hazard。
- `Computer Organization Chapter04 for 24H.pdf` p.136-p.146：data hazard、stall、forwarding。
- `Computer Organization Chapter04 for 24H.pdf` p.143-p.145：load-use hazard。
- `Computer Organization Chapter04 for 24H.pdf` p.147-p.158：control hazard、flush、branch prediction。
- `Computer Organization Chapter04 for 24H.pdf` p.159-p.162：pipeline summary。
- `Lab2 Pipelining for 24H.pdf` p.1-p.3：Lab2 题面。

## 做题重点

### 指令执行五阶段

```text
IF -> ID -> EX -> MEM -> WB
```

| 阶段 | 含义 |
|---|---|
| IF | 取指令，更新 PC |
| ID | 译码，读寄存器 |
| EX | ALU 运算 / 地址计算 |
| MEM | 读写数据存储器 |
| WB | 写回寄存器 |

### Pipeline register

必须会解释：

```text
Pipeline registers are temporary registers between adjacent pipeline stages.
They hold instruction, PC, register values, immediate values and control signals.
```

四个名字：

```text
IF/ID
ID/EX
EX/MEM
MEM/WB
```

### 三类 hazard

Structural hazard：

```text
同一周期两条指令要用同一个硬件资源。
解决：增加硬件或 stall。
```

Data hazard：

```text
后一条指令要用前一条还没写回的结果。
解决：forwarding 或 stall。
```

Control hazard：

```text
branch / jump 改变 PC，已经取到的指令可能不该执行。
解决：stall、flush、branch prediction。
```

Load-use hazard：

```text
lw 的结果到 MEM 后才可用，紧跟着用它通常要插入一个 stall。
```

### Lab2 cycle diagram

做题步骤：

1. 先把每条指令按 `IF ID EX MEM WB` 斜着排。
2. 看 data hazard，必要时插 stall。
3. 看 branch / jump，必要时考虑 delay slot 或 flush。
4. 问第 N 个 cycle，就竖着读第 N 列。

## 不值得现在深挖

- 微程序控制器 p111-p118。
- 异常与中断 p119-p124。
- 超流水线、多发射 p131。

## 练习目标

- Lab2 的两个简答必须会：pipeline register；single-cycle / multi-cycle / pipeline 比较。
- 能闭卷写三类 hazard 和解决方法。
- 能看懂一个五级流水线时序图。

