# Day 4：MIPS 分支跳转与 Lab3

对应资料：

- 学校 PPT：`Computer Organization 02 for 24H.pdf`
- Lab：`Lab3 Task for 24H.pdf`
- 已完成作业：`22H034160227-徐楚明-Lab3-ISA.docx`
- 视频：p88-p92

## 1. 三类地址计算

MIPS 里最容易丢分的是：

1. `lw/sw` 的有效地址。
2. `beq/bne` 的 branch target。
3. `j` 的 jump target。

## 2. `lw/sw` 有效地址

格式：

```text
lw rt, offset(rs)
sw rt, offset(rs)
```

地址计算：

```text
Effective Address = Reg[rs] + sign_extend(offset)
```

例：

```text
lw $t0, 32($s3)
```

含义：

```text
$t0 = Memory[$s3 + 32]
```

```text
sw $t0, 8($s3)
```

含义：

```text
Memory[$s3 + 8] = $t0
```

## 3. `beq/bne` 分支地址

格式：

```text
beq rs, rt, label
bne rs, rt, label
```

机器码里保存的不是完整目标地址，而是 16 位 offset。

核心公式：

```text
Branch Target = PC + 4 + (sign_extend(immediate) << 2)
```

反过来求 immediate：

```text
immediate = (Target Address - (PC + 4)) / 4
```

为什么用 `PC + 4`：

因为当前指令取出后，PC 通常已经更新到下一条指令。

## 4. Branch 例题

题目：

```text
PC = 0x00400050
bne $s3, $s4, Else
immediate = 0x0003
```

计算：

```text
PC + 4 = 0x00400054
immediate << 2 = 0x0003 × 4 = 0x000C
Target = 0x00400054 + 0x000C = 0x00400060
```

所以：

```text
Else = 0x00400060
```

## 5. `j` 跳转地址

J-format：

```text
op: 6 bits
address: 26 bits
```

真实 target：

```text
Jump Target = { PC+4 的高 4 位, address field, 00 }
```

也就是：

```text
Jump Target = (PC+4 upper 4 bits) | (address field << 2)
```

例：

```text
j Exit
address field = 0x100018
```

则：

```text
0x100018 << 2 = 0x00400060
```

如果 `PC+4` 高 4 位也是 `0x0`，那么：

```text
Exit = 0x00400060
```

## 6. `lui + ori`

`lui`：

```text
lui rt, imm
```

作用：

```text
Reg[rt] = imm << 16
```

例：

```text
lui $s1, 0x1234
```

结果：

```text
$s1 = 0x12340000
```

`ori`：

```text
ori rt, rs, imm
```

作用：

```text
Reg[rt] = Reg[rs] | zero_extend(imm)
```

例：

```text
ori $s1, $s1, 0x5678
```

结果：

```text
$s1 = 0x12345678
```

## 7. `jr $ra`

`jr` 是 R-type。

```text
jr $ra
```

含义：

```text
PC = Reg[$ra]
```

常用于函数返回。

## 8. Lab3 做题方法

Lab3 常见空格：

1. 某条指令执行前 PC 是多少。
2. `($PC)` 也就是 PC 地址处的机器码是什么。
3. 执行后某个寄存器是多少。
4. label 地址是多少。
5. branch/jump 的机器码字段是多少。

做题顺序：

1. 先列出 `.data` 中每个变量的地址和值。
2. 再列出 `.text` 中每条真实指令的地址。
3. 注意 pseudo-instruction，比如 `la` 可能会被汇编器展开。
4. 每执行一条指令，更新寄存器表。
5. 遇到 branch/jump，单独算 target。

## 9. Lab3 核心提醒

### 9.1 `la`

`la` 是 pseudo-instruction，不一定是一条真实机器指令。

QtSPIM 里可能展开成：

```text
lui
ori
```

所以算 PC 时不要想当然，要以模拟器里真实展开后的 instruction address 为准。

### 9.2 `bne $s3, $s4, Else`

如果：

```text
$s3 != $s4
```

跳到 Else。

如果：

```text
$s3 == $s4
```

继续执行下一条。

### 9.3 `sw $s0, 16($t0)`

含义：

```text
Memory[$t0 + 16] = $s0
```

如果 `$t0` 指向 `g`，并且 `f` 在 `g` 后面 16 bytes，那就是把 `$s0` 存到 `f`。

## 10. Day 4 自测

闭卷写出：

1. branch target 公式。
2. branch immediate 反推公式。
3. jump target 公式。
4. `lui $s1,0x1234` 后 `$s1` 是多少。
5. `ori $s1,$s1,0x5678` 后 `$s1` 是多少。

