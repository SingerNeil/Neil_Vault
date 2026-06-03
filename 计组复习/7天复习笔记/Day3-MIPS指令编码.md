# Day 3：MIPS 指令编码

对应资料：

- 学校 PPT：`Computer Organization 02 for 24H.pdf`
- 作业：`Homework2 of Chapter02 for 24H.pdf`
- 视频：p84-p87、p94

## 1. ISA 是什么

ISA 是 Instruction Set Architecture。

它定义软件能看到的硬件接口，包括：

- 有哪些指令。
- 有哪些寄存器。
- 指令格式是什么。
- 内存如何寻址。
- 数据类型和寻址方式。

一句话：ISA 是软件和硬件之间的接口。

## 2. RISC 和 CISC

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

### 4.1 R-format

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

