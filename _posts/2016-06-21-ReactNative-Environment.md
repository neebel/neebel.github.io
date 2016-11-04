---
layout: post
title: React Native开发环境搭建
categories: ReactNative 
date: 2016-06-21 14:33:24
pid: 20160621-143324
---

**Part 1. Mac上iOS环境配置**

*1.安装 [Homebrew](http://www.brew.sh)*

在终端执行以下命令：

>/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

*2.用 Homebrew 安装 [Node.js](https://nodejs.org)*

在终端执行以下命令：(React Native需要NodeJS 4.0或更高版本)

>brew install node

*3.用 npm 安装 react-native-cli ( React Native的命令行工具 )*

在终端执行以下命令：

>npm install -g react-native-cli

如果没有翻墙，可以先为npm配置下淘宝镜像源，再执行以上命令：

>npm config delete registry 
 npm config delete disturl  
 npm config set registry https://registry.npm.taobao.org 
 npm config set disturl https://npm.taobao.org/dist 

*4.推荐安装 [watchman](https://facebook.github.io/watchman/docs/install.html)和 [flow](https://www.flowtype.org)*

>brew install watchman
 brew install flow

至此，iOS环境配置已经OK，试着初始化我们的第一个React Native项目，在某个文件夹下执行命令：

>react-native init HelloWord

可以看到以下文件被创建：
![create-success](/img/reactnative-create.png)

运行ios目录下的工程文件，大功告成：
![run-success](/img/reactnative-success.png)

**Part 2. Mac上Android环境配置**

*1.安装最新版 [JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)，用dmg包直接安装就行*

*2.用 Homebrew 安装 Android SDK*

>brew install android-sdk

*3.配置 ANDROID_HOME 环境变量*

（1）编辑.bashrc文件

>sudo vi ~/.bashrc

（2）在.bashrc文件中加入下面的配置
	
>export ANDROID_HOME=/usr/local/opt/android-sdk

*4.设置 SDK*

在终端输入 android ，打开 Android SDK Manager，需要安装以下项目：
![reactnative-sdk](/img/reactnative-sdk.png)

两点提示：

Android Support Repository 或者 Local Maven repository for Support Libraries 一样的

Android SDK Build-tools version 23.0.1（必须是这个版本）

*5.安装 Genymotion*

 (1) 下载并安装 [Genymotion](https://www.genymotion.com)

 (2) 打开 Genymotion，如果尚未安装VirtualBox，需要先安装之

 (3) 创建一个模拟器（需要注册账号）

至此，Android 环境配置已经OK。创建好 HelloWorld 工程之后，运行项目：

>cd HelloWorld
 react-native run-android

截图：
![reactnative-android](/img/reactnative-android.png)

**Tips**: 

1.第一次运行 react-native run-android 命令时，需要等待很长时间，是因为在下载gradle，可以翻墙解决

2.

>09:56:47 E/adb: error: could not install *smartsocket* listener: Address already in use
 09:56:47 E/adb: ADB server didn't ACK
 09:56:47 E/adb: * failed to start daemon *
 09:56:47 E/ddms: '/usr/local/opt/android-sdk/platform-tools/adb,start-server' failed -- run manually if necessary
 09:56:47 E/adb: error: cannot connect to daemon
 :app:installDebug FAILED

解决办法是配置 Genymotion-》Settings-》ADB 为自己的SDK路径:
![reactnative-problem](/img/reactnative-problem.png)
