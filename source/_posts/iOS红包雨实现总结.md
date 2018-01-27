---
title: iOS红包雨实现总结
date: 2017-07-19 19:52:27
tags: CALayer
---

![image](http://upload-images.jianshu.io/upload_images/301129-d52e215a1dbafc60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 生成一个红包layer

        //创建画布
        let redPLayer = CALayer()
        redPLayer.bounds = CGRectMake(0, 0, 35, 46);
        redPLayer.anchorPoint = CGPointMake(0, 0)
        redPLayer.position = CGPointMake(0, -46)
        redPLayer.contents = UIImage(named: hbArr[Int(arc4random()%3)])!.CGImage
        self.bgView.layer.addSublayer(redPLayer)

### 添加layer的动画
        
        //画布动画
        //此处keyPath为CALayer的属性
        let  moveAnimation:CAKeyframeAnimation = CAKeyframeAnimation(keyPath:"position")
        //动画路线，一个数组里有多个轨迹点
        let screenW = UInt32(CGRectGetWidth(UIScreen.mainScreen().bounds))
        let screenH = CGRectGetHeight(UIScreen.mainScreen().bounds)
        moveAnimation.values = [NSValue(CGPoint: CGPointMake(CGFloat(Float(arc4random_uniform(screenW))), 0)),NSValue(CGPoint: CGPointMake(CGFloat(Float(arc4random_uniform(screenW))), screenH))]
        //动画间隔
        moveAnimation.duration = 5
        //重复次数
        moveAnimation.repeatCount = 1
        //动画的速度
        moveAnimation.timingFunction = CAMediaTimingFunction(name: kCAMediaTimingFunctionLinear)
        redPLayer.addAnimation(moveAnimation, forKey: "move")

### 定时生成红包

    public func start() {
        self.timer?.invalidate()
        self.timer = nil
        
        self.timer =  NSTimer.scheduledTimerWithTimeInterval(0.5, target: self, selector: #selector(self.showRain), userInfo: "", repeats: true)
    }

### 判断红包点击事件

    @objc private func clickRedPacket(tapgesture:UITapGestureRecognizer) -> Void {
        let touchPoint = tapgesture.locationInView(self.bgView)
        
        if let sublayers = self.bgView.layer.sublayers {
            for e in sublayers.enumerate() {
                if (e.element.presentationLayer()?.hitTest(touchPoint) != nil) {
                    print("点中了第\(e.index)个元素")
                    
                    let num = randomNums[Int(arc4random()%3)]
                    if e.index != 0 && e.index % num == 0 {//不等于0切能被整除
                        self.stop()
                        if delegate != nil {
                            delegate?.clickRedPacket(e.index)
                        }
                    }
                    
                    break
                }
            }
        }
    }

### 停止红包雨

    public func stop() {
        self.timer?.invalidate()
        //删除当前红包layers
        if let sublayers = self.bgView.layer.sublayers {
            for item in sublayers {
                item.removeAllAnimations()
                item.removeFromSuperlayer()
            }
        }
        
        self.dismiss()
    }

代码参考
https://github.com/jackyshan/swifticonrain