
---
# WIFI
连接WIFI以访问内网: ***EDS-5G***, ***EDS***
password: ***20210929***

# 连接INDIGO Server Manager
## 1. 连接
- 公司WI-FI后，确保设备连上有限网在浏览器访问 [192.168.124.12:7624](http://192.168.124.12:7624) ，或在设备只连接WI-FI时通过 [192.168.124.248:7624](http://192.168.124.248:7624) 以进入 **INDIGO Server Manager** 控制相机（端口和ip有多个组合，这里仅展示一个举例） 。
## 2. 配置
- 进入控制云台后，按照如下配置：

![](assets/Week%201/file-20251124145405278.png)

# 在linux里登录账号
## root账户
在terminal通过命令 
有线网络:
```
ssh root@192.168.124.12
``` 
WIFI:
```
ssh root@192.168.124.248
```
进入管理员账户；
password: **`zynq`**。

若登陆别的账号，有时候需要加上端口（例如7624）：
```
ssh [username]@192.168.124.12 -p 7624
```

可以在root权限账号下，通过命令
- **`/etc/init.d/S91indigo stop`** 来停止设备。
- **`/etc/init.d/S91indigo start`** 来重新启动indigo，以便查看无穷无尽的日志(:=P）。

# 通过gdb-server和gdb来对indigo设备进行debug

通过现在设备上开一个**gdb-server**的端口，再通过在已ssh登录过的VScode上，配置好launch.json文件，可以以一种不本地保存代码的方式来调试设备（板子）。
## launch.json配置内容：
```C
{
	"version": "0.2.0",
	"configurations": [
		{
			"name": "Debug edscap (gdbserver)",
			"type": "cppdbg",
			"request": "launch",
			"program": "${workspaceFolder}/output/usr/local/bin/indigo_server", // TODO: 替换为你的目标可执行文件路径
			"miDebuggerServerAddress": "192.168.124.12:2333", // TODO: 替换为你的目标板IP和端口
			"miDebuggerPath": "${workspaceFolder}/toolchain/bin/arm-buildroot-linux-gnueabihf-gdb", // TODO: 替换为你的交叉GDB路径
			"cwd": "${workspaceFolder}",
			"stopAtEntry": true
		}
	]
}
```

## debug准备步骤（在root账户下）
### 1. 检验HOME路径是否正确
执行 `echo $HOME` ，若是输出 /root ，代表HOME路径恢复了默认，需要执行：
``
```
export HOME=/media/ssd
````
### 2. 开放ndigo设备上的server端口
#### 在使用端口前，先确定目标端口是否被占用

监测查看indigo的gdb-server：
```	
ps aux |grep gdbserver
```

若被占用，则在获得其PID后使用`kill 14671`（PID例为14671）杀死占用端口的进程；或者使用`kill gdbserver`。
![](assets/Week%201/file-20251124145444722.png)

#### **开启gdb-server的端口**
确认端口未被占用后，执行
```
gdbserver :2333 /usr/local/bin/indigo_server
```
或
```
gdbserver :2333 /usr/local/bin/indigo_server --
```
（以我个人选择的端口为==[2333]==为例）。

## 在 Visual Studio Code 上进行debug
### 代码同步
 若存在代码内容的修改，则需要在Visual Studio Code的terminal里输入两条命令（位于~/code/seeing下）：
```
scp -O -r ./output/usr/local/lib/* root@192.168.124.12:/usr/local/lib/
```

```
scp -O -r ./output/usr/local/bin/* root@192.168.124.12:/usr/local/bin/
```
做到将 **设备板子内的代码**和 **个人账户上的修改过的代码** 同步。
或直接输入以下命令来粗暴地替换全部（不建议）
```
scp -r ./output/usr/local/* root@192.168.124.12:/usr/local/
```
### 开始debug
1. 完成同步前，需要在VSCode的terminal内输入 **`make all`** 来编译代码。
2. 在VSCode里点击debug图标，选择***Debug edscap(gdbserver)*** 后直接设置断点，启动即可（无需再在root账户下重复start indigo，因为按下debug的时候已经开启了设备）。
3. 回到网页[192.168.124.12:7624](http://192.168.124.12:7624)上，重新按照前文内容配置INDIGO Server Manager，随后照常操作、调试。

