# Day 1：性能公式卡

对应资料：

- 学校 PPT：`Computer Organization 01 for 24H.pdf`
- 作业：`Homework1 of Chapter01 for 24H.pdf`
- 视频：
  - `1-7 计算机系统的性能指标—（1）基本性能指标`
  - `1-7 计算机系统的性能指标—（2）与运算速度相关的性能指标`

页码说明：下面的 `PPT p.X` 指 PDF 文件打开后的实际页序。

## PPT 页码快速索引

| 知识点 | 对应 PPT 页码 |
|---|---|
| 课程目标、性能影响因素 | `Computer Organization 01 for 24H.pdf` p.2-p.4、p.37-p.40 |
| Great Ideas | `Computer Organization 01 for 24H.pdf` p.41-p.43 |
| 程序从高级语言到机器码 | `Computer Organization 01 for 24H.pdf` p.47-p.52 |
| ISA、软硬件接口、系统层次 | `Computer Organization 01 for 24H.pdf` p.52-p.54、p.131-p.133 |
| object code 如何进入内存并被 CPU 执行 | `Computer Organization 01 for 24H.pdf` p.55-p.57 |
| LCD / bit map / frame buffer 题型 | `Computer Organization 01 for 24H.pdf` p.84-p.85 |
| execution time、relative performance | `Computer Organization 01 for 24H.pdf` p.99-p.102 |
| clock cycle、clock rate、CPU time | `Computer Organization 01 for 24H.pdf` p.101-p.110 |
| CPI、instruction count、编译器比较 | `Computer Organization 01 for 24H.pdf` p.109-p.117、p.119 |
| benchmark / SPEC | `Computer Organization 01 for 24H.pdf` p.121-p.129 |

## 1. 计算机系统主线

PPT 对照：`Computer Organization 01 for 24H.pdf` p.47、p.52-p.54、p.131-p.133。

一台计算机可以从高到低理解成：

1. Application software：应用程序。
2. System software：操作系统、编译器、汇编器、链接器、加载器。
3. Instruction Set Architecture：ISA，软件和硬件之间的接口。
4. Microarchitecture / Organization：数据通路、控制器、流水线、Cache。
5. Logic circuits：门电路、寄存器、ALU。
6. Devices：晶体管、存储器芯片、I/O 设备。

考试里遇到“抽象”相关问题，核心回答是：复杂系统靠分层抽象来隐藏下层细节，上层只依赖接口。

## 2. 高级语言程序如何被 CPU 执行

PPT 对照：`Computer Organization 01 for 24H.pdf` p.47、p.49-p.52。

标准流程：

```text
C program
-> compiler
-> assembly program
-> assembler
-> object file
-> linker, with libraries
-> executable file
-> loader
-> memory
-> processor executes machine instructions
```

答题句式：

高层语言程序先由编译器翻译成汇编语言，再由汇编器翻译成机器码目标文件。链接器把多个目标文件和库函数合并成可执行文件。加载器把可执行文件放入内存，处理器按照 ISA 逐条取指、译码、执行机器指令。

## 3. Great Ideas

PPT 对照：`Computer Organization 01 for 24H.pdf` p.41-p.43。

常见教材中的计算机体系结构思想：

1. Design for Moore's Law：设计要考虑芯片资源持续增长。
2. Use abstraction to simplify design：用抽象简化复杂系统。
3. Make the common case fast：优化最常发生的情况。
4. Performance via parallelism：用并行提升性能。
5. Performance via pipelining：用流水线提升吞吐率。
6. Performance via prediction：用预测减少等待。
7. Hierarchy of memories：用存储层次兼顾速度、容量、成本。
8. Dependability via redundancy：用冗余提高可靠性。

如果题目写 Seven Great Ideas，至少把前 7 个写清楚；如果老师按教材完整版本，也可以把第 8 个冗余一起写上。

## 4. 性能核心公式

PPT 对照：`Computer Organization 01 for 24H.pdf` p.99-p.111；公式总复习也见 `Computer Organization Chapter04 for 24H.pdf` p.160。

### 4.1 时间和性能

PPT 对照：`Computer Organization 01 for 24H.pdf` p.99-p.100。

```text
Performance = 1 / Execution Time
```

如果 A 比 B 快：

```text
Performance_A / Performance_B = ExecutionTime_B / ExecutionTime_A
```

例：

如果 A 用 10s，B 用 20s：

```text
A 的性能 / B 的性能 = 20 / 10 = 2
```

所以 A is 2 times faster than B。

### 4.2 CPU 执行时间

PPT 对照：`Computer Organization 01 for 24H.pdf` p.101-p.109。

```text
CPU Time = Instruction Count × CPI × Clock Cycle Time
```

因为：

```text
Clock Cycle Time = 1 / Clock Rate
```

所以：

```text
CPU Time = Instruction Count × CPI / Clock Rate
```

必须背熟：

- Instruction Count：程序执行的指令条数。
- CPI：Cycles Per Instruction，每条指令平均需要多少时钟周期。
- Clock Cycle Time：一个周期多长。
- Clock Rate：每秒多少周期。

### 4.3 平均 CPI

PPT 对照：`Computer Organization 01 for 24H.pdf` p.109-p.110。

如果有多类指令：

```text
Average CPI = total clock cycles / total instruction count
```

也可以写成：

```text
Average CPI = sum(IC_i × CPI_i) / sum(IC_i)
```

## 5. 常见题型模板

PPT 对照：`Computer Organization 01 for 24H.pdf` p.109-p.115、p.119。

### 5.1 求 CPU time

题目给 IC、CPI、clock rate：

```text
CPU Time = IC × CPI / Clock Rate
```

题目给 IC、CPI、cycle time：

```text
CPU Time = IC × CPI × Cycle Time
```

### 5.2 求 CPI

PPT 对照：`Computer Organization 01 for 24H.pdf` p.109-p.111。

题目给 execution time、clock cycle time、instruction count：

```text
CPI = Execution Time / (Instruction Count × Clock Cycle Time)
```

题目给 execution time、clock rate、instruction count：

```text
CPI = Execution Time × Clock Rate / Instruction Count
```

### 5.3 比较两个编译器

PPT 对照：`Computer Organization 01 for 24H.pdf` p.110-p.115、p.119。

写法：

1. 分别求 A、B 的 CPU time 或 CPI。
2. 用 `ExecutionTime_B / ExecutionTime_A` 比较性能。
3. 如果执行时间相同但 CPI 不同，用：

```text
ClockRate_A / ClockRate_B = (IC_A × CPI_A) / (IC_B × CPI_B)
```

如果同一个程序、同样指令数，可以简化成：

```text
ClockRate_A / ClockRate_B = CPI_A / CPI_B
```

## 6. Frame Buffer 题

PPT 对照：`Computer Organization 01 for 24H.pdf` p.84-p.85。

公式：

```text
Frame Buffer Size = width × height × bits per pixel / 8
```

RGB 每个颜色 8 bits：

```text
bits per pixel = 8 + 8 + 8 = 24 bits
```

例：1280 × 1024，RGB 每个颜色 8 bits：

```text
1280 × 1024 × 24 / 8
= 1280 × 1024 × 3
= 3,932,160 bytes
```

换算成 MiB：

```text
3,932,160 / 1024 / 1024 = 3.75 MiB
```

## 7. Day 1 自测

闭卷写出：

1. CPU execution time 公式。
2. Performance 和 execution time 的关系。
3. 平均 CPI 公式。
4. 高级语言程序到机器执行的完整流程。
5. Frame buffer 计算公式。

能全部写出来，再去做 Homework1。
