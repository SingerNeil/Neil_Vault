

[B站视频教程](https://www.bilibili.com/video/BV1fZCyBYEuT/?spm_id_from=333.1387.favlist.content.click&vd_source=4a26dae16046c52dd00442733da50418)

---
# 背景

***Obsidian***是本人目前用过最流畅的免费笔记软件，它拥有十分强大的笔记间连接功能和海量的第三方插件库，十分适合个人用户使用。
同时其笔记均以**库的形式**存储在本地，没有对运营方跑路和个人隐私泄露的担忧（反观一些云笔记软件），且在Windows、MacOS、Linux各大发行版、Android、IOS上均有稳定可靠的版本可供使用。

要说其唯一不足便是没有同样免费的笔记云同步服务，***Obsidian***自带的云同步笔记服务需要充值会员才可使用。
但众所周知，GitHub现如今已然成为计算机科学相关产业的基础设施之一。于是本篇笔记，受B站up主**技术爬爬虾**启发，为解决前文提到的困难而给出一种方案——**提供一个基于GitHub的Obsidian云同步方案**。

---
# 下载Obsidian
- 前往[Obsidian官网下载页面](https://obsidian.md/download)下载符合自己操作系统的版本。

---
# 在GitHub上为笔记库建立repository
1. 登录自己的[GitHub](https://github.com/dashboard)账户
2. 创建一个新的repository
![](assets/基于Github的Obsidian库免费同步方案/file-20260105231442728.png)
	如上图所示输入相应内容，同时若你不希望其他人访问你的库中的内容，需要在`Choose visibility`中选择`Private`。

---
# 将GitHub上创建好的库clone到本地
**PS：若是不会使用命令行可自行询问`LLM`或者使用`GitHub Desktop`，这里仅展示命令行操作方案。**
1. 打开terminal后，先自行使用`cd`命令抵达到你想存放笔记库的根目录下。
2. 复制GitHub上，笔记repository的`HTTPS`地址
![](assets/基于Github的Obsidian库免费同步方案/file-20260105232641088.png)


3. 在终端内输入以下内容，从而将repository复制到本地（尽管此时的repository还是空库）
```bash
 git clone https://github.com/你的用户名/你的repository名.git
```









---
