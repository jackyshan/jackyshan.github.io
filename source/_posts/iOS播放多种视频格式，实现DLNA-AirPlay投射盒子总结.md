---
title: iOS播放多种视频格式，实现DLNA|AirPlay投射盒子总结
date: 2017-11-02 21:44:09
tags: DLNA
---

![img](http://upload-images.jianshu.io/upload_images/301129-610ae05ab3062a0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 视频播放
VLC media player is a free and open-source software, a portable and cross-platform media player and streaming media server written by the VideoLAN project. [Wikipedia](https://en.wikipedia.org/wiki/VLC_media_player)

`MobileVLCKit.framework `这是VLC开源框架，基于这个框架，我对自己视频资源做了一个web服务器，可以进行资源浏览，这样可以通过webview进行浏览视频资源。通过js调用可以输出webview链接，并传给VLC进行播放。

支持格式
`mp4、avi、mkv、3gp、rmvb、wmv、mpg、flv、swf`

### DLNA
Digital Living Network Alliance was founded by a group of consumer electronics companies in June 2003 to develop and promote a set of interoperability guidelines for sharing digital media among multimedia ...[Wikipedia](https://en.wikipedia.org/wiki/Digital_Living_Network_Alliance)

家里的电子盒子基本上都支持DLNA协议，使用它可以把视频链接投射进行播放。
我使用了第三方框架`Neptune.framework`和`Platinum.framework`，完成了这方面的开发。

### 手机演示
![gif](https://github.com/jackyshan/iosremoteplaydlna/raw/master/test.gif)

![电视](https://github.com/jackyshan/iosremoteplaydlna/raw/master/test.jpg)

`代码参考`
https://github.com/jackyshan/iosremoteplaydlna