---
layout: article
title: 'adb 实时输出logcat日志到指定文件'
tags: code adb shell
---

# # adb 实时输出logcat日志到指定文件


## 应用场景

  adb 实时输出logcat日志到指定文件

## adb命令

0. adb shell  logcat -v time > C:\Users\Administrator\Desktop\logcat.txt


```logcat
adb shell ps -A | grep launcher
adb shell ps -A | grep systemdia
adb logcat | grep 1071
```
1.adb install +包名       adb安装apk (覆盖安装是使用 -r 选项)

2.adb uninstall +包名      adb卸载apk

3.adb connect +设备IP      网络连接Android设备

4.adb reboot       重启Android设备

5.adb devices      获取连接的设备列表及设备状态

6.adb get-state    获取设备的状态 (设备的状态有 3 钟，device:设备正常连接 , offline:连接出现异常，设备无响应 , unknown:没有连接设备)

7.查看运行在 Android设备上的 adb 后台进程：

执行 adb shell ps | grep adbd ，可以找到该后台进程，windows 请使用 findstr 替代 grep


## adb shell 命令

adb 命令是 adb 这个程序自带的一些命令，而 adb shell 则是调用的 Android 系统中的命令，这些 Android 特有的命令都放在了 Android 设备的 system/bin 目录下

### 1.[SHELL] adb shell  bugreport , 打印dumpsys、dumpstate、logcat的输出，也是用于分析错误

- 输出比较多，建议重定向到一个文件中

- adb shell dumpsys > d:\bugreport.log

### 2.[PM]Package Manager , 可以用获取到一些安装在 Android 设备上得应用信息

-   adb shell pm list package      列出所有的应用的包名 （-s：列出系统应用  -3：列出第三方应用 -f：列出应用包名及对应的apk名及存放位置  -i：列出应用包名及其安装来源）

-  adb shell pm path+包名     列出对应包名.apk 位置

-  adb shell pm install +apk存放路径   安装应用（目标 apk 存放于PC端，用 adb install 安装   目标 apk 存放于Android设备上，用 pm install 安装） 

### 3.[AM]adb shell  am start +包名/.Activity (要启动的Activity)     启动一个 Activity （-s先停止目标应用，再启动  -w 等待应用完成启动  -a 启动默认浏览器打开一个网页例：adb shell am start -a android.intent.action.VIEW -d http://testerhome.com）

- 1  adb shell am monitor        监控 crash 与 ANR

- 2  adb shell am force-stop    后跟包名，结束应用

- 3 adb shell am startservice    启动一个服务

- 4 adb shell am broadcast       发送一个广播


### 4.INPUT 这个命令可以向 Android 设备发送按键事件

- adb shell input text +具体内容    发送文本内容，不能发送中文 

- adb shell input keyevent + 按键事件   发送按键事件 例如：adb shell input keyevent KEYCODE_HOME 模拟按下Home键

- adb shell input tap +触摸事件的位置 , 对屏幕发送一个触摸事件 例如：点击屏幕上坐标为 500 500 的位置（adb shell input tap 500 500)

- adb shell input tap , 对屏幕发送一个触摸事件

- adb shell input swipe   滑动事件  例如：从右往左滑动屏幕 

~~~
adb shell input swipe 800 600 100 600
~~~


### **5.screencap**

截图命令

```
adb shell screencap -p /sdcard/DCIM/screenTest.png
```
录制命令
```
adb shell screenrecord /sdcard/demo.mp4
```
输入法
```
adb shell ime list -s
```

### 重要系统信息获取

1.获取系统版本

adb shell getprop ro.build.version.release

2.获取系统api版本

adb shell getprop ro.build.version.sdk

3.获取手机相关制造商信息

    adb shell getprop | grep "model\|version.sdk\|manufacture
    r\|hardware\|platform\|revision\|serialno\|product.name\|brand"

3,获取手机系统信息（ CPU，厂商名称等）

adb shell "cat /system/build.prop | grep "product""

4,获取手机设备型号

adb -d shell getprop ro.product.model

5，获取手机厂商名称

adb -d shell getprop ro.product.brand

6，获取手机的序列号

    有两种方式
    1，adb get-serialno
    2，adb shell getprop ro.serialno

7，获取手机MAC地址

adb shell cat /sys/class/net/wlan0/address

8，获取手机内存信息

adb shell cat /proc/meminfo

9，获取手机存储信息

adb shell df

10，获取手机内部存储信息

adb shell df /data

11，获取Android设备屏幕分辨率

adb shell "dumpsys window | grep mUnrestrictedScreen"

12，连接多个设备对其中一个进行操作
//以adb shell 为例
adb -s 192.168.101.37:5555 shell

13，查看运行进程

adb shell procrank

14，关闭或杀掉进程

adb shell kill 366

15，保留数据和缓存文件，重新安装，升级

adb install -r test.apk

16，卸载app但保留数据和缓存文件

adb uninstall -k cnblogs.apk

17，查看目录下的文件大小

adb shell du -sh *

18，查看正在运行的Services

adb shell dumpsys activity services [<packagename>]

19，查看正在运行的Activity

adb shell dumpsys activity [<packagename>]

adb shell dumpsys activity containers  > c:\containers.txt

20,clear 清除应用数据

adb shell pm clear com.baidu

21，cp复制文件

adb shell 进入Android Linux命令中

cp -f system/app/Music/Music.apk /sdcard/Music.apk

22，删除命令

adb shell 进入Android Linux命令中

rm  -r  /mnt/sdcard/a.mp3 

删除文件夹的时候需要加上-r参数 

cd dir 
rm *    删除dir中所有文件

23，重启进入recovery模式

adb reboot recovery

24，cat查看文件

cat  /sdcard/test.txt

25，查看指定进程PID

ps +  进程的包名

26，查看进程具体的信息

例如：1460是要查看的进程的PID
cat /proc/1460/maps    查看进程的文件结构 
cat /proc/1460/status   查看进程的状态

27，findstr 和 grep过滤搜索

1）cmd下搜索包名为com.android.launcher3的进程 
adb shell ps|findstr /i “com.android.launcher3” 

2）shell下面搜索 
先使用adb shell进去，然后使用grep命令过滤 
ps | grep “com.linux.test”







