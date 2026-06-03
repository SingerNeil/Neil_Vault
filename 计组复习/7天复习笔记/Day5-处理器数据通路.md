# Day 5：处理器数据通路

对应资料：

- 学校 PPT：`Computer Organization 04 Part1 for 24H.pdf`
- 学校 PPT：`Computer Organization Chapter04 for 24H.pdf`
- 作业：`Homework4 of Chapter04 for 24H.pdf`
- 视频：p98-p105、p106

## 1. 单周期 MIPS 的核心部件

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

| 指令 | RegDst | ALUSrc | MemtoReg | RegWrite | MemRead | MemWrite | Branch | ALUOp |
|---|---|---|---|---|---|---|---|---|
| R-type | 1 | 0 | 0 | 1 | 0 | 0 | 0 | 10 |
| `lw` | 0 | 1 | 1 | 1 | 1 | 0 | 0 | 00 |
| `sw` | X | 1 | X | 0 | 0 | 1 | 0 | 00 |
| `beq` | X | 0 | X | 0 | 0 | 0 | 1 | 01 |
| `addi` | 0 | 1 | 0 | 1 | 0 | 0 | 0 | 00 |

X 表示 don't care。

## 5. 各类指令的数据流

### 5.1 R-type

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

