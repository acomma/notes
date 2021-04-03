### 准备硬件

1. NUC8i5BEH
2. 金士顿(Kingston) DDR4 2400 16GB 笔记本内存条，型号：`KVR24S17D8/16-SP`
3. 西部数据（Western Digital）500GB SSD固态硬盘 M.2接口（NVMe协议）WD Blue SN550，型号：`WDS500G2B0C`
4. AOC 27英寸 4K高清，型号：`U2790PQU`
5. 金士顿（Kingston）32GB USB3.0 U盘，型号：`DT100G3/32G`

### 准备软件

1. balenaEtcher：链接: `https://pan.baidu.com/s/1rnGRqkU3xEBj8eQ_MkudAQ`，提取码: `2fuq`
2. Catalina 10.15.3：链接: `https://pan.baidu.com/s/1OfcMvk7WY0tNc5Qfur7Q5Q` 提取码: `y9ug`
3. BE0077.bio：[Gitee](https://gitee.com/wangdudyb/nuc8i5beh)，[GitHub](https://github.com/dongyubin/nuc8i5beh)

### 准备教程

1. [NUC8黑果安装教程](https://www.bilibili.com/video/BV1bE411G7hj)，上面两个软件均来源于该视频教程作者：[维客数码](https://space.bilibili.com/443044257)
2. [指南：nuc8i5beh 安装黑苹果的教程，接近完美运行](https://chengxuxiaohei.cn/mac-anzhuang.html)

### 安装过程

1. 按照教程 2 更新 `BIOS` 版本为 `0077`
2. 按照教程 2 设置 `BIOS`
    * « Intel VT for directed I/VO (VT-d) » ： `disabled`
    * « Secure Boot » ： `disabled`
    * « Legacy Boot » ：`enabled`
    * « Fast Boot » ： `disabled`
    * Boot->Boot Devices-> « USB » ： `enabled`
    * SATA mode ： `AHCI`
    * Boot->Boot Configuration-> « Boot Network Devices Last » ： `disabled`
    * Power->Secondary Power Settings, « Wake on LAN from S4/S5 », set to « `Stay  Off` »
    * Devices->Video, « IGD Minimum Memory » set to `64mb`
    * Devices->Video, « IGD Aperture Size » set to `256mb`
3. 按照教程 1 一步一步安装

### 遇到的问题

[这个“安装macOS Catalina”应用程序副本已损坏，不能用来安装macOS？](https://www.zhihu.com/question/370370265)

1. 从[这条新闻](https://www.ithome.com/0/470/431.htm)得到 `Catalina 10.15.3` 的发布时间为：`2020-01-29`
2. 按照[天空的回答](https://www.zhihu.com/question/370370265/answer/1170241777)操作，先断网，然后`实用工具-终端`，输入：`date 010100002020`