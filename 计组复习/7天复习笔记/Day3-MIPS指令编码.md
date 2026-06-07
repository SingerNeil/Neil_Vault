# Day 3：MIPS 指令编码

对应资料：

- 学校 PPT：`Computer Organization 02 for 24H.pdf`
- 作业：`Homework2 of Chapter02 for 24H.pdf`
- 视频：
  - `4-1-5 存储器概述 — 主存中数据的存放`：只对应 endian / memory layout。
  - MIPS 机器码编码没有强匹配视频，必须以学校 PPT 和 Homework2 为主。
  - `5-1 指令系统概述`、`5-5 复杂指令集计算机和精简指令集计算机` 只作为 ISA / RISC-CISC 概念补充。

页码说明：下面的 `PPT p.X` 指 PDF 文件打开后的实际页序。

## 视频匹配度修正

我重新对照了本地视频 p84-p97 和学校 PPT02 / Homework2：

- `5-2-1`、`5-2-2`、`5-2-3` 讲的是通用指令字长、地址码字段、操作码字段，不讲 MIPS 的 R/I/J 格式，也不讲 `op/rs/rt/rd/shamt/funct` 的具体编码。
- `5-3-1` 到 `5-3-4` 讲的是通用寻址方式，不适合作为 MIPS 机器码题的主资料。
- `5-5` 只讲 CISC/RISC 对比，和 Homework2 只有概念层面的弱关联。
- `4-1-5 存储器概述 — 主存中数据的存放` 能补 Homework2 第 1 题的 big-endian / little-endian。

所以 Day 3 的正确用法是：

1. endian 题：看 `4-1-5`，再做 Homework2 第 1 题。
2. MIPS 编码题：直接看 `Computer Organization 02 for 24H.pdf` p.61、p.75-p.78、p.132-p.135，再做 Homework2 第 2 题。
3. 不要用 `5-2-1`、`5-2-2`、`5-2-3` 替代学校 PPT 的 MIPS 编码部分。

## PPT 页码快速索引

| 知识点 | 对应 PPT 页码 |
|---|---|
| ISA、architecture / organization、抽象接口 | `Computer Organization 02 for 24H.pdf` p.3-p.14 |
| MIPS / RISC / instruction overview | `Computer Organization 02 for 24H.pdf` p.15-p.25 |
| MIPS arithmetic、register operands、寄存器编号 | `Computer Organization 02 for 24H.pdf` p.26-p.36 |
| memory operands、byte address、alignment、endian | `Computer Organization 02 for 24H.pdf` p.43-p.59 |
| R-format 字段、R-format 例题 | `Computer Organization 02 for 24H.pdf` p.60-p.62、p.75-p.78 |
| I-format、load/store、immediate | `Computer Organization 02 for 24H.pdf` p.63-p.74、p.82 |
| branch / jump 基础 | `Computer Organization 02 for 24H.pdf` p.86-p.100 |
| MIPS 指令表、op/funct、logical operations | `Computer Organization 02 for 24H.pdf` p.103-p.117、p.121-p.122 |
| addressing modes 总结、R/I/J 总览 | `Computer Organization 02 for 24H.pdf` p.127-p.142 |
| endian 复习图 | `Computer Organization 02 for 24H.pdf` p.49-p.53、p.85、p.101 |

## 1. ISA 是什么

PPT 对照：`Computer Organization 02 for 24H.pdf` p.12、p.16-p.17；抽象接口补充见 p.4。

ISA 是 Instruction Set Architecture。

它定义软件能看到的硬件接口，包括：

- 有哪些指令。
- 有哪些寄存器。
- 指令格式是什么。
- 内存如何寻址。
- 数据类型和寻址方式。

一句话：ISA 是软件和硬件之间的接口。

## 2. RISC 和 CISC

PPT 对照：`Computer Organization 02 for 24H.pdf` p.13-p.14、p.25、p.141-p.142。

RISC：

- 指令少。
- 格式规整。
- 多数指令长度固定。
- load/store 架构。
- MIPS 是典型 RISC。

CISC：

- 指令复杂。
- 指令长度可能不固定。
- 一条指令可能做很多事。
- x86 是典型 CISC。

## 3. MIPS 常用寄存器

PPT 对照：`Computer Organization 02 for 24H.pdf` p.31-p.32。

| 名称 | 编号 | 用途 |
|---|---:|---|
| `$zero` | 0 | 永远为 0 |
| `$at` | 1 | assembler temporary |
| `$v0-$v1` | 2-3 | 返回值 |
| `$a0-$a3` | 4-7 | 函数参数 |
| `$t0-$t7` | 8-15 | 临时寄存器 |
| `$s0-$s7` | 16-23 | saved registers |
| `$t8-$t9` | 24-25 | 临时寄存器 |
| `$gp` | 28 | global pointer |
| `$sp` | 29 | stack pointer |
| `$fp` | 30 | frame pointer |
| `$ra` | 31 | return address |

必须熟悉：

```text
$t0 = 8
$t1 = 9
$t2 = 10
$s0 = 16
$s1 = 17
$s2 = 18
$s3 = 19
```

## 4. MIPS 三种指令格式

PPT 对照：`Computer Organization 02 for 24H.pdf` p.61-p.64、p.71、p.76-p.82、p.132-p.135。

### 4.1 R-format

PPT 对照：`Computer Organization 02 for 24H.pdf` p.61-p.62、p.76-p.77、p.132。

```text
op     rs     rt     rd     shamt  funct
6 bits 5 bits 5 bits 5 bits 5 bits 6 bits
```

常见写法：

```text
add rd, rs, rt
sub rd, rs, rt
and rd, rs, rt
or  rd, rs, rt
slt rd, rs, rt
```

R-type 的 `op` 通常是 0，真正操作由 `funct` 决定。

### 4.2 I-format

PPT 对照：`Computer Organization 02 for 24H.pdf` p.63-p.64、p.82、p.133-p.134。

```text
op     rs     rt     immediate
6 bits 5 bits 5 bits 16 bits
```

常见写法：

```text
addi rt, rs, imm
lw   rt, offset(rs)
sw   rt, offset(rs)
beq  rs, rt, offset
bne  rs, rt, offset
lui  rt, imm
ori  rt, rs, imm
```

### 4.3 J-format

PPT 对照：`Computer Organization 02 for 24H.pdf` p.93-p.95、p.135。

```text
op     address
6 bits 26 bits
```

常见写法：

```text
j target
jal target
```

## 5. 常用 op / funct

PPT 对照：`Computer Organization 02 for 24H.pdf` p.61、p.76、p.100、p.103-p.107、p.115-p.117、p.122。

| 指令 | 类型 | op | funct |
|---|---|---:|---:|
| `add` | R | 0 | 32 |
| `addu` | R | 0 | 33 |
| `sub` | R | 0 | 34 |
| `and` | R | 0 | 36 |
| `or` | R | 0 | 37 |
| `slt` | R | 0 | 42 |
| `sll` | R | 0 | 0 |
| `addi` | I | 8 | - |
| `lw` | I | 35 | - |
| `sw` | I | 43 | - |
| `beq` | I | 4 | - |
| `bne` | I | 5 | - |
| `lui` | I | 15 | - |
| `ori` | I | 13 | - |
| `j` | J | 2 | - |
| `jal` | J | 3 | - |

十六进制常用：

```text
lw = 0x23
sw = 0x2B
beq = 0x04
bne = 0x05
addi = 0x08
lui = 0x0F
ori = 0x0D
```

## 6. 汇编转机器码步骤

PPT 对照：`Computer Organization 02 for 24H.pdf` p.75-p.78、p.82、p.91、p.96、p.132-p.135。

1. 判断 R/I/J 类型。
2. 写出字段顺序。
3. 查寄存器编号。
4. 查 op/funct。
5. immediate 转 16 位二进制。
6. 拼成 32 位。
7. 每 4 位转 1 位十六进制。

### 例 1：`sw $t1, 32($t2)`

I-format：

```text
op = sw = 43 = 101011
rs = $t2 = 10 = 01010
rt = $t1 = 9  = 01001
imm = 32 = 0000000000100000
```

机器码：

```text
101011 01010 01001 0000000000100000
= 0xAD490020
```

### 例 2：`sub $v1, $v1, $v0`

R-format：

```text
op = 0
rs = $v1 = 3
rt = $v0 = 2
rd = $v1 = 3
shamt = 0
funct = sub = 34
```

机器码：

```text
000000 00011 00010 00011 00000 100010
= 0x00621822
```

## 7. 机器码转汇编步骤

PPT 对照：`Computer Organization 02 for 24H.pdf` p.75-p.78、p.132-p.135。

1. 看前 6 位 op。
2. 如果 op = 0，按 R-format 解析 funct。
3. 如果 op 不是 0，按 I-format 或 J-format 解析。
4. 把寄存器编号翻译成寄存器名。
5. 按指令格式写汇编。

例：

```text
0000 0010 0001 0000 1000 0000 0010 0000
```

整理：

```text
000000 10000 10000 10000 00000 100000
```

字段：

```text
op = 0
rs = 16 = $s0
rt = 16 = $s0
rd = 16 = $s0
shamt = 0
funct = 32 = add
```

所以：

```text
add $s0, $s0, $s0
```

## 8. Endian

PPT 对照：`Computer Organization 02 for 24H.pdf` p.49-p.53、p.85、p.101。

假设从地址 0 开始存 `0xABCDEF12`。

Big-endian：最高有效字节放低地址。

```text
Address 0: AB
Address 1: CD
Address 2: EF
Address 3: 12
```

Little-endian：最低有效字节放低地址。

```text
Address 0: 12
Address 1: EF
Address 2: CD
Address 3: AB
```

## 9. Day 3 自测

闭卷写出：

1. R/I/J 三种格式。
2. `$t0-$t2`、`$s0-$s3` 的编号。
3. `lw`、`sw`、`beq`、`bne`、`j` 的 op。
4. `add`、`sub` 的 funct。
5. `sw $t1, 32($t2)` 的机器码。
