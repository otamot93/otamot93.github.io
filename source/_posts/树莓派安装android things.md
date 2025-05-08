---

title: 树莓派安装Android things

date: 2017-04-01 18:38:35

tags:
	- raspberry
	- 树莓派
    - Antroid things
    - 翻译

categories: Raspberry

---



## 树莓派3

树莓派3型号B是最新一代的流行的单板计算机。它拥有1.2GHz四核的64为处理器、USB 2.0接口、有线网和无线网、HDMI和复合视频输出、一个GPIO接口。

<!-- more -->

## 烧录镜像

在你准备烧录之前，你需要为你的树莓派准备好以下东西：
- HDMI线（不用也可以，并不一定需要看显示屏）
- 支持HDMI接口的显示屏（支持VGA接口的也可以，可以通过HDMI转VGA的线连接）
- USB线（供电）
- 网线（联网咯）
- MicroSD读卡器（将镜像烧录至SD卡）

为了可以将Android Things 烧录到你的树莓派种，请先下载[镜像](https://developer.android.com/things/preview/download.html "镜像下载地址")（选择Rashpberry Pi）并按照下列步骤操作:
1.将一张至少拥有8GB空间的microSD卡插入电脑（通过读卡器）
2.解压已经下载的镜像压缩包
3.按照树莓派官方的介绍将镜像写入SD卡种：

- [Linux](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md )

- [Mac](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md)

- [Windows](https://www.raspberrypi.org/documentation/installation/installing-images/windows.md)(使用 Win32DiskImage这个软件)

4.将microSD卡插入树莓派中(下图2号接口的上方)
5.连接树莓派
	1.USB线连到1号接口（供电啦）
	2.将网线插入2号接口
	3.将HDMI接口插入3号接口，并连接外部显示器

![接口](树莓派安装android things/raspberrypi-connections.png)

6.验证Android是否已经运行在设备上。在显示屏中会看到Android Things的一些信息，包括当前设备的ip地址
7.通过[adb tool](https://developer.android.com/studio/command-line/adb.html)连接ip地址（机器）(下载了android sdk后，在SDKPath的platform-tools中可以找到adb工具)
```
$ adb connect <ip-address>
connected to <ip-address>:5555
```

## 连接无线（有根网线很烦有木有）
使用adb工具连接无线
1.发送SSID(无线网名称)和passcode(密码)到WI-FI服务
```
$ adb shell am startservice \
    -n com.google.wifisetup/.WifiSetupService \
    -a WifiSetupService.Connect \
    -e ssid <Network_SSID> \
    -e passphrase <Network_Passcode>
```
> 如果你的无线网没有密码，去掉 `-e passphrase <Network_Passcode>`

2.通过logcat 命令验证连接是否成功
```
$ adb logcat -d | grep Wifi
...
V WifiWatcher: Network state changed to CONNECTED
V WifiWatcher: SSID changed: ...
I WifiConfigurator: Successfully connected to ...
```
3.验证可以访问远程ip地址
```
$ adb shell ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=57 time=6.67 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=57 time=55.5 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=57 time=23.0 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=57 time=245 ms
```

如果你想清除所有已经保存的网络，使用
```
$ adb shell am startservice \
    -n com.google.wifisetup/.WifiSetupService \
    -a WifiSetupService.Reset
```

现在可以拔掉网线，重启下树莓派，没有问题的话，显示屏中依然会显示当前设备的ip地址（通过无线访问的）


## 其他
在[原文](https://developer.android.com/things/hardware/raspberrypi.html)基础上翻译，并带了点注解![dog](树莓派安装android things/dog.gif)










