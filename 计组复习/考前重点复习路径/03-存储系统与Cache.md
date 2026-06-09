# 03-存储系统与 Cache

优先级：最高。Cache 地址划分和映射方式是后两天最值得投入的内容。

## 对应视频

### 主存和层次结构

- `p44 4-1 存储器概述`
- `p46 4-1-2 存储器概述 —存储器性能指标和存储系统层次结构`
- `p49 4-1-5 存储器概述 — 主存中数据的存放`

### SRAM / DRAM 可选补充

- `p50 4-2-1 静态随机存取存储器SRAM — 存储元`
- `p51 4-2-2 静态随机存取存储器SRAM — 存储元扩展和存储阵列扩展`
- `p52 4-2-3 静态随机存取存储器SRAM — 存储器结构及其芯片实例`
- `p53 4-3-1 动态随机存取存储器DRAM — 存储元及其扩展`
- `p54 4-3-2 动态随机存取存储器DRAM — 存储器的动态刷新`
- `p55 4-3-3 动态随机存取存储器DRAM — 存储器芯片实例和DRAM发展`

### Cache 必看

- `p65 4-7-1 高速缓冲存储器cache—cache的相关基本概念`
- `p66 4-7-2 高速缓冲存储器cache—cache的读、写流程`
- `p67 4-7-3 高速缓冲存储器cache — 地址映射 — （1）直接映射`
- `p68 4-7-3 高速缓冲存储器cache — 地址映射 — （2）直接映射 （习题课）`
- `p69 4-7-3 高速缓冲存储器cache — 地址映射 — （3）全相联映射`
- `p70 4-7-3 高速缓冲存储器cache — 地址映射 — （4）组相联映射`
- `p71 4-7-3 高速缓冲存储器cache — 地址映射 — （5）习题课`
- `p72 4-7-4 高速缓冲存储器cache — 替换算法（1）`
- `p73 4-7-4 高速缓冲存储器cache — 替换算法（2）习题课1`
- `p74 4-7-4 高速缓冲存储器cache — 替换算法（3）习题课2`
- `p75 4-7-5 高速缓冲存储器cache — 写入策略`
- `p76 4-7-6 高速缓冲存储器cache — cache的分类和应用`

## 对应 PPT 页码

- `Computer Organization 05 for 24H.pdf` p.5-p.9、p.37-p.38、p.90：memory hierarchy。
- `Computer Organization 05 for 24H.pdf` p.13-p.18：locality、hit / miss。
- `Computer Organization 05 for 24H.pdf` p.20-p.36：SRAM / DRAM / memory technology。
- `Computer Organization 05 for 24H.pdf` p.39-p.41：cache 基本问题。
- `Computer Organization 05 for 24H.pdf` p.42-p.55：direct mapped cache。
- `Computer Organization 05 for 24H.pdf` p.44-p.55：valid bit、tag、例题。
- `Computer Organization 05 for 24H.pdf` p.56-p.57：write-back / write-through。
- `Computer Organization 05 for 24H.pdf` p.59-p.68：fully associative / set associative。
- `Computer Organization 05 for 24H.pdf` p.72-p.76：tag / index / offset 计算例题。
- `Computer Organization 05 for 24H.pdf` p.78-p.82：placement、identification、replacement、write strategy。
- `Computer Organization 05 for 24H.pdf` p.83：cache 对 CPI 的影响。
- `Homework5 of Chapter05 for 24H.pdf` p.1：典型地址划分题。
- `Lab4 Cache Performance for 24H.pdf` p.1-p.5：Lab4 参数分析。

## 做题重点

### 地址划分通用模板

```text
address bits = log2(main memory size in bytes)
offset bits = log2(block size)
cache blocks = cache size / block size
```

Direct mapped：

```text
index bits = log2(cache blocks)
tag bits = address bits - index bits - offset bits
format = Tag | Index | Offset
```

Fully associative：

```text
index bits = 0
tag bits = address bits - offset bits
format = Tag | Offset
```

N-way set associative：

```text
sets = cache blocks / N
set index bits = log2(sets)
tag bits = address bits - set index bits - offset bits
format = Tag | Set index | Offset
```

### Homework5 例题必须会

题目条件：

```text
Main memory = 64KB = 2^16 bytes
Cache = 256B
Block size = 8B = 2^3 bytes
cache blocks = 256 / 8 = 32 = 2^5
```

答案：

```text
Direct mapped: Tag(8) | Index(5) | Offset(3)
Fully associative: Tag(13) | Offset(3)
2-way set associative: Tag(9) | Set index(4) | Offset(3)
```

### Cache 策略简答

Four questions：

```text
Where can a block be placed? placement
How is a block found? tag / index / valid bit
Which block is replaced? replacement policy
What happens on a write? write policy
```

Write-through：

```text
write cache and memory together; simple but slower
```

Write-back：

```text
write cache first; write memory when replaced; needs dirty bit
```

### Lab4 结论

Cache capacity 增大：

```text
miss rate 通常下降，capacity miss 和 conflict miss 减少。
```

Associativity 增大：

```text
conflict miss 减少，但收益会递减。
```

Block size 增大：

```text
利用 spatial locality，但过大会减少 cache lines 数量，可能增加冲突。
```

## 练习目标

- 重做 Homework5 第 2 题。
- 能闭卷写出 direct / fully / 2-way 的地址格式。
- 能用中文解释 Lab4 三个趋势：capacity、associativity、block size。

