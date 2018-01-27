---
title: RxSwift基础UI绑定实战总结
date: 2017-07-19 19:56:03
tags: RxSwift
---

![image](http://upload-images.jianshu.io/upload_images/301129-8b5dd2b06f7ef948.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 项目案列
案例是用户反馈界面，当用户点击推荐列表的btn或者在输入框输入内容的时候，提交反馈的按钮的isEnabled状态实时更新，使用Swift3代码

![gif](http://upload-images.jianshu.io/upload_images/301129-a7469f6e5612b393.gif?imageMogr2/auto-orient/strip)

### 实时更新被点中的数组状态
初始化listCount变量，该变量代表了当前推荐列表的btn数量是否大于0，初始化checkedList，代表被选中的btn数量
 
    let listCount: Variable<Bool> = Variable(false)
    var checkedList: [DriverFeedbackModel] = [DriverFeedbackModel]()

实现点击btn的方法，通过判断btn的isSelected状态，checkedList增删btn代表的model数据，listCount的值根据checkedList的数量进行赋值
> checkedList.append(list[sender.tag])
> checkedList = checkedList.filter({$0 !== list[sender.tag]})//实现数组的remove效果

    func clickRecommandBtn(_ sender: UIButton) {
        if sender.isSelected == false {
            sender.layer.borderColor = AppConfig.XXT_Green.cgColor
            sender.isSelected = true
            
            if let list = listData {
                checkedList.append(list[sender.tag])
            }
        }
        else {
            sender.layer.borderColor = AppConfig.XXT_LightGray.cgColor
            sender.isSelected = false
            if let list = listData {
                checkedList = checkedList.filter({$0 !== list[sender.tag]})//实现数组的remove效果
            }
        }
        
        listCount.value = !checkedList.isEmpty
    }

listCount的值根据选中的btn数量实时变化
> listCount.value = !checkedList.isEmpty

### 联合textView.rx.text信号和listCount信号，绑定到提交按钮的isEnabled状态
在方法体内返回bool值
>{ ($0 == true || !$1.isEmpty) && $1.characters.count < 151 }

绑定到submitBtn的rx.isEnabled，根据方法体内返回bool值实时修改isEnabled值
>bind(to: submitBtn.rx.isEnabled)

      Observable.combineLatest(listCount.asObservable(), textView.rx.text.orEmpty.asObservable()){ ($0 == true || !$1.isEmpty) && $1.characters.count < 151 }.bind(to: submitBtn.rx.isEnabled).addDisposableTo(disposeBag)

### 对textview的非法字符进行过滤
把textview.rx.text转换成Observable，监听map方法，bind to textview

    textView.rx.text.orEmpty.asObservable().distinctUntilChanged().map({
            $0.trimmingCharacters(in: CharacterSet(charactersIn: "[\"'<>%;]"))
        }).bind(to: textView.rx.text).addDisposableTo(disposeBag)

### 完结
[这里](https://gist.github.com/jackyshan/4451040ed8053a057ae9c3daa52e8b46)有代码片段，可以作为参考