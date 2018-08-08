---
title: Android Studio 自动检查依赖库是否有新版本
tags: Android
categories: Android
abbrlink: f7a3c6c0
date: 2017-09-12 15:45:32
---

**Preferences** -> **Editor** -> **Inspections** -> “**Android > Lint > Correctness**” -> **Newer Library Versions Available (available for Analyze|Inspect Code)**    <font color="red">**这一项打上勾**</font>。
![](/media/15051956933872/15051957572874.png)

如果想要检查的话，同时按住 **shift+alt+command+i** ，会弹出 **Enter Inspect name** 窗口。输入 **Newer Library Versions Available** ，一般不用输入全部的，输入 **Newer** 就好了，包含 **Newer** 的选项就会全部搜索出来。
![](/media/15051956933872/15051957698386.png)

选择 **Newer Library Versions Available** 这一项
![](/media/15051956933872/15051957934804.png)

选择你需要检查的模块

* **Whole project**  ->完整的模块。 
* **Module** -> 单独的一个模块
* **File** -> 单独的一个文件
* **Custom scope** -> 自定义范围

我这里选择 **Whole project** ，点击 **OK** 就可以开始检查了。
![](/media/15051956933872/15051958715001.jpg)

点击 **Update to [VersionName]** 就可以把版本号替换到最新版本。
最后点击 **Sync Now** 同步一下项目
![](/media/15051956933872/15051961312946.jpg)

需要注意升级版本后，新版本可能不兼容旧版本，需要仔细检查代码，防止出错。







