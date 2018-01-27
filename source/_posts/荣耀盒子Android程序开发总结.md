---
title: 荣耀盒子Android程序开发总结
date: 2017-07-19 19:54:56
tags: 盒子
---

![image](http://upload-images.jianshu.io/upload_images/301129-52ab3adbcd517f7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 当前环境
荣耀盒子：Android4.4
开发工具：Android Studio 2.3 
__使用2.3新建一个项目，装到盒子上一直闪退，后来使用之前2.1新建的项目打开编译装到盒子上就运行成功了__

### 配置gradle
buildToolsVersion "22.0.0"，之前设置“25.0.0”，盒子一直打开闪退

    apply plugin: 'com.android.application'

    android {
        compileSdkVersion 21
        buildToolsVersion "22.0.0"

        defaultConfig {
            applicationId "io.xx.xx.myapplication"
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
        compile fileTree(include: ['*.jar'], dir: 'libs')
        testCompile 'junit:junit:4.12'
        compile 'com.android.support:appcompat-v7:21.0.3'
    }

### 上传apk
在盒子应用界面搜索小白文件管理器，打开里面的ftp服务

    ftp 192.168.1.105 2121
    put /Users/xxx/app_debug.apk /app.apk

打开小白安装包，安装apk

[这里](https://github.com/jackyshan/macOSVideoList_Android)有自己开发的可以安装到荣耀盒子的Android程序，仅供参考。