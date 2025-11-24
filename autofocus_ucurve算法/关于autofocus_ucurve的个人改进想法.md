

---

![[autofocus_ucurve.pdf]]


```c
if (quality_has_zero(hfds[n], DEVICE_PRIVATE_DATA->ucurve_samples_number))
{
	INDIGO_DRIVER_DEBUG(DRIVER_NAME, "UC: U-Curve data for star #%d has bad quality data points - skipping", n);
	continue; // 跳过质量不佳的数据点
}
```
***其中，quality_has_zero的定义是***
```c
static bool quality_has_zero(double *quality, int count) {
    for (int i = 0; i < count; i++) {
        if (quality[i] == 0) {
            INDIGO_DRIVER_DEBUG(DRIVER_NAME, "Quality[%d] = %g -> has_zero = true", i, quality[i]);
            return true;
        }
    }
    return false;
}
```

## 方法语境 ：
这个方法在每次刚进入 “对每个星体进行U型函数拟合” 的循环时使用。用意是通过判断每个星体的hfds数据，直接跳过含有HFD为0（无效测量）的星体，进入下一个循环，以此来提高运算效率。

## 看法：
个人认为这个判断过于严格，因为在这个算法里，每一个被find出来的星体到了这一步，其hfds[]数组内部都有的不少HFD值。若仅是因为这些值中只有一位是0就摒弃这个数据，我觉得太严格了。

1. 现有的问题：
- 只要数组中有一个 0 值就认为整个星点测量无效
- 没有考虑临时性的测量失败
- 可能会浪费大量有效数据

1. 可能的原因导致单次测量 HFD = 0：
- 瞬时的大气抖动
- 云层短暂遮挡
- 望远镜追踪的小幅抖动
- 相机的临时噪声

## 建议：
将逻辑从  **“只要含有0，该星体数据无效“**，
变成  **”若含有多于n个0，该星体数据无效“**。

**n = 一个星体含有的总样本数 ✖️ 某个比例**

同时，若监测到该星体含多个0数（无效数据）据但是尚处于可信范围内：检验该星体含有的几个0数据 是否是连续的，若连续，最长连续序列的长度是多少。
若长度超过一个固定值，则说明采样时可能出跟踪丢失，不是简单的误差。则该星体的数据不必再采用。

AIGC:
```c
// 建议的改进版本
static bool quality_is_unreliable(double *quality, int count) {
    /* 1. 统计部分 */
    int zero_count = 0;    // 无效测量（HFD=0）的计数
    int valid_count = 0;   // 有效测量的计数
    
    // 遍历所有测量值进行统计
    for (int i = 0; i < count; i++) {
        if (quality[i] == 0) {
            zero_count++;      // 累计无效测量数
        } else {
            valid_count++;     // 累计有效测量数
        }
    }
    
    /* 2. 总体质量判断 */
    // 如果无效测量超过30%，认为数据不可靠
    if (zero_count > count * 0.3) {  // 0.3是可调节的阈值
        INDIGO_DRIVER_DEBUG(DRIVER_NAME, 
            "Star quality unreliable: %d of %d measurements are invalid", 
            zero_count, count);
        return true;  // 数据不可靠
    }
    
    /* 3. 连续性判断 */
    int consecutive_zeros = 0;     // 当前连续无效测量计数
    int max_consecutive_zeros = 0; // 最大连续无效测量数
    
    // 寻找最长的连续无效序列
    for (int i = 0; i < count; i++) {
        if (quality[i] == 0) {
            consecutive_zeros++;  // 发现无效测量，增加连续计数
            // 更新最大连续无效数
            if (consecutive_zeros > max_consecutive_zeros) {
                max_consecutive_zeros = consecutive_zeros;
            }
        } else {
            consecutive_zeros = 0;  // 遇到有效测量，重置连续计数
        }
    }
    
    // 如果有太长的连续无效序列，可能表示跟踪丢失
    if (max_consecutive_zeros > 3) {  // 3是可调节的阈值
        INDIGO_DRIVER_DEBUG(DRIVER_NAME, 
            "Star tracking may be lost: %d consecutive invalid measurements", 
            max_consecutive_zeros);
        return true;  // 数据不可靠
    }
    
    return false;  // 通过所有检查，数据可靠
}
```