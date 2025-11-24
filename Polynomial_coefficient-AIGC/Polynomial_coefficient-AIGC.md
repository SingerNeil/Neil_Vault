这段代码正是 U-Curve 自动对焦算法里用于“拟合出清晰度曲线”的核心数学部分——
****多项式最小二乘拟合（Polynomial Least Squares Fitting）***。

---

## ** 一、算法的作用**

  

在自动对焦中，我们在不同焦点位置（例如焦距 motor position）采样得到一系列点：

  

$(x_i, y_i)$

其中：

- $x_i$：焦点位置（focus position）
    
- $y_i$：清晰度指标（如HFD、FWHM、Sharpness等）
    

  

这些点通常呈现一个 **U型趋势**（清晰度在最佳焦点处最小），所以我们想拟合出一条平滑的曲线，找到它的**最低点**，即最佳焦点位置。

---

## **二、算法原理（Least Squares Polynomial Fit）**

  

我们希望找到一个 n 次多项式：

$$y = a_0 + a_1 x + a_2 x^2 + \dots + a_n x^n$$

使其尽可能“贴近”所有采样点 $(x_i, y_i)$。

  

最小二乘法要求最小化平方误差：

$$S = \sum_{i=1}^{m} (y_i - (a_0 + a_1 x_i + a_2 x_i^2 + \dots + a_n x_i^n))^2$$

  

对每个系数 a_j 求偏导并令其为零：

$\frac{\partial S}{\partial a_j} = 0$

得到 **法方程（Normal Equations）**：

$\sum_{i=1}^{m} x_i^j y_i = \sum_{k=0}^{n} a_k \sum_{i=1}^{m} x_i^{j+k}$

这是一组线性方程：

$A \cdot a = B$

其中：

$$ A_{jk} = \sum x_i^{j+k}
    
- B_j = \sum y_i x_i^j
    
- a = [a_0, a_1, …, a_n]^T$$
    

  

解这个线性系统就能得到最优的多项式系数。

---

## ** 三、代码逻辑分解**

  

这段 C 代码正是 **手动实现了法方程 + 高斯消元求逆矩阵** 的过程。

---

### **① 计算 B 向量（右侧）**

```c
for (ii = 0; ii < point_count; ii++) {
	x = x_values[ii];
	y = y_values[ii];
	powx = 1;
	for (jj = 0; jj < (order + 1); jj++) {
		B[jj] += y * powx;
		powx *= x;
	}
}
```

即：

$B_j = \sum y_i x_i^j$

---

### **② 计算幂次矩阵 P（左侧元素的公共部分）**

```c
P[0] = point_count;
for (ii = 0; ii < point_count; ii++) {
	x = x_values[ii];
	powx = x;
	for (jj = 1; jj < ((2 * (order + 1)) + 1); jj++) {
		P[jj] += powx;
		powx *= x;
	}
}
```

即：

$P_k = \sum x_i^k$

  

这些是构造矩阵 A 时的基础项。

---

### **③ 构造 A 矩阵（法方程的系数矩阵）**

```c
for (ii = 0; ii < (order + 1); ii++) {
	for (jj = 0; jj < (order + 1); jj++) {
		A[(ii * (2 * (order + 1))) + jj] = P[ii+jj];
	}
	A[(ii*(2 * (order + 1))) + (ii + (order + 1))] = 1;
}
```

构造的是一个扩展矩阵 [A | I]，用于之后求逆（高斯-约旦消元）。

---

### **④ 高斯-约旦消元法求逆矩阵**

```c
for (ii = 0; ii < (order + 1); ii++) {
	x = A[(ii * (2 * (order + 1))) + ii];
	if (x != 0) {
		// 当前行标准化
		for (kk = 0; kk < (2 * (order + 1)); kk++)
			A[(ii * (2 * (order + 1))) + kk] /= x;

		// 对其他行做消元
		for (jj = 0; jj < (order + 1); jj++) {
			if (jj != ii) {
				y = A[(jj * (2 * (order + 1))) + ii];
				for (kk = 0; kk < (2 * (order + 1)); kk++)
					A[(jj * (2 * (order + 1))) + kk] -= y * A[(ii * (2 * (order + 1))) + kk];
			}
		}
	}
}
```

这部分手工求得 A^{-1}，将单位矩阵 I 变换成 A^{-1}。

---

### **⑤ 最终计算多项式系数**

```c
for (ii = 0; ii < (order + 1); ii++) {
	for (jj = 0; jj < (order + 1); jj++) {
		x = 0;
		for (kk = 0; kk < (order + 1); kk++) {
			x += A[(ii * (2 * (order + 1))) + (kk + (order + 1))] * B[kk];
		}
		polynomial_coefficients[ii] = x;
	}
}
```

相当于：

a = A^{-1} B

即得到多项式的各阶系数。

---

## ** 四、在U-Curve算法中的用途**

  

在自动对焦过程中：

1. 控制镜头在不同位置采样焦点清晰度（x: focus pos, y: HFD）
    
2. 将采样点输入 indigo_polynomial_fit 拟合多项式
    
3. 求导：
    
    $\frac{dy}{dx} = a_1 + 2a_2x + 3a_3x^2 + …$
    
1. 令导数为0，解出极小值点 $x_{best}$
    
2. 该位置即为最佳对焦位置。
    

---

## ** 总结**

| **阶段** | **数学含义**    | **代码对应**              |
| ------ | ----------- | --------------------- |
| 采样     | 收集 $(x, y)$ | autofocus_ucurve 采样部分 |
| 拟合     | 求多项式系数 a    | indigo_polynomial_fit |
| 求极值    | $dy/dx=0$   | 通常用求导解方程              |
| 应用     | 回到该位置对焦     | 控制马达位置                |

