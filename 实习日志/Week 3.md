
---
# TODO：
1. 继续优化 **`autofocus_ucruve`** 算法
2. 在INDIGO Client的**control panel**界面添加合适的Backlash参数，具体是何参数需要在调试设备时寻找![](assets/Week%203/file-20251124145514652.png)
3. 尝试理解 **`find_star`** 算法，即设备是如何找到星体的

# BackLash
## 本地数据
手动记录了两组**焦点移动方向相反**的 **`autofocus_ucurve`** 过程数据，具体包括：

1. **`positions`**：记录不同焦点位置的空间坐标信息（如镜头位移毫米值或传感器坐标）；
    
2. **`hfds`**：记录对应焦点位置的对焦质量指标（HFD值），该值可视为对焦清晰度的量化评估——**HFD值越低，表示对焦质量越好**。

## 数据处理

### 数据关联

将同一自动对焦过程中采集的 `positions`（焦点位置）与 `hfd`（对焦质量值）按顺序一一对应，形成两组同步变化的序列数据；
    
### 曲线拟合
基于这两组对应数据，分别构建 **二阶多项式拟合曲线**（二次函数模型），数学表达式为：
$$HFD=a⋅(Position)^2+b⋅(Position)+c$$可视为
$$y=ax^2+bx+c$$
    

其中：

- a、b、c为多项式系数，通过最小二乘法拟合确定；
    
### 目标
通过拟合曲线分析焦点位置与对焦质量之间的非线性关系（例如是否存在U型/倒U型趋势，即特定位置区间内对焦质量最优）。

### 在Microsoft Excel中拟合曲线

两次 **`autofocus_ucurve`** 过程中记录的数据：
![](assets/Week%203/file-20251124145528520.png)

在此基础上，按照上述步骤在Microsoft Excel内通过 **插入浮点图** 和 **添加趋势线** ，所拟合出的两条曲线：
![](assets/Week%203/file-20251124145534751.png)
	可以得知两条曲线趋近于在Positions轴上的平移关系，且平移距离为 ***113.315***（两次`autofocus_ucurve`中`best_focuses`的差值）
	
	于是可以视为，该设备的**backlash**约等于 ***113.315***

# indigo_server相关

## Indigo Server 帮助文档查看
- 以 `root` 账户登录 [192.168.124.12:7624](http://192.168.124.12:7624) 后，可通过以下命令查看服务端帮助信息：
```
/usr/local/bin/indigo_server --help
```
## GDBServer 调试模式启动
- 在 `root` 权限下，如需通过 `gdbserver` 输出日志并开启调试端口（2333），执行:
```
gdbserver :2333 /usr/local/bin/indigo_server -l -- -vv
```
- 执行后可在 Visual Studio Code 中连接该端口进行调试。

# 日志管理(log)
## 查看历史日志
- 在 `root` 账户下，通过以下命令可以查看历史日志记录：
```
cat /var/log/messages
```

## 监听
在 `root` 账户下，通过以下命令可查看最近最新的日志，同时保持后续实时监听日志输出：
```
tail -f /var/log/messages
```

