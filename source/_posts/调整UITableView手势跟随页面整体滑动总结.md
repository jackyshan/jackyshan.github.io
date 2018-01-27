---
title: 调整UITableView手势跟随页面整体滑动总结
date: 2017-11-02 21:48:52
tags: iOS
---

![img](http://upload-images.jianshu.io/upload_images/301129-15b431c6e5ed740f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

产品中经常出现手势滑动的需求，但是结合tableview的手势就会出现冲突，下面是一些tableview结合全局手势的实践方案。

![gif](http://upload-images.jianshu.io/upload_images/301129-26a6cd8d6ef98ce7.gif?imageMogr2/auto-orient/strip)

### 页面滑动

* 给页面添加滑动手势

        let pan = UIPanGestureRecognizer()
        pan.rx.event.subscribe(onNext: { [weak self] (recognizer: UIPanGestureRecognizer) in
            self?.panGestureStationTogetherAction(recognizer)
        }).addDisposableTo(disposeBag)
        lineListView?.stationHeaderDetailView.addGestureRecognizer(pan)


* 手势处理方式


    // MARK: 站点滑动手势统一处理
    func panGestureStationTogetherAction(_ recognizer: UIPanGestureRecognizer) {
        let wself = self
        guard let gestureView = wself.lineListView else {return}
        
        if recognizer.state == .began || recognizer.state == .changed {
            let movement = recognizer.translation(in: wself.view)
            var old_rect = gestureView.frame
            let listViewEmptyHeight = gestureView.getEmptyHeight()
            let stationDetailHeight: CGFloat = 68.0
            if recognizer.state == .began {
                old_rect.origin.y = old_rect.origin.y - listViewEmptyHeight - stationDetailHeight
            }
            else {
                old_rect.origin.y = old_rect.origin.y + movement.y
            }
            if old_rect.origin.y < 0 {
                gestureView.frame = wself.view.bounds
            }
            else if old_rect.origin.y > wself.view.bounds.height {
                gestureView.frame = CGRect.init(x: 0, y: wself.view.bounds.height, width: wself.view.bounds.width, height: wself.view.bounds.height)
            }
            else {
                gestureView.frame = old_rect
            }
            
            recognizer.setTranslation(CGPoint.zero, in: wself.view)
        }
        else if recognizer.state == .ended || recognizer.state == .cancelled {
            let halfPoint = wself.view.bounds.height*1/4
            if gestureView.frame.origin.y > halfPoint {
                gestureView.dismiss()
            }
            else {
                gestureView.show(superView: wself.view)
            }
        }
    }

滑动时候`.began || recognizer.state == .changed`是在滑动手势没有停止的时候，这个时候可以动态调整`ListView`的`y`值

        guard let gestureView = wself.lineListView else {return}

`lineListView `是被处理的view，并不一定是滑动的view，滑动view可以使任何一个view，把手势的变化统一传到这里就好了`panGestureStationTogetherAction `。

    let movement = recognizer.translation(in: wself.view)

获取手势滑动离原位置的`CGPoint`，这样我们就有了移动的x，y。

            if recognizer.state == .began {
                old_rect.origin.y = old_rect.origin.y - listViewEmptyHeight - stationDetailHeight
            }
            else {
                old_rect.origin.y = old_rect.origin.y + movement.y
            }

开始状态，初始化`ListView`的`frame`，`changed`状态，获取当前最新的frame的y加上最新的移动的y，`old_rect.origin.y = old_rect.origin.y + movement.y`

            recognizer.setTranslation(CGPoint.zero, in: wself.view)

改变frame之后，重置手势`CGPoint`为zero

        else if recognizer.state == .ended || recognizer.state == .cancelled {
            let halfPoint = wself.view.bounds.height*1/4
            if gestureView.frame.origin.y > halfPoint {
                gestureView.dismiss()
            }
            else {
                gestureView.show(superView: wself.view)
            }
        }

当手势结束或取消的时候，处理`ListView`的弹出或者下沉动画。

    func show(superView: UIView) {
        showBlock?()
        tableView.setContentOffset(CGPoint.zero, animated: false)
        UIView.animate(withDuration: 0.3) {
            self.frame = superView.bounds
        }
        
    }
    
    @objc func dismiss() {
        dismissBlock?()
        UIView.animate(withDuration: 0.3) {
            self.frame = CGRect.init(x: 0, y: self.bounds.height - 268, width: self.bounds.width, height: self.bounds.height)
        }
    }

### TableView滑动结合页面滑动处理

        //linelistview_tableview滑动
        lineListView?.tableView.panGestureRecognizer.rx.event.subscribe(onNext: { [weak self] (recognizer: UIPanGestureRecognizer) in
            guard let foffy = self?.lineListView?.frame.origin.y else {return}
            guard let offy = self?.lineListView?.tableView.contentOffset.y else {return}
            
            if foffy > CGFloat(0.0) || offy < CGFloat(0.0) {
                self?.panGestureStationTogetherAction(recognizer)
            }
            
        }).addDisposableTo(disposeBag)

`tableview`是继承`scrollview`完成的封装。scrollview自带`panGestureRecognizer`手势，所以可以直接获取scrollview手势的动态变化。由于封装的手势处理可以给任何滑动手势用，所以可以直接把tableview滑动手势直接传给统一手势处理方法。

            guard let foffy = self?.lineListView?.frame.origin.y else {return}
            foffy > CGFloat(0.0)

1、判断滑动的`ListView`的frame的y值是否已经超出规定范围，超出就不再处理滑动，我的实现是小于0就不再处理滑动。

            guard let offy = self?.lineListView?.tableView.contentOffset.y else {return}
           offy < CGFloat(0.0)

2、tableview的偏移量必须是小于0的这也才能处理滑动，不然会造成`tableview`和`listview`一起滑动的效果，很不自然。

__上面1、2条件必须放在一起使用，才能保证滑动手势和`view`本身的手势完美结合__

          tableView.setContentOffset(CGPoint.zero, animated: false)

滑动结束之后要记得把tableview的偏移量设为zero，这样可以解决快速滑动时出现的小bug。
