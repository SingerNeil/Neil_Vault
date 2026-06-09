# 计组考前重点复习路径

目标：从现在到 6月12日 18:00 考试前，不再平均复习整门课，只保最可能出题、最能拿分的题型。

页码说明：下面所有 `PPT p.X` 都是 PDF 打开后的实际页序。视频编号按你本地 B 站下载文件里的 `pXX + 标题`。

## 总优先级

### 第一优先级：必须先完成

1. 性能计算：CPU Time、CPI、clock rate、frame buffer。
2. 数据表示：原码、反码、补码、移码、IEEE754。
3. Cache：tag / index / offset、direct / fully / set associative、Lab4 结论。
4. 流水线：IF / ID / EX / MEM / WB、hazard、stall、forwarding。

### 第二优先级：保底掌握

1. ALU / 运算器：补码加减法、溢出、加法器、Lab1 功能表。
2. CPU 执行过程：fetch、decode、execute、memory、write back。
3. Datapath / LWI：只会 Homework4 那种扩展题即可。

### 第三优先级：只保留基础题型

1. MIPS 基础字段、endian、branch / jump 公式、Lab3 填表。
2. 不背完整寄存器编号表，不钻复杂机器码大题。
3. 总线、I/O、中断、DMA、复杂乘除法器、微程序控制器：时间不够先跳过。

## 文件使用顺序

1. `01-性能与数据表示.md`
2. `02-运算器与ALU.md`
3. `03-存储系统与Cache.md`
4. `04-CPU执行与流水线.md`
5. `05-低投入保底题型.md`
6. `06-考前最后检查清单.md`

## 推荐时间分配

### 6月10日

- 2-3 小时：性能计算 + 数据表示。
- 2-3 小时：Cache。
- 1.5 小时：流水线基础。

### 6月11日

- 2 小时：运算器 / ALU。
- 2 小时：CPU 执行过程 + Datapath / LWI。
- 2 小时：Homework1-5 按小抄重做一遍。
- 1 小时：Lab2 / Lab3 / Lab4 只看题型。

### 6月12日考试前

- 不看新视频。
- 只看 `考试做题小抄-计算机组成原理.md` 和本文件夹里的 `06-考前最后检查清单.md`。
- 最后 2 小时只背公式、字段、Cache 地址划分、流水线 hazard。

## 看视频原则

- 视频只服务于“能不能做题”，不要追求整章听懂。
- 一段视频看 10 分钟还没转化成做题步骤，就停，回到小抄和 PPT。
- 本课程 PPT 明确有 MIPS / Datapath / Pipeline，但 MIPS 机器码不是最高性价比，保底即可。

