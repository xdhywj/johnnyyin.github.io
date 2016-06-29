---
layout: post
title: Android模拟后台进程被杀
date: 2016-06-29 20:02:50
categories: [Android]
tags: [Android]
---
Android开发中，有时候我们需要测试下后台进程被杀，然后重新进入App时恢复现场的case。如果采用填充内存的方式，比较麻烦，下面介绍几种快速模拟后台进程被杀的方式：
<!-- more -->
### 方式1：
最简单的方法是在DDMS中点击”Stop Porcess”杀掉你的程序，在你调试程序的时候可以这样做。

### 方式2：
适合debug程序
Android Studio中打开Android Monitor，选择进程，将app按home键退到后台，点击terminate application按钮即可

### 方式3：
adb shell am kill package-name 杀掉后台程序，同样需要先将目标程序按home进入后台, 然后再执行此命令
adb shell am kill-all 杀掉所有后台程序，需要先将目标程序按home进入后台，然后打开一个其他程序

### 方式4：
适合所有程序
打开手机开发者选项-后台进程限制-不允许后台进程，同样按home键退到后台后，打开个其他应用再退出，进程就被杀了。

### 方式5：
通过模拟器或者一个Root过的真机：
1. 按Home按键退出你的程序；
2. 在控制台，敲入如下命令（Windows系统下 WIN + R -> cmd -> 回车）
> 找到该APP的进程ID adb shell ps 
> 找到你APP的包名 
> Mac/Unix: save some time by using grep: adb shell ps | grep your.app.package
> 按照上述命令操作后，看起来是这样子的: 
> USER      PID  PPID  VSIZE  RSS    WCHAN    PC        NAME # u0_a198  21997 160  827940 22064 ffffffff 00000000 S your.app.package
> 通过PID将你的APP杀掉 adb shell kill -9 21997 