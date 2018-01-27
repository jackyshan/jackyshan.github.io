---
title: iOS设置UIView阴影遇到的一些坑
date: 2017-11-02 21:49:31
tags: iOS
---

![img](http://upload-images.jianshu.io/upload_images/301129-b7d0fc235fa57a79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

目的是为了给这块view下半部分加上阴影，实现代码如下。

        topView.layer.masksToBounds = false
        topView.layer.shadowOffset = CGSize.init(width: 0, height: 3)
        topView.layer.shadowOpacity = 0.3
        topView.layer.shadowRadius = 3
        topView.layer.shadowColor = ViewUitl.colorWithHexString(hex: "#6691FB").cgColor
        topView.layer.cornerRadius = 5
        topView.layer.borderWidth = 1
        topView.layer.borderColor = UIColor.white.cgColor

### 1坑

`masksToBounds `默认为false，也许项目中加了默认为true的效果。true的情况会导致阴影效果一直不会出来。
`clipsToBounds`默认也是false，最好也设置一下false，防止不出阴影效果。

### 2坑

`shadowOffset `是`CGSize `实现的，实际功能是偏移量。width是整个阴影x偏移几个像素，height是整个阴影y偏移几个像素。

这个属性要配合`shadowRadius `使用，比如我半径Radius设置是3，我想实现下半部分显示阴影，我要设置`shadowOffset`的height为3，这样上部分的阴影向下偏移3个像素，上半部分的阴影就看不到了。（如果height设置为-3的话，就是下半部分隐藏了，向上移动了3个像素）

### 解释

`masksToBounds `layer对子layer进行切割，为true后切割后，阴影就看不到了。
`shadowOffset `layer阴影的偏移量设置。
`shadowOpacity `阴影的不透明度。
`shadowRadius `阴影的半径。
`shadowColor `阴影的颜色，会随着不透明度变。
`cornerRadius `view的圆角弧度。
`borderWidth `view的边线宽度。
`borderColor `view的边线颜色。