# Day 7：Cache 与总复盘

对应资料：

- 学校 PPT：`Computer Organization 05 for 24H.pdf`
- 作业：`Homework5 of Chapter05 for 24H.pdf`
- Lab：`Lab4 Cache Performance for 24H.pdf`
- 视频：
  - 可选补 PPT05 前半：`4-1 存储器概述`
  - 可选补 PPT05 前半：`4-1-2 存储器概述 —存储器性能指标和存储系统层次结构`
  - 可选补 PPT05 前半：`4-2-1 静态随机存取存储器SRAM — 存储元`
  - 可选补 PPT05 前半：`4-2-2 静态随机存取存储器SRAM — 存储元扩展和存储阵列扩展`
  - 可选补 PPT05 前半：`4-2-3 静态随机存取存储器SRAM — 存储器结构及其芯片实例`
  - 可选补 PPT05 前半：`4-3-1 动态随机存取存储器DRAM — 存储元及其扩展`
  - 可选补 PPT05 前半：`4-3-2 动态随机存取存储器DRAM — 存储器的动态刷新`
  - 可选补 PPT05 前半：`4-3-3 动态随机存取存储器DRAM — 存储器芯片实例和DRAM发展`
  - `4-7-1 高速缓冲存储器cache—cache的相关基本概念`
  - `4-7-2 高速缓冲存储器cache—cache的读、写流程`
  - `4-7-3 高速缓冲存储器cache — 地址映射 — （1）直接映射`
  - `4-7-3 高速缓冲存储器cache — 地址映射 — （2）直接映射 （习题课）`
  - `4-7-3 高速缓冲存储器cache — 地址映射 — （3）全相联映射`
  - `4-7-3 高速缓冲存储器cache — 地址映射 — （4）组相联映射`
  - `4-7-3 高速缓冲存储器cache — 地址映射 — （5）习题课`
  - `4-7-4 高速缓冲存储器cache — 替换算法（1）`
  - `4-7-4 高速缓冲存储器cache — 替换算法（2）习题课1`
  - `4-7-4 高速缓冲存储器cache — 替换算法（3）习题课2`
  - `4-7-5 高速缓冲存储器cache — 写入策略`
  - `4-7-6 高速缓冲存储器cache — cache的分类和应用`

页码说明：下面的 `PPT p.X` 指 PDF 文件打开后的实际页序。

## 视频匹配度检查

我重新对照了本地视频 p44-p76、PPT05、Homework5 和 Lab4：Day 7 的 cache 视频是强匹配，但原清单漏了 PPT05 前半的存储器技术补充。

- p65-p76 直接覆盖 cache 基本概念、读写流程、直接映射、全相联、组相联、替换算法、写策略，和 Homework5 / Lab4 的核心题型匹配。
- PPT05 p.5-p.18 的 memory hierarchy / locality，可用 `4-1`、`4-1-2` 补理解。
- PPT05 p.20-p.36 的 SRAM / DRAM / memory technology，可用 `4-2-1` 到 `4-3-3` 补理解。
- 如果时间不够，Day7 必看仍然是 p65-p76；前半存储器技术视频只在你复习 PPT05 p.5-p.36 时补看。

## PPT 页码快速索引

| 知识点 | 对应 PPT 页码 |
|---|---|
| memory hierarchy 目标和结构 | `Computer Organization 05 for 24H.pdf` p.5-p.9、p.37-p.38、p.90 |
| locality、hit/miss terminology | `Computer Organization 05 for 24H.pdf` p.13-p.18 |
| memory technology：SRAM / DRAM | `Computer Organization 05 for 24H.pdf` p.20-p.36 |
| cache 基本问题和工作原理 | `Computer Organization 05 for 24H.pdf` p.39-p.41 |
| direct mapped cache | `Computer Organization 05 for 24H.pdf` p.42-p.55 |
| valid bit、tag、简单例题 | `Computer Organization 05 for 24H.pdf` p.44-p.55 |
| write-back / write-through 初步 | `Computer Organization 05 for 24H.pdf` p.56-p.57 |
| fully associative / set associative | `Computer Organization 05 for 24H.pdf` p.59-p.68 |
| tag/index/offset 计算例题 | `Computer Organization 05 for 24H.pdf` p.72-p.76 |
| block placement / identification / replacement / write strategy | `Computer Organization 05 for 24H.pdf` p.78-p.82 |
| cache 对 CPI 的影响 | `Computer Organization 05 for 24H.pdf` p.58、p.83 |
| Homework5 题面 | `Homework5 of Chapter05 for 24H.pdf` p.1 |
| Lab4 题面 | `Lab4 Cache Performance for 24H.pdf` p.1-p.5 |

## 1. Memory Hierarchy

PPT 对照：`Computer Organization 05 for 24H.pdf` p.5-p.9、p.13、p.16-p.20、p.37。

存储层次从快到慢：

```text
Registers -> L1 Cache -> L2 Cache -> Main Memory -> Disk
```

越靠近 CPU：

- 速度越快。
- 容量越小。
- 单位成本越高。

越远离 CPU：

- 速度越慢。
- 容量越大。
- 单位成本越低。

## 2. Locality

PPT 对照：`Computer Organization 05 for 24H.pdf` p.13-p.18。

Cache 能工作，靠 locality。

### 2.1 Temporal Locality

PPT 对照：`Computer Organization 05 for 24H.pdf` p.13-p.14、p.18。

时间局部性：

最近访问过的数据，很可能很快再次访问。

例：

循环变量、反复使用的数组元素。

### 2.2 Spatial Locality

PPT 对照：`Computer Organization 05 for 24H.pdf` p.13-p.14、p.18。

空间局部性：

访问某个地址后，很可能访问附近地址。

例：

顺序访问数组。

## 3. Cache 基本概念

PPT 对照：`Computer Organization 05 for 24H.pdf` p.17、p.39、p.44-p.56。

Block：

Cache 和 Memory 之间传输的基本单位。

Hit：

要访问的数据在 Cache 中。

Miss：

要访问的数据不在 Cache 中，需要去下一层取。

Hit rate：

```text
hit rate = hits / total accesses
```

Miss rate：

```text
miss rate = misses / total accesses
```

## 4. 地址划分通用步骤

PPT 对照：`Computer Organization 05 for 24H.pdf` p.42-p.55、p.59-p.68、p.78-p.82。

给定：

- Main memory size。
- Cache size。
- Block size。
- Associativity。

步骤：

1. 求 address bits。
2. 求 block offset bits。
3. 求 cache blocks 数量。
4. 根据映射方式求 index bits。
5. 求 tag bits。

## 5. Direct Mapped Cache

PPT 对照：`Computer Organization 05 for 24H.pdf` p.42、p.44-p.55、p.67-p.68、p.79。

每个 memory block 只能放到 cache 中唯一一个位置。

```text
cache index = memory block address mod number of cache blocks
```

地址格式：

```text
Tag | Index | Block Offset
```

计算：

```text
offset bits = log2(block size)
cache blocks = cache size / block size
index bits = log2(cache blocks)
tag bits = address bits - index bits - offset bits
```

## 6. Fully Associative Cache

PPT 对照：`Computer Organization 05 for 24H.pdf` p.59-p.61、p.67-p.68、p.79。

每个 memory block 可以放到 cache 任意位置。

没有 index。

地址格式：

```text
Tag | Block Offset
```

计算：

```text
offset bits = log2(block size)
tag bits = address bits - offset bits
```

## 7. Set Associative Cache

PPT 对照：`Computer Organization 05 for 24H.pdf` p.62-p.68、p.79-p.80。

n-way set associative：

每个 set 里有 n 个 blocks。

```text
sets = cache blocks / n
```

地址格式：

```text
Tag | Set Index | Block Offset
```

计算：

```text
offset bits = log2(block size)
cache blocks = cache size / block size
sets = cache blocks / ways
index bits = log2(sets)
tag bits = address bits - index bits - offset bits
```

## 8. Homework5 典型题

PPT 对照：地址 tag/index/offset 的划分见 `Computer Organization 05 for 24H.pdf` p.44-p.55、p.67-p.68、p.79；题面见 `Homework5 of Chapter05 for 24H.pdf` p.1。

题目：

```text
Main memory = 64 KB
Cache = 256 B
Block size = 8 B
```

假设按 byte address：

```text
Main memory = 64 KB = 2^16 B
address bits = 16
block size = 8 B = 2^3 B
offset bits = 3
cache blocks = 256 / 8 = 32 = 2^5
```

### 8.1 Direct mapped

```text
index bits = 5
tag bits = 16 - 5 - 3 = 8
```

格式：

```text
Tag 8 | Index 5 | Offset 3
```

### 8.2 Fully associative

```text
index bits = 0
tag bits = 16 - 3 = 13
```

格式：

```text
Tag 13 | Offset 3
```

### 8.3 2-way set associative

```text
cache blocks = 32
ways = 2
sets = 32 / 2 = 16 = 2^4
index bits = 4
tag bits = 16 - 4 - 3 = 9
```

格式：

```text
Tag 9 | Set Index 4 | Offset 3
```

## 9. Valid Bit 和 Dirty Bit

PPT 对照：Valid/tag 见 `Computer Organization 05 for 24H.pdf` p.44-p.47、p.55；Dirty/write-back 见 p.56、p.82。

Valid bit：

表示 cache block 中的数据是否有效。

Dirty bit：

用于 write-back，表示 cache 中的数据是否已经被修改但还没写回 memory。

## 10. Replacement Policy

PPT 对照：`Computer Organization 05 for 24H.pdf` p.78-p.80。

Direct mapped：

没有选择，只能替换唯一位置。

Set associative / fully associative：

需要 replacement policy。

常见：

- Random。
- LRU：Least Recently Used，替换最久没用的块。

## 11. Write Policy

PPT 对照：`Computer Organization 05 for 24H.pdf` p.56-p.57、p.81-p.82。

### 11.1 Write-through

PPT 对照：`Computer Organization 05 for 24H.pdf` p.57、p.81。

写 Cache 的同时写 Memory。

优点：

- Memory 和 Cache 一致。

缺点：

- 写操作慢，流量大。

### 11.2 Write-back

PPT 对照：`Computer Organization 05 for 24H.pdf` p.56、p.82。

只写 Cache，替换时再写回 Memory。

优点：

- 写操作快，减少 memory traffic。

缺点：

- 需要 dirty bit。
- 一致性更复杂。

## 12. Lab4 趋势结论

PPT 对照：Cache 映射和替换策略见 `Computer Organization 05 for 24H.pdf` p.59-p.68、p.78-p.82；Lab4 题面见 `Lab4 Cache Performance for 24H.pdf` p.1-p.5。

Cache capacity 增大：

```text
miss rate 通常下降
```

Associativity 增大：

```text
conflict miss 通常下降
```

Block size 增大：

```text
可能利用 spatial locality 降低 miss rate
但过大可能带来浪费，甚至增加 miss penalty
```

替换策略：

```text
LRU 通常比 Random 更符合 temporal locality
```

## 13. 最终速记表

PPT 对照：性能公式见 `Computer Organization 01 for 24H.pdf` p.99-p.111；MIPS 编码见 `Computer Organization 02 for 24H.pdf` p.61-p.82；Branch/Jump 见 p.86-p.100；IEEE 754 见 `Computer Organization 03 for 24H.pdf` p.45-p.70；Pipeline 见 `Computer Organization Chapter04 for 24H.pdf` p.101-p.161；Cache 见 `Computer Organization 05 for 24H.pdf` p.5-p.82。

### 性能

```text
CPU Time = IC × CPI × Cycle Time
CPU Time = IC × CPI / Clock Rate
Performance = 1 / Execution Time
```

### MIPS

```text
R: op rs rt rd shamt funct
I: op rs rt immediate
J: op address
```

### Branch

```text
Target = PC + 4 + (immediate << 2)
immediate = (Target - (PC + 4)) / 4
```

### IEEE 754

```text
single: sign 1, exponent 8, fraction 23, bias 127
value = (-1)^S × 1.F × 2^(E-127)
```

### Cache

```text
offset = log2(block size)
direct index = log2(cache blocks)
set index = log2(cache blocks / ways)
tag = address bits - index - offset
```

### Pipeline

```text
IF -> ID -> EX -> MEM -> WB
```

## 14. Day 7 自测

闭卷写出：

1. direct mapped 地址格式。
2. fully associative 地址格式。
3. 2-way set associative 地址格式。
4. Homework5 的 64KB / 256B / 8B 三种答案。
5. write-through 和 write-back 区别。
6. capacity、associativity、block size 对 miss rate 的影响。
