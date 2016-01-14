---
layout: post
title: "Android开发从新建到发布整个流程"
date: 2016-01-13 19:01:52 +0800
comments: true
categories: Android
---

##环境

 >系统：OS X EI Capitan
 
 >java ：1.8.0_25

##工具
`Android Studio下载`

	http://developer.android.com/sdk/index.html

##安装Android Studio

* 是否有工程已有配置文件，选择无，下一步

![](/images/accept.jpg)

* Unable to access Android SDK add-on list

![](/images/add-on list.jpg)

`解决方法`

	sudo vi /Applications/Android\ Studio.app/Contents/bin/idea.properties
.
		
	#
	# *DO NOT* modify this file directly. If there is a value that you would like to override,
	# please add it to your user specific configuration file.
	#
	# See http://tools.android.com/tech-docs/configuration
	#
	apple.laf.useScreenMenuBar=true
	apple.awt.fullscreencapturealldisplays=false
	apple.awt.graphics.UseQuartz=true
	CVS_PASSFILE=~/.cvspass
	idea.xdebug.key=-Xdebug
	java.endorsed.dirs=
	idea.smooth.progress=false
	idea.fatal.error.notification=disabled
	com.apple.mrj.application.live-resize=false
	JVMVersion=1.6*
	disable.android.first.run=true

点击取消，进行下一步

* Welcome Android Studio Setup Wizard，下一步

![](/images/Welcome.jpg)

* Install Type，下一步

![](/images/Install Type.jpg)

* Verify Settings，完成

![](/images/Verify Settings.jpg)

* Downloading android-sdk-r22-macosx.zip, connect timed out，点击取消，等下解决

![](/images/Downloading android-sdk-r22.jpg)

##新建项目
* 点击Start a new Android Studio project，报错

![](/images/new Android Studio project.jpg)

`解决方法`

打开网页[http://www.androiddevtools.cn/](http://www.androiddevtools.cn/) 下载android-sdk_r24.3.4-macosx.zip (SDK Tools)

![](/images/android-sdk_r24.3.4.jpg)


.

	点击Configure-->Project Defaults-->Project Structure-->Android SDK location
	
![](/images/Android SDK location.jpg)

* 点击Start a new Android Studio project，New Project，下一步

![](/images/New Project.jpg)

* Target Android Devices，`选择API 21`，下一步

![](/images/API 21.jpg)

* Installing Android SDK, Loading SDK information... There is nothing to install or update. 

![](/images/Loading SDK information.jpg)

`解决方法`

打开网页[http://www.androiddevtools.cn/](http://www.androiddevtools.cn/) 下载 `android 5.0`	 (SDK)

![](/images/android 5.0.jpg)

	解压放到/Users/jackyshan/android-sdk-macosx/platforms目录下
	
打开网页[http://www.androiddevtools.cn/](http://www.androiddevtools.cn/) 下载`Build-Tools 21.1.2`	 (Build-Tools)

![](/images/Build-Tools 21.1.2.jpg)
	
	解压放到/Users/jackyshan/android-sdk-macosx/build-tools目录下

打开网页[http://www.androiddevtools.cn/](http://www.androiddevtools.cn/) 下载`Android SDK Extras 21.0.3` (Android SDK Extras)

![](/images/Android SDK Extras 21.0.3.jpg)

	解压放到/Users/jackyshan/android-sdk-macosx/extras目录下


重新点击Start a new Android Studio project，下一步，下一步

* Add an activity to Mobile，下一步

![](/images/Add an activity to Mobile.jpg)

* Customize the Activity，完成

![](/images/Customize the Activity.jpg)

* gradle project download，等待gradle下载完`不要点击取消,有点慢`

![](/images/gradle project download.jpg)

	
* 下载完，进入工程

![](/images/main project home.jpg)

* 运行报错，Error running app: Unable to obtain debug bridge

![](/images/Unable to obtain debug bridge.jpg)

`解决方法`

打开网页[http://www.androiddevtools.cn/](http://www.androiddevtools.cn/) 下载 `platform-tools-r22`	 (SDK Platform-Tools)

![](/images/platform-tools-r22.jpg)

	解压放到/Users/jackyshan/android-sdk-macosx目录下
	
* 运行没有问题，弹出设备选择

![](/images/choise devices.jpg)


* 连接android手机调试，报错minSdkVersion大于当前android设备(API16)

`解决方法`

>打开build.gradle文件，更改minSdkVersion 16

	apply plugin: 'com.android.application'
	
	android {
	    compileSdkVersion 21
	    buildToolsVersion "22.0.0"
	
	    defaultConfig {
	        applicationId "io.gitcafe.jackyshan.myapplication"
	        minSdkVersion 16
	        targetSdkVersion 21
	        versionCode 1
	        versionName "1.0"
	    }
	    buildTypes {
	        release {
	            minifyEnabled false
	            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
	        }
	    }
	}
	
	dependencies {
	    compile fileTree(dir: 'libs', include: ['*.jar'])
	    testCompile 'junit:junit:4.12'
	    compile 'com.android.support:appcompat-v7:21.0.3'
	}



* sdk目录截图

![](/images/androidsdklist.gif)

* 我的sdk目录打包下载`如果自己怎么都弄不好，可以去百度云下载弄好的sdk`

>链接: [http://pan.baidu.com/s/1pKustcv 密码: zhcy](http://pan.baidu.com/s/1pKustcv密码zhcy) 密码: zhcy



##持续更新中...
	
	




