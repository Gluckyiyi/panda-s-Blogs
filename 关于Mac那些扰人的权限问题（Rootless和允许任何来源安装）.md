# 关于Mac那些扰人的权限问题（Rootless和允许任何来源安装）

## 前言

前段时间电脑升级了Mojave，被其切换界面，输入法找不到焦点的bug折磨的死去活来，后来实在难以忍受只得装回了10.13，在安装node的时候却出现了问题，npm全局装包的时候，权限问题，权限问题，简直让人疯狂。

## 原因

究其原因，罪魁祸首就是OSX10.11之后推出的Rootless机制，一个内核保护措施，系统默认会锁定`/system`、`/sbin`、`/usr`这三个目录，带来的后果就是，当你安装一些需要操作这三个目录的包的时候就会出现**Operation not permitted**之类的错误，即使你`sudo`授权依然无效。

## 处理Rootless

### 1、查看是否开启了Rootless

在终端输入如下命令即可

`csrutil status `

如果显示的是`System Integrity Protection status: disabled. `则表示没有开启。

如果显示的是`System Integrity Protection status: enabled. `则表示已开启。

### 2、关闭Rootless

重启Mac，在听到经典的启动声后，按下`command+R`进入恢复模式，在菜单栏中**实用工具**找到**终端(Terminal)**，输入如下指令

`csrutil disable; reboot`

电脑会重新启动，进入系统后可以使用1的方法查看是否关闭成功。

### 3、开启Rootless

方式和2一致，只是指令换成如下:

``csrutil enable; reboot``

## 最后

到上面关于Rootless的处理就结束了，最后再附加一个有朋友遇到的安装一些破解软件无法安装的问题，这是因为在安全性与隐私里面，现在的mac系统（10.12之后）默认没有了允许任何来源，想要打开的话，只需在终端输入如下命令：

`sudo spctl --master-disable`

当然如果为了安全考虑，你也可以在安装完了你需要的软件之后，重新关闭掉，命令如下:

`sudo spctl --master-enable`.