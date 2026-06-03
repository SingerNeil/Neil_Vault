# Day 2：数据表示与 IEEE 754

对应资料：

- 学校 PPT：`Computer Organization 03 for 24H.pdf`
- 作业：`Homework3 for Chapter03 for 24H.pdf`
- 视频：p10-p20、p25-p26

页码说明：下面的 `PPT p.X` 指 PDF 文件打开后的实际页序。

## 1. 进制转换

PPT 对照：`Computer Organization 03 for 24H.pdf` p.12-p.13、p.44；十六进制补充见 `Computer Organization 02 for 24H.pdf` p.78、p.81。

### 1.1 二进制转十进制

按权展开：

```text
101101_2 = 1×2^5 + 0×2^4 + 1×2^3 + 1×2^2 + 0×2^1 + 1×2^0
         = 32 + 8 + 4 + 1
         = 45
```

### 1.2 十进制整数转二进制

不断除以 2，倒序写余数。

### 1.3 十六进制和二进制

每 1 位十六进制 = 4 位二进制。

```text
0x5ED4 = 0101 1110 1101 0100
0x07A4 = 0000 0111 1010 0100
```

## 2. 无符号整数

PPT 对照：`Computer Organization 03 for 24H.pdf` p.13。

n 位无符号数范围：

```text
0 到 2^n - 1
```

8 位：

```text
0 到 255
```

16 位：

```text
0 到 65535
```

## 3. 有符号整数表示

PPT 对照：`Computer Organization 03 for 24H.pdf` p.15、p.17-p.23、p.32。

### 3.1 Sign-magnitude 原码

最高位是符号位：

- 0 表示正。
- 1 表示负。
- 剩余位表示数值大小。

缺点：有 +0 和 -0，运算不方便。

### 3.2 One's complement 反码

正数和原码相同。

负数：正数按位取反。

缺点：仍然有 +0 和 -0。

### 3.3 Two's complement 补码

正数和原码相同。

负数求法：

```text
先写正数二进制 -> 按位取反 -> 加 1
```

等价公式：

```text
n 位补码中，-x = 2^n - x
```

n 位补码范围：

```text
-2^(n-1) 到 2^(n-1)-1
```

8 位：

```text
-128 到 +127
```

16 位：

```text
-32768 到 +32767
```

## 4. Sign Extension 符号扩展

PPT 对照：`Computer Organization 03 for 24H.pdf` p.32、p.37。

规则：复制最高符号位。

例：8 位补码扩展到 16 位。

```text
0110 1010 -> 0000 0000 0110 1010
1001 0110 -> 1111 1111 1001 0110
```

如果最高位是 0，前面补 0。

如果最高位是 1，前面补 1。

## 5. 补码加减法

PPT 对照：`Computer Organization 03 for 24H.pdf` p.77-p.83。

减法改成加负数：

```text
A - B = A + (-B)
```

在固定 n 位中，超出的进位丢掉。

### 5.1 溢出判断

只对有符号数有 overflow。

两个正数相加得到负数：overflow。

两个负数相加得到正数：overflow。

一正一负相加：不会 overflow。

口诀：

```text
同号相加才可能溢出，结果变号就是溢出。
```

### 5.2 十六进制补码题步骤

题目：16 位 `0x5ED4 - 0x07A4`

如果当 unsigned：

```text
0x5ED4 - 0x07A4 = 0x5730
```

如果当 signed，要先判断最高位：

- `0x5ED4` 最高位是 0，是正数。
- `0x07A4` 最高位是 0，是正数。
- 结果 `0x5730` 最高位仍是 0，是正数。

所以 signed 情况下结果也是正数 `0x5730`。

## 6. IEEE 754 Single Precision

PPT 对照：`Computer Organization 03 for 24H.pdf` p.45-p.56、p.64-p.65。

单精度 32 位：

```text
sign:     1 bit
exponent: 8 bits
fraction: 23 bits
bias:     127
```

格式：

```text
(-1)^S × (1.Fraction) × 2^(Exponent - Bias)
```

这是规格化数，要求 exponent 不是全 0，也不是全 1。

## 7. 十进制转 IEEE 754

PPT 对照：`Computer Organization 03 for 24H.pdf` p.67-p.70。

步骤：

1. 确定符号位 S。
2. 把绝对值转成二进制。
3. 写成规格化形式 `1.xxx × 2^E`。
4. exponent field = `E + 127`。
5. fraction field = 小数点后面的部分，不够 23 位补 0。
6. 拼成 32 位，再转十六进制。

例：`-0.75`

```text
0.75 = 0.11_2
0.11_2 = 1.1_2 × 2^-1
S = 1
E = -1
Exponent = -1 + 127 = 126 = 01111110_2
Fraction = 10000000000000000000000
```

所以：

```text
1 01111110 10000000000000000000000
= 0xBF400000
```

## 8. IEEE 754 转十进制

PPT 对照：`Computer Organization 03 for 24H.pdf` p.68-p.70、p.92。

步骤：

1. 写成 32 位二进制。
2. 拆成 S、Exponent、Fraction。
3. E = Exponent - 127。
4. 有效数 = 1.Fraction。
5. 计算 `(-1)^S × 有效数 × 2^E`。

例：`0x0C000000`

二进制：

```text
00001100 00000000 00000000 00000000
```

拆分：

```text
S = 0
Exponent = 00011000_2 = 24
Fraction = 0
E = 24 - 127 = -103
```

所以作为 IEEE 754 浮点数：

```text
1.0 × 2^-103
```

同一个 bit pattern 如果作为 two's complement integer：

```text
0x0C000000 = 201,326,592
```

因为最高位是 0，所以是正整数。

## 9. Day 2 自测

闭卷写出：

1. n 位补码范围。
2. 负数补码求法。
3. 符号扩展规则。
4. 补码溢出判断规则。
5. IEEE 754 single precision 的字段位数和 bias。
6. `-0.75` 的 IEEE 754 表示。
