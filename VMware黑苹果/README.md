### 准备软件

1. Windows 10 专业版，版本号：`1909`
2. [VMware Workstation Pro 15.1.0](https://download3.vmware.com/software/wkst/file/VMware-workstation-full-15.1.0-13591040.exe)，激活码：`UY758-0RXEQ-M81WP-8ZM7Z-Y3HDA`
3. [unlocker](https://github.com/theJaxon/unlocker)，版本：`3.0.2`
4. [com.vmware.fusion.tools.darwin.zip.tar](http://softwareupdate.vmware.com/cds/vmw-desktop/fusion/11.0.2/10952296/packages/com.vmware.fusion.tools.darwin.zip.tar)，版本：`11.0.2`
5. [com.vmware.fusion.tools.darwinPre15.zip.tar](http://softwareupdate.vmware.com/cds/vmw-desktop/fusion/11.0.2/10952296/packages/com.vmware.fusion.tools.darwinPre15.zip.tar)，版本：`11.0.2`
6. [Install macOS Mojave](https://pan.baidu.com/s/1P1evQj3fNEJJSYOvBUoH_g)，提取码：`p5ku`，版本：`10.14.3`，来源于参考资料[1]

### 安装过程

1. 安装 `VMware` 并激活
2. 解压 `unlocker` 到 `D:\unlocker`，并创建 `tools` 目录
3. 将 `com.vmware.fusion.tools.darwin.zip.tar` 文件中的 `darwin.iso` 和 `darwin.iso.sig` 解压到 `tools` 目录
4. 将 `com.vmware.fusion.tools.darwinPre15.zip.tar` 文件中的 `darwinPre15.iso` 和 `darwinPre15.iso.sig` 解压到 `tools` 目录
5. 将 `D:\unlocker\win-install.cmd` 文件第 18 行修改为 `set KeyName="HKLM\SOFTWARE\Wow6432Node\VMware, Inc.\VMware Workstation"`
6. 将 `D:\unlocker\win-install.cmd` 文件第 51 行 `gettools.exe` 删除，因这一工作已经在第 3 步和第 4 步完成
7. 以管理员身份运行 `win-install.cmd`
8. 安装参考资料[1]和[2]安装 `Mojave`

在安装 Mojave 的过程中可能会遇到[这个安装macOS Mojave 应用程序副本已损坏，不能用来安装Mac OS](https://zhuanlan.zhihu.com/p/88597219)，此时需要

1. 断开网络连接
2. 在`实用工具-终端`执行：`date 010100002016.10`

### 其他事项

因为 VMware 更新了而 unlocker 没有更新，所以上面是人工完成了 `gettools.exe` 的工作。

`unlocker` 变迁史

1. `unlocker` 的原作者 [DrDonk](https://github.com/DrDonk) 删除了仓库
2. 用户 [theJaxon](https://github.com/theJaxon) fork 了一个版本的 [unlocker](https://github.com/theJaxon/unlocker)，但是不再维护
3. [BDisp](https://github.com/BDisp) 在 [theJaxon](https://github.com/theJaxon) 的基础上维护了 3.0.3 版本的 [unlocker](https://github.com/BDisp/unlocker)

因此似乎可以使用 [BDisp 的版本](https://github.com/BDisp/unlocker) 来完成 unlocker 的安装，但是还没有试验过，可以阅读参考资料[9]获得更多信息。

### 参考资料

1. [[安装实录]如何在 Vmware虚拟机中安装 macOS Mojave -- Windows 版](https://zhuanlan.zhihu.com/p/59412199)
2. [VMware 安装 macOS Mojave，以及制作安装镜像](https://jingyan.baidu.com/article/90bc8fc8a0645ef653640cfd.html)
3. [升级VMware至15.1.0版本解决Windows 10 1903下VMware Workstation 15 Pro虚拟机死机问题](https://blog.csdn.net/discoverer100/article/details/94932856)
4. [威睿虚拟机 VMware Workstation Pro 15.1.0 中文版 + 注册机](https://www.cnblogs.com/baojun/p/11184149.html)
5. [虚拟机 VMware Workstation Pro 15.5.0 及永久激活密钥](https://www.cnblogs.com/zero-vic/p/11584437.html)
6. [黑苹果 安装系统出现"安装 macOS xxx"应用程序副本已损坏，不能用来安装macOS解决方法](https://blog.csdn.net/qq_41855420/article/details/102762647)
7. [Windows下VMware Workstations Pro15.5.0安装dmg镜像（macOS Catalina 10.15虚拟机）](https://hestyle.blog.csdn.net/article/details/104672651)
8. [Windows下VMware Workstations Pro15.1.0安装macOS Mojave10.14.5虚拟机](https://blog.csdn.net/qq_41855420/article/details/100086669)
9. [Windows1903安装VMware Workstations Pro15.1.0并解锁Unlock3.0.2](https://blog.csdn.net/qq_41855420/article/details/100082441)