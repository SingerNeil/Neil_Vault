# HFD U-Curve 自动对焦 — 数据采集与拟合算法规格书

> **版本**: 1.0  
> **日期**: 2026-03-20  
> **目标读者**: App 端开发团队  
> **范围**: 描述 INDIGO Imager Agent 自动对焦过程中，客户端如何通过 WebSocket 实时获取 HFD 数据点，以及如何在客户端进行 U-Curve 二次拟合并可视化

---

## 1. 系统架构概览

```
┌────────────┐   WebSocket (JSON)   ┌──────────────────────┐
│   App 端   │ ◀────────────────── │   INDIGO Server      │
│  (客户端)  │ ──────────────────▶ │   + Imager Agent     │
└────────────┘   newXxxVector       └──────────────────────┘
                                           │
                                    ┌──────┴──────┐
                                    │  CCD 驱动   │  调焦器驱动
                                    └─────────────┘
```

- **INDIGO Server** 运行在远端设备上，加载 `Imager Agent` 驱动
- **Imager Agent** 既是设备（暴露控制属性）又是客户端（控制 CCD、调焦器）
- **App 端** 通过 WebSocket 连接 Server，接收属性更新推送，发送控制指令
- 自动对焦过程由 Agent 内部状态机驱动，客户端只需 **启动/停止** 并 **被动接收统计数据**

---

## 2. WebSocket 连接

### 2.1 连接地址

```
ws://<server_ip>:7624
```

### 2.2 握手 — 枚举所有属性

连接建立后，客户端发送：

```json
{
  "getProperties": {
    "version": 512,
    "client": "MyApp"
  }
}
```

Server 回复所有设备的全量属性定义（`defNumberVector` / `defSwitchVector` / `defTextVector` 等）。

### 2.3 消息格式总览

| 方向 | 消息类型 | 用途 |
|------|----------|------|
| Server → Client | `defNumberVector` | 首次定义数值属性（含元数据 + 当前值） |
| Server → Client | `setNumberVector` | 属性值更新（增量推送） |
| Client → Server | `newNumberVector` | 请求修改属性值 |
| Server → Client | `defSwitchVector` / `setSwitchVector` | 开关属性定义/更新 |
| Client → Server | `newSwitchVector` | 请求修改开关属性 |

---

## 3. 启动自动对焦

客户端向 Server 发送以下指令序列（顺序执行）：

```json
// 1. 启用预览模式（可选，用于实时预览）
{
  "newSwitchVector": {
    "device": "Imager Agent",
    "name": "CCD_PREVIEW",
    "items": [{"name": "ENABLED", "value": true}]
  }
}

// 2. 启动对焦流程
{
  "newSwitchVector": {
    "device": "Imager Agent",
    "name": "AGENT_START_PROCESS",
    "items": [{"name": "FOCUSING", "value": true}]
  }
}
```

### 3.1 停止/中止对焦

```json
{
  "newSwitchVector": {
    "device": "Imager Agent",
    "name": "AGENT_ABORT_PROCESS",
    "items": [{"name": "ABORT", "value": true}]
  }
}
```

### 3.2 判断对焦流程结束

监听 `AGENT_START_PROCESS` 属性的 `state` 字段：
- `"Busy"` → 对焦进行中
- `"Ok"` → 对焦成功完成
- `"Alert"` → 对焦失败

---

## 4. 数据采集 — `AGENT_IMAGER_STATS` 属性

### 4.1 属性定义

对焦启动后，Server 会频繁推送 `AGENT_IMAGER_STATS` 属性更新。这是**唯一的数据源**，每次调焦器移动并拍摄一帧后推送一次。

**属性全名**: `AGENT_IMAGER_STATS`  
**设备名**: `Imager Agent`  
**类型**: Number（数值型）  
**权限**: RO（只读）

### 4.2 绘制 U-Curve 所需的关键字段

| 字段名 (wire name) | 类型 | 含义 | 用途 |
|---------------------|------|------|------|
| `FOCUS_POSITION` | double | 当前调焦器位置（步数） | **X 轴坐标** |
| `HFD` | double | 半通量直径（arc-pixel） | **Y 轴坐标** |
| `BEST_FOCUS_DEVIATION` | double | 当前 HFD 相对最佳焦点的偏差百分比 | **阶段判断标志** |

### 4.3 完整字段列表（供参考）

| Index | 字段名 | 含义 | 典型值范围 |
|-------|--------|------|------------|
| 0 | `EXPOSURE` | 剩余曝光时间 (s) | 0–3600 |
| 1 | `DELAY` | 剩余延迟时间 (s) | 0–3600 |
| 2 | `FRAME` | 当前帧序号 | 0–N |
| 3 | `FRAMES` | 总帧数 | 0–N |
| 4 | `BATCH_INDEX` | 当前 batch 索引 | 0–N |
| 5 | `BATCH` | 当前 batch 序号 | 0–N |
| 6 | `BATCHES` | batch 总数 | 0–N |
| 7 | `PHASE` | 批次阶段 | 0–N |
| 8 | `DRIFT_X` | X 方向漂移 (px) | -1000–1000 |
| 9 | `DRIFT_Y` | Y 方向漂移 (px) | -1000–1000 |
| 10 | `DITHERING` | 抖动 RMSE | ≥0 |
| 11 | `FOCUS_OFFSET` | 对焦偏移量（步） | ±65535 |
| 12 | `FOCUS_POSITION` | 调焦器位置（步） | ±16777215 |
| 13 | `RMS_CONTRAST` | RMS 对比度 | 0–1 |
| 14 | `BEST_FOCUS_DEVIATION` | 最佳焦点偏差 (%) | -100–100, 初始值=100 |
| 15 | `FRAMES_TO_DITHERING` | 离下次抖动的帧数 | ≥0 |
| 16 | `BAHTINOV_ERROR` | Bahtinov 对焦误差 | ≥0 |
| 17 | `MAX_STARS_TO_USE` | 最大使用星点数 | ≥0 |
| 18 | `PEAK` | 峰值 | 0–65535 |
| 19 | `FWHM` | 半高全宽 | 0–65535 |
| 20 | `HFD` | 半通量直径 | 0–65535 |

### 4.4 WebSocket 消息示例

**首次定义**（连接后收到）：

```json
{
  "defNumberVector": {
    "version": 512,
    "device": "Imager Agent",
    "name": "AGENT_IMAGER_STATS",
    "group": "Agent",
    "label": "Statistics",
    "perm": "ro",
    "state": "Idle",
    "items": [
      {"name": "EXPOSURE", "label": "Exposure", "value": 0, "min": 0, "max": 3600, "step": 0, "format": "%g"},
      {"name": "DELAY", "label": "Delay", "value": 0, "min": 0, "max": 3600, "step": 0, "format": "%g"},
      {"name": "FRAME", "label": "Frame", "value": 0, "...": "..."},
      "... (共 21 个 item，完整列表见上表)"
    ]
  }
}
```

**实时更新**（对焦过程中频繁收到）：

```json
{
  "setNumberVector": {
    "device": "Imager Agent",
    "name": "AGENT_IMAGER_STATS",
    "state": "Busy",
    "items": [
      {"name": "EXPOSURE", "value": 0},
      {"name": "DELAY", "value": 0},
      {"name": "FRAME", "value": 5},
      {"name": "FRAMES", "value": 0},
      {"name": "BATCH_INDEX", "value": 0},
      {"name": "BATCH", "value": 0},
      {"name": "BATCHES", "value": 0},
      {"name": "PHASE", "value": 0},
      {"name": "DRIFT_X", "value": 0.123},
      {"name": "DRIFT_Y", "value": -0.045},
      {"name": "DITHERING", "value": 0},
      {"name": "FOCUS_OFFSET", "value": -200},
      {"name": "FOCUS_POSITION", "value": 16880},
      {"name": "RMS_CONTRAST", "value": 0},
      {"name": "BEST_FOCUS_DEVIATION", "value": 100},
      {"name": "FRAMES_TO_DITHERING", "value": 0},
      {"name": "BAHTINOV_ERROR", "value": 0},
      {"name": "MAX_STARS_TO_USE", "value": 1},
      {"name": "PEAK", "value": 42350},
      {"name": "FWHM", "value": 3.82},
      {"name": "HFD", "value": 7.65}
    ]
  }
}
```

---

## 5. 数据点解析与阶段判断

### 5.1 核心逻辑：两阶段模型

利用 `BEST_FOCUS_DEVIATION` 字段将收到的数据分为两个阶段：

```
┌──────────────────────────────────────────────────────────┐
│  |BEST_FOCUS_DEVIATION| ≥ 90  →  【采样阶段】          │
│  Agent 正在移动调焦器并采集 HFD 数据                     │
│  每条消息 = 一个数据点 (FOCUS_POSITION, HFD)             │
│                                                          │
│  |BEST_FOCUS_DEVIATION| < 90  →  【结果阶段】           │
│  Agent 已完成拟合，正在移向最佳焦点                       │
│  此时 FOCUS_POSITION = 最终位置, HFD = 最终 HFD          │
└──────────────────────────────────────────────────────────┘
```

### 5.2 伪代码

```
收到 AGENT_IMAGER_STATS 更新:
    pos = items["FOCUS_POSITION"].value
    hfd = items["HFD"].value
    dev = items["BEST_FOCUS_DEVIATION"].value

    IF hfd <= 0:
        忽略此条消息（无效测量）
        RETURN

    IF |dev| >= 90:
        // ═══ 采样阶段 ═══
        IF 数据点列表最后一个点的位置 ≈ pos (差值 < 0.1 步):
            // 同一位置的多次测量，更新 HFD（Agent 可能在该位置重新测量）
            更新最后一个点的 HFD = hfd
        ELSE:
            // 新位置
            添加数据点 { x: pos, y: hfd }

        选取最后 N 个点进行拟合（N = 可配置的窗口大小，当前 Web 端为 10）
        执行二次拟合 → 更新图表

    ELSE:
        // ═══ 结果阶段 ═══
        显示最终结果:
            Device Best   = pos（整数，调焦器最终位置）
            HFD (Final)   = hfd
            Deviation     = dev%
```

### 5.3 数据点选择策略

Agent 在 U-Curve 对焦中通常采集 **7~24 个样本点**（由 `AGENT_IMAGER_FOCUS_UCURVE_SAMPLES` 属性控制）。

客户端显示时，将最后 `maxSamples`（当前设为 10）个点标记为 **活跃点（蓝色）** 参与拟合，更早的点标记为 **非活跃点（灰色）** 仅作展示。这样确保拟合使用的是调焦器最终扫过的、包含 U-Curve 底部的对称区间。

---

## 6. U-Curve 拟合算法

### 6.1 客户端拟合目的

客户端拟合的目的是 **实时可视化**，让用户直观看到 HFD 变化趋势与估算的最佳焦点。最终结果以 Server 端计算为准。

### 6.2 算法：最小二乘法二次多项式拟合

**模型**：$y = a x'^2 + b x' + c$，其中 $x' = x - \bar{x}$

给定 $n$ 个数据点 $(x_i, y_i)$，先计算 $\bar{x} = \frac{1}{n}\sum x_i$，然后令 $x'_i = x_i - \bar{x}$。

构造正规方程组 $\mathbf{A}\beta = \mathbf{d}$：

$$
\begin{bmatrix}
n & \sum x'_i & \sum x'^2_i \\
\sum x'_i & \sum x'^2_i & \sum x'^3_i \\
\sum x'^2_i & \sum x'^3_i & \sum x'^4_i
\end{bmatrix}
\begin{bmatrix}
c \\ b \\ a
\end{bmatrix}
=
\begin{bmatrix}
\sum y_i \\
\sum x'_i y_i \\
\sum x'^2_i y_i
\end{bmatrix}
$$

使用 **Cramer 法则** 求解 3×3 线性方程组。

### 6.3 为什么必须中心化

调焦器位置 $x$ 通常在 10000~30000 范围。不中心化时：

$$\sum x^4_i \approx 10 \times 17000^4 \approx 8 \times 10^{17}$$

而 JavaScript `float64` 只有 15~16 位有效数字。Cramer 法则展开行列式时，近似相等的巨大数相减（灾难性相消），二次项系数 $a$ 将严重失真。

中心化后 $x'_i \in [-100, +100]$ 量级，$\sum x'^4_i \approx 10^8$，精度损失可忽略。

### 6.4 最佳焦点计算

拟合完成后：

1. **验证**：$a > 0$（开口向上才是有效 U-Curve），否则判定拟合失败
2. **中心坐标下的最小值**：$x'_{min} = -\frac{b}{2a}$
3. **换算回原始坐标**：$x_{min} = x'_{min} + \bar{x}$（即 **Calculated Best** 位置）
4. **最小 HFD**：$y_{min} = a \cdot x'^2_{min} + b \cdot x'_{min} + c$（即 **Min HFD (Calc)**）
5. **曲率**：输出 $a$ 值（即 **Curvature (a)**）

### 6.5 拟合函数参考实现（JavaScript）

```javascript
function calculateHfdQuadratic(data) {
    if (data.length < 3) return null;
    const n = data.length;

    // 中心化：避免大数精度相消
    const xMean = data.reduce((s, p) => s + p.x, 0) / n;

    let Sx = 0, Sy = 0, Sx2 = 0, Sx3 = 0, Sx4 = 0, Sxy = 0, Sx2y = 0;
    for (const p of data) {
        const x = p.x - xMean;
        const y = p.y;
        const x2 = x * x;
        Sx  += x;     Sy   += y;
        Sx2 += x2;    Sx3  += x2 * x;   Sx4 += x2 * x2;
        Sxy += x * y; Sx2y += x2 * y;
    }

    // 正规方程矩阵 A 和右端向量 b
    const A = [
        [n,   Sx,  Sx2],
        [Sx,  Sx2, Sx3],
        [Sx2, Sx3, Sx4]
    ];
    const b = [Sy, Sxy, Sx2y];

    // 3×3 行列式
    const det = m =>
        m[0][0] * (m[1][1]*m[2][2] - m[1][2]*m[2][1]) -
        m[0][1] * (m[1][0]*m[2][2] - m[1][2]*m[2][0]) +
        m[0][2] * (m[1][0]*m[2][1] - m[1][1]*m[2][0]);

    const detA = det(A);
    if (Math.abs(detA) < 1e-12) return null;  // 奇异矩阵

    // Cramer 法则求解
    const c = det([
        [b[0], A[0][1], A[0][2]],
        [b[1], A[1][1], A[1][2]],
        [b[2], A[2][1], A[2][2]]
    ]) / detA;
    const b_coef = det([
        [A[0][0], b[0], A[0][2]],
        [A[1][0], b[1], A[1][2]],
        [A[2][0], b[2], A[2][2]]
    ]) / detA;
    const a = det([
        [A[0][0], A[0][1], b[0]],
        [A[1][0], A[1][1], b[1]],
        [A[2][0], A[2][1], b[2]]
    ]) / detA;

    if (a <= 0) return null;  // 开口向下 → 无效

    const minX_centered = -b_coef / (2 * a);
    const minX = minX_centered + xMean;  // 还原到原始坐标
    const minY = a * minX_centered * minX_centered + b_coef * minX_centered + c;

    return { a, b: b_coef, c, xMean, minX, minY };
}
```

### 6.6 曲线绘制

绘制拟合曲线时，X 范围在活跃点区间基础上两端各延伸 50%，在该范围内等分 50 个步点：

```javascript
const xs = activePoints.map(p => p.x);
const rangeMin = Math.min(...xs);
const rangeMax = Math.max(...xs);
const extend  = (rangeMax - rangeMin) * 0.5;
const drawMin = rangeMin - extend;
const drawMax = rangeMax + extend;

const curvePoints = [];
for (let i = 0; i <= 50; i++) {
    const x  = drawMin + i * (drawMax - drawMin) / 50;
    const xc = x - fit.xMean;  // 必须用中心化坐标求值
    const y  = fit.a * xc * xc + fit.b * xc + fit.c;
    curvePoints.push({ x, y });
}
```

---

## 7. 与 Server 端算法的差异对照

| 维度 | Server 端（C） | 客户端（JS/App） |
|------|----------------|-------------------|
| **用途** | 计算最佳焦点并驱动调焦器移动 | 实时可视化（仅供参考） |
| **多项式阶数** | 4 阶（5 个系数） | 2 阶（3 个系数） |
| **拟合方法** | 最小二乘 + Gauss-Jordan 消元 | 最小二乘 + Cramer 法则 |
| **数值精度** | `long double`（80/128 bit） | `double`（64 bit），需中心化补偿 |
| **多星支持** | 多星独立拟合 → 众数融合 | 仅使用第一颗星的 HFD |
| **数据量** | 使用全部 U-Curve 采样点（7~24） | 使用最后 N 个（当前 N=10） |
| **最终结果** | 驱动调焦器移动到最佳位置 | 仅在 UI 上显示估算值 |

> **重要**：App 端拟合结果（Calculated Best / Min HFD）仅是视觉反馈，不影响 Agent 的实际对焦决策。Agent 内部有独立的 4 阶多项式拟合。

---

## 8. UI 展示规格

### 8.1 图表

- **散点图 + 拟合曲线** 叠加显示
- X 轴：调焦器位置（步数）
- Y 轴：HFD 值
- 活跃点（参与拟合）：蓝色实心圆，直径 10px
- 非活跃点（早期采样）：灰色实心圆，直径 10px
- 拟合曲线：蓝色平滑线，2px 宽度，带半透明蓝色填充

### 8.2 指标面板

| 指标名 | 来源 | 说明 |
|--------|------|------|
| **Device Best** | `FOCUS_POSITION`（结果阶段） | Agent 驱动到的最终调焦器位置（整数） |
| **HFD (Final)** | `HFD`（结果阶段） | 最终位置实测 HFD |
| **Deviation** | `BEST_FOCUS_DEVIATION`（结果阶段） | 偏差百分比，越小越好 |
| **Calculated Best** | 拟合 `minX` | 客户端二次拟合估算的最佳位置 |
| **Min HFD (Calc)** | 拟合 `minY` | 客户端估算的最小 HFD |
| **Curvature (a)** | 拟合系数 `a` | 二次项系数，反映 HFD 对焦灵敏度 |

### 8.3 计时器

在对焦启动时记录 `startTime`，实时更新显示已用时间。对焦结束时停止并显示最终用时，成功为绿色，失败为红色。

---

## 9. 完整数据流时序

```
Client                                     Server (Imager Agent)
  │                                                │
  │─── newSwitchVector ───────────────────────────▶│ AGENT_START_PROCESS.FOCUSING=true
  │                                                │
  │    （Agent 开始 U-Curve 对焦流程）                │
  │                                                │
  │◀── setSwitchVector ────────────────────────────│ AGENT_START_PROCESS.state="Busy"
  │                                                │
  │    ┌─── 循环：移动调焦器 → 拍摄 → 计算 HFD ────────┐
  │    │                                           │
  │◀── setNumberVector ────────────────────────────│ AGENT_IMAGER_STATS
  │    │  FOCUS_POSITION = 16820                   │  ← 数据点 1
  │    │  HFD = 13.5                               │
  │    │  BEST_FOCUS_DEVIATION = 100               │  ← |dev|≥90 → 采样阶段
  │    │                                           │
  │◀── setNumberVector ────────────────────────────│ AGENT_IMAGER_STATS
  │    │  FOCUS_POSITION = 16840                   │  ← 数据点 2
  │    │  HFD = 12.3                               │
  │    │  BEST_FOCUS_DEVIATION = 100               │
  │    │                                           │
  │    │  ... （每移动一步推送一次）...                │
  │    │                                           │
  │◀── setNumberVector ────────────────────────────│ AGENT_IMAGER_STATS
  │    │  FOCUS_POSITION = 16940                   │  ← 数据点 N
  │    │  HFD = 5.1                                │
  │    │  BEST_FOCUS_DEVIATION = 100               │
  │    └───────────────────────────────────────────┘
  │                                                │
  │    （Agent 内部完成 4 阶多项式拟合，               │
  │     移动到最佳焦点位置）                          │
  │                                                │
  │◀── setNumberVector ────────────────────────────│ AGENT_IMAGER_STATS
  │    FOCUS_POSITION = 16917                      │  ← 最终位置
  │    HFD = 4.2953                                │  ← 最终 HFD
  │    BEST_FOCUS_DEVIATION = 0.11                 │  ← |dev|<90 → 结果阶段
  │                                                │
  │◀── setSwitchVector ────────────────────────────│ AGENT_START_PROCESS.state="Ok"
  │                                                │
  │    对焦完成                                     │
```

---

## 10. 接口汇总

### 10.1 需要监听的属性

| 属性                                     | 用途                    |
| -------------------------------------- | --------------------- |
| `Imager Agent` / `AGENT_IMAGER_STATS`  | HFD 数据点采集             |
| `Imager Agent` / `AGENT_START_PROCESS` | 对焦流程状态（Busy/Ok/Alert） |

### 10.2 需要发送的指令

| 操作 | 目标属性 | 值 |
|------|----------|-----|
| 启动对焦 | `AGENT_START_PROCESS` | `{"FOCUSING": true}` |
| 中止对焦 | `AGENT_ABORT_PROCESS` | `{"ABORT": true}` |
| 启用预览 | `CCD_PREVIEW` | `{"ENABLED": true}` |

---

## 附录 A：测试用例建议

| 场景 | 验证项 |
|------|--------|
| 调焦器位置 ~16000 | 拟合曲线应穿过数据点，而非退化为水平线 |
| 数据点 < 3 | 不绘制拟合曲线 |
| HFD = 0 的消息 | 应丢弃 |
| 调焦器位置范围极小（如所有点位于同一位置 ±1） | 应优雅降级（不拟合或显示提示） |
| 快速连续收到相同位置的更新 | 应覆盖同一点而非新增 |
| 对焦中途中止 | 显示已采集的点和曲线，停止更新 |
