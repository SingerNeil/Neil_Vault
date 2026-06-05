# Day 5：处理器数据通路

对应资料：

- 学校 PPT：`Computer Organization 04 Part1 for 24H.pdf`
- 学校 PPT：`Computer Organization Chapter04 for 24H.pdf`
- 作业：`Homework4 of Chapter04 for 24H.pdf`
- 视频：
  - `6-1 中央处理器概述`
  - `6-2-1 指令的执行过程 — 指令执行的一般流程`
  - `6-2-2 指令的执行过程 — 指令周期`
  - `6-2-3 指令的执行过程 — 指令周期各阶段的数据流`
  - `6-3-1 数据通路 — CPU内部单总线结构`
  - `6-3-2 数据通路 — 习题课（1）`
  - `6-3-2 数据通路 —习题课（2）`
  - `6-3-2 数据通路 — 习题课（3）`
  - `6-4-1 控制器 — CPU的时序及其控制方式`

页码说明：下面的 `PPT p.X` 指 PDF 文件打开后的实际页序。

## PPT 页码快速索引

| 知识点 | 对应 PPT 页码 |
|---|---|
| Chapter 4 范围、MIPS register/memory 回顾 | `Computer Organization Chapter04 for 24H.pdf` p.3-p.21 |
| latches / flip-flops / edge-triggered methodology | `Computer Organization Chapter04 for 24H.pdf` p.27-p.28 |
| 实现 MIPS、fetch/execute 总览 | `Computer Organization Chapter04 for 24H.pdf` p.29-p.31 |
| instruction fetch / PC increment | `Computer Organization Chapter04 for 24H.pdf` p.32-p.34 |
| decode / register file read | `Computer Organization Chapter04 for 24H.pdf` p.35-p.36 |
| R-format datapath | `Computer Organization Chapter04 for 24H.pdf` p.37-p.39、p.62 |
| `lw/sw` datapath | `Computer Organization Chapter04 for 24H.pdf` p.40-p.46、p.63-p.64 |
| `beq` / branch / jump datapath | `Computer Organization Chapter04 for 24H.pdf` p.47-p.54、p.65 |
| single-cycle complete datapath / mux / control path | `Computer Organization Chapter04 for 24H.pdf` p.55-p.66 |
| multicycle overview and RTL summary | `Computer Organization Chapter04 for 24H.pdf` p.67-p.90 |
| single-cycle vs multicycle comparison | `Computer Organization Chapter04 for 24H.pdf` p.97-p.100、p.162 |
| LWI 扩展题参考 | `Homework4 of Chapter04 for 24H.pdf` p.1；对照 datapath p.55-p.66 |

## 1. 单周期 MIPS 的核心部件

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.21-p.33、p.55-p.60；完整图也见 `Computer Organization Chapter04 for 24H.pdf` p.61-p.65。

必须会认：

- PC：保存当前指令地址。
- Instruction Memory：根据 PC 取指令。
- Register File：读写寄存器。
- ALU：做算术/逻辑/地址计算/比较。
- Data Memory：给 `lw/sw` 访问数据。
- Sign Extend：把 16 位立即数扩展到 32 位。
- Shift left 2：branch offset 左移 2 位。
- Add：计算 `PC + 4` 和 branch target。
- Mux：多路选择器。
- Control：根据 opcode 产生控制信号。

## 2. 一条指令的基本阶段

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.30-p.35；完整 Chapter04 的 fetch/decode/datapath 见 `Computer Organization Chapter04 for 24H.pdf` p.30-p.36；五阶段命名和流水线对比见 p.104-p.112。

单周期里逻辑上仍然可以分成：

1. Fetch：取指。
2. Decode：译码，读寄存器。
3. Execute：ALU 执行。
4. Memory：访存。
5. Write back：写回寄存器。

单周期处理器的特点：

- 每条指令一个 clock cycle 完成。
- cycle time 必须足够容纳最慢指令。
- 简单但效率低。

## 3. 常见控制信号

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.55-p.60；执行 add/lw/sw/beq 的控制路径见 `Computer Organization Chapter04 for 24H.pdf` p.62-p.65。

| 信号 | 含义 |
|---|---|
| RegDst | 选择写回寄存器是 `rt` 还是 `rd` |
| RegWrite | 是否写寄存器 |
| ALUSrc | ALU 第二个输入来自寄存器还是立即数 |
| ALUOp | 告诉 ALU 做什么类型操作 |
| MemRead | 是否读 Data Memory |
| MemWrite | 是否写 Data Memory |
| MemtoReg | 写回寄存器的数据来自 ALU 还是 Memory |
| Branch | 是否是 branch 指令 |

## 4. 控制信号表

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.57-p.60；`Computer Organization Chapter04 for 24H.pdf` p.62-p.65。

| 指令 | RegDst | ALUSrc | MemtoReg | RegWrite | MemRead | MemWrite | Branch | ALUOp |
|---|---|---|---|---|---|---|---|---|
| R-type | 1 | 0 | 0 | 1 | 0 | 0 | 0 | 10 |
| `lw` | 0 | 1 | 1 | 1 | 1 | 0 | 0 | 00 |
| `sw` | X | 1 | X | 0 | 0 | 1 | 0 | 00 |
| `beq` | X | 0 | X | 0 | 0 | 0 | 1 | 01 |
| `addi` | 0 | 1 | 0 | 1 | 0 | 0 | 0 | 00 |

X 表示 don't care。

## 5. 各类指令的数据流

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.37-p.51、p.55-p.60；`Computer Organization Chapter04 for 24H.pdf` p.62-p.65。

### 5.1 R-type

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.37-p.39；`Computer Organization Chapter04 for 24H.pdf` p.62。

例：

```text
add rd, rs, rt
```

路径：

```text
PC -> Instruction Memory
-> Register File 读 rs 和 rt
-> ALU 做 add
-> 结果写回 rd
```

不会访问 Data Memory。

### 5.2 `lw`

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.40-p.46；`Computer Organization Chapter04 for 24H.pdf` p.63。

例：

```text
lw rt, offset(rs)
```

路径：

```text
PC -> Instruction Memory
-> Register File 读 rs
-> Sign Extend offset
-> ALU 计算 Reg[rs] + offset
-> Data Memory 读数据
-> 写回 rt
```

### 5.3 `sw`

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.40-p.46；`Computer Organization Chapter04 for 24H.pdf` p.64。

例：

```text
sw rt, offset(rs)
```

路径：

```text
PC -> Instruction Memory
-> Register File 读 rs 和 rt
-> Sign Extend offset
-> ALU 计算 Reg[rs] + offset
-> Data Memory[地址] = Reg[rt]
```

不会写回寄存器。

### 5.4 `beq`

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.47-p.51、p.60；`Computer Organization Chapter04 for 24H.pdf` p.65。

例：

```text
beq rs, rt, label
```

路径：

```text
PC -> Instruction Memory
-> Register File 读 rs 和 rt
-> ALU 做 Reg[rs] - Reg[rt]
-> 如果 zero=1 且 Branch=1，则 PC = branch target
```

branch target：

```text
PC + 4 + (sign_extend(offset) << 2)
```

## 6. Homework4：LWI 新指令

PPT 对照：LWI 是 Homework4 扩展题；可对照单周期完整 datapath：`Computer Organization 04 Part1 for 24H.pdf` p.55-p.60。

题目：

```text
LWI Rt, Rd(Rs)
Reg[Rt] = Mem[Reg[Rd] + Reg[Rs]]
```

含义：

地址不是 `Reg[Rs] + immediate`，而是两个寄存器相加：

```text
Address = Reg[Rd] + Reg[Rs]
```

然后从 Data Memory 读出数据写回 `Rt`。

## 7. LWI 可以复用的模块

PPT 对照：`Computer Organization 04 Part1 for 24H.pdf` p.55-p.60；R/lw/sw/beq 原路径见 p.37-p.51。

可以复用：

- PC。
- Instruction Memory。
- Register File。
- ALU。
- Data Memory。
- MemtoReg mux。
- RegWrite 写回路径。

ALU 可以用来做：

```text
Reg[Rd] + Reg[Rs]
```

Data Memory 可以用来读：

```text
Mem[Address]
```

Register File 可以写回：

```text
Reg[Rt]
```

## 8. LWI 需要新增或修改的地方

PPT 对照：原 Register File 读端口和 mux 位置见 `Computer Organization 04 Part1 for 24H.pdf` p.35-p.39、p.57-p.60。

原 datapath 的问题：

标准 MIPS 的 Register File 通常读 `rs` 和 `rt`。但是 LWI 需要读 `rs` 和 `rd`，同时写回 `rt`。

所以需要：

1. 在 Register File 第二个读地址前增加一个 mux，让第二读端口可以选择 `rt` 或 `rd`。
2. 增加新的控制信号，例如 `ReadReg2Sel`，用于选择第二读寄存器字段。
3. ALUSrc 应该选择寄存器输入，而不是 immediate。
4. 写回目的寄存器选择 `rt`。

LWI 控制思路：

```text
RegWrite = 1
MemRead = 1
MemWrite = 0
MemtoReg = 1
ALUSrc = 0
Write register = rt
Second read register = rd
ALU operation = add
```

## 9. 三种处理器实现对比

PPT 对照：`Computer Organization Chapter04 for 24H.pdf` p.31、p.61-p.72、p.98-p.100、p.109、p.162。

| 实现方式 | 优点 | 缺点 |
|---|---|---|
| Single-cycle | 简单，每条指令一个周期 | cycle time 被最慢指令决定 |
| Multi-cycle | 每类指令用不同周期数，资源可复用 | 控制更复杂 |
| Pipeline | 提高吞吐率 | 需要处理 hazard，控制更复杂 |

重点：

- Single-cycle 改善不了单条指令延迟。
- Pipeline 主要提高 throughput，不一定降低单条指令 latency。

## 10. Day 5 自测

闭卷写出：

1. R-type、lw、sw、beq 的数据流。
2. 控制信号表。
3. LWI 可复用哪些模块。
4. LWI 为什么需要新增 mux。
5. single-cycle、multi-cycle、pipeline 优缺点。
