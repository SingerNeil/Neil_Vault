
---

**U-Curve自动对焦算法（U型曲线法）**，
![[autofocus_ucurve.pdf]]核心思想是通过测量不同焦点位置的图像清晰度（HFD）拟合出一个“U型曲线”，然后找到曲线最低点（最清晰的焦点）。

下面将按照**算法逻辑顺序**详细为你讲解整个函数 autofocus_ucurve() 的实现原理与关键点。

---

## **一、算法整体流程概览**

从宏观上看：

```
1. 初始化配置、参数与状态
2. 在不同焦点位置采样（移动焦点 → 拍照 → 测量星点HFD）
3. 记录每个焦点位置的 HFD 值
4. 对这些数据进行多项式拟合（形成U型曲线）
5. 求曲线最低点（HFD最小处） → 最佳焦点位置
6. 将焦点器移动到该最佳位置
7. 校验、结束或恢复原位
```

这正是自动对焦（autofocus）中最常见的流程。

---

## **二、参数与初始化**

```c
double steps = AGENT_IMAGER_FOCUS_UCURVE_STEP_ITEM->number.value;
DEVICE_PRIVATE_DATA->ucurve_samples_number = (int)rint(AGENT_IMAGER_FOCUS_UCURVE_SAMPLES_ITEM->number.value);
int limit = AF_MOVE_LIMIT_UCURVE * steps * DEVICE_PRIVATE_DATA->ucurve_samples_number;
```

这几行定义了算法的“步进和限制条件”：

- steps：每次移动焦点的步进量（例如：每次移动50步）
    
- ucurve_samples_number：采样点数量（例如9点或11点）
    
- limit：最大允许移动距离（防止跑出焦点器行程）
  

然后：

```c
bool moving_out = true;
int sample = 0;
double hfds[stars][samples];
double focus_pos[samples];
```

- moving_out：焦点移动方向标志（向外or向内）
    
- hfds：存储每颗星在每个焦点位置的HFD值
    
- focus_pos：记录每次采样时焦点器的绝对位置
    

---

## **三、采样阶段（核心循环）**

  

算法进入主循环：

```c
while (repeat) {
    ...
    capture_and_process_frame(device, NULL);
    ...
    move_focuser_with_overshoot_if_needed(device, moving_out, steps, ...);
}
```

每一次循环代表一次**采样步骤**，过程如下：

  

### **步骤1：拍照并计算HFD**

  

通过 capture_and_process_frame() 拍摄图像并提取星点的HFD（Half Flux Diameter，半通量直径）。

HFD衡量星点的“大小”，越小表示越清晰（对焦越好）。

  

HFD来自星点亮度分布：

```
HFD = 直径，使得包围星点总光通量的一半。
```

因此，对焦的目标是 **最小化HFD**。

  

### **步骤2：多帧取优**

  

代码中：

```c
for (int i = 0; i < 3 && frame_count < stack_value; i++)
```

会拍多帧，选出HFD最小（质量最高）的一帧作为该位置的代表值，避免单帧误差。

  

### **步骤3：保存采样数据**

  

一旦成功测量到HFD：

```c
focus_pos[sample_index] = DEVICE_PRIVATE_DATA->focuser_position;
hfds[i][sample_index] = AGENT_IMAGER_STATS_HFD_ITEM[i].number.value;
```

保存该焦点位置与HFD值，形成 (position, HFD) 点对。

---

## **四、确定移动方向**

  

在前两次采样后，算法会判断清晰度变化趋势：

```c
if (quality_comparator(prev_quality, quality, star_count) >= 0)
    moving_out = true; // 越移越糊 → 说明原来方向朝“焦外”
else
    moving_out = false; // 越移越清晰 → 说明焦点在“反方向”
```

所以它会：

1. 从当前焦点位置出发，
    
2. 判断HFD是变大还是变小，
    
3. 自动决定往哪一边继续移动去“扫描整条U曲线”。
    

---

## **五、U曲线数据采集**

  

随着循环推进：

- 焦点位置不断变化；
    
- 每次都拍照并测量HFD；
    
- 最终获得一组 (focuser_pos, HFD) 数据点。
    

  

如果画出来，大概是这样👇：

```
HFD
│          *  *
│       *        *
│     *            *
│  *                 *
└─────────────────────────▶ 焦点位置
          最小HFD点（焦点）
```

---

## **六、U曲线拟合（Polynomial Fit）**

  

当采集完所有样本后：

```c
indigo_polynomial_fit(samples, focus_pos, hfds[n], UCURVE_ORDER + 1, polynomial);
```

这一步做的是多项式拟合（通常是二次多项式，即二次曲线）。

形式如下：

```
HFD = a*x² + b*x + c
```

拟合后，再求导：

```
dHFD/dx = 2a*x + b = 0
```

解得最小点：

```
x_min = -b / (2a)
```

也就是 **最佳焦点位置（best_focus）**。

---

## **七、多星合成与异常剔除**

  

代码支持多星测量（Multi-star Focus）：

```c
double best_focuses[stars_used];
best_focus = reduce_ucurve_best_focus(best_focuses, stars_used);
```

它会：

1. 对每颗星单独拟合出一个最小点；
    
2. 将所有结果汇总；
    
3. 去掉离群值（异常星）；
    
4. 平均出最终焦点位置。
    

  

这样可以显著提高鲁棒性，防止单颗星测量误差。

---

## **八、移动到最佳焦点位置**

```c
move_focuser_with_overshoot_if_needed(device, !moving_out, steps_to_focus, ...);
```

算法根据刚算出的 best_focus，让焦点器移动到这个位置。

---

## **九、验证聚焦质量**

  

最后再次拍一张图，验证：

- 当前HFD是否接近曲线最低点；
    
- 偏差超过20%视为失败；
    
- 否则视为对焦成功。
    

```c
if (AGENT_IMAGER_STATS_FOCUS_DEVIATION_ITEM->number.value > 20)
    focus_failed = true;
```

---

## **十、收尾与错误处理**

  

若失败（如星点消失、超出移动范围、拟合错误等）：

- 打印错误日志；
    
- 恢复反向间隙设置；
    
- 可选恢复焦点初始位置。
    

---

## **✳️ 总结：核心数学逻辑**

|**步骤**|**内容**|**数学/逻辑含义**|
|---|---|---|
|1|采集焦点位置与HFD|得到离散数据点 (x, y)|
|2|二次多项式拟合|求 U 曲线模型 y = a·x² + b·x + c|
|3|寻找极小值|解导数方程 2a·x + b = 0 得最小点|
|4|多星平均|去除离群值，取平均|
|5|移动焦点器|移动至最小点位置完成对焦|

---

## **🔧 工程实现要点**

1. **HFD作为对焦指标**比峰值亮度稳定；
    
2. **多星加权平均**避免因单星饱和或光度异常导致的误差；
    
3. **多帧堆叠取优**提高测量精度；
    
4. **步进自适应**：根据HFD变化方向智能确定扫描方向；
    
5. **多项式拟合**能在噪声下平滑出理想U型曲线；
    
6. **limit约束**避免机械风险。

