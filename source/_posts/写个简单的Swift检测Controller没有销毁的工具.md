---
title: 写个简单的Swift检测Controller没有销毁的工具
date: 2018-01-27 20:25:06
tags: Swift
---

![img](http://upload-images.jianshu.io/upload_images/301129-f3b662834e3679e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 思路

在Swift的代码中，Controller没有销毁大部分的原因都是没有weak self。怎么检测一个Controller没有释放呢？

1、当一个对象销毁的时候，它会调用deinit的方法，一般Controller页面我们都是放在UINavigationController里面的，然后调用push和pop，实现我们页面的跳转。

2、继承UINavigationController，重写push方法，在push方法里面把push的controller的名字放到单例的数组里面，deinit的时候在把当前的controller从单例里面释放，然后检测单例里面有没有controller的相同名字存在2个以上的。

### 代码实现

因为是检测Controller有没有销毁的工具，然后打印到console里查看，所以代码应该在DEBUG模式下执行。

如果使用的UI架构师tabbar加几个controllers的样式，初始化tabbar的时候会调用push。所以判断如果是tabbarcontroller直接return
```
func defaultController() -> UINavigationController {
    let navi = JYNavigationViewController.init(rootViewController: JYTabBarViewController())

    return navi
}
```

```
func pushVc(_ vc: UIViewController) {
    #if DEBUG

    if vc is JYTabBarViewController {
        return
    }
    
    vcs.append(NSStringFromClass(vc.classForCoder))
    
    #endif
}

```

在继承的NavigationController实现push代码
```
override open func pushViewController(_ viewController: UIViewController, animated: Bool) {
    viewController.hidesBottomBarWhenPushed = self.viewControllers.count > 0
    
    if self.viewControllers.count > 0 {
        viewController.showLeftButton()
    }
    
    CheckWselfHelper.shared.pushVc(viewController)
    
    super.pushViewController(viewController, animated: animated)
}

```

在pop代码里调用数组的filter函数，过滤掉当前controller名字相同的内容，然后遍历数组，筛选出数组中名字相同有大于1个controller，并打印出来

```
func popVc(_ vc: UIViewController?) {
    #if DEBUG
        guard let vc = vc else {return}
        
        let str = NSStringFromClass(vc.classForCoder)
        vcs = vcs.filter({$0 != str})
        
        var datas = [String: Int]()
        
        vcs.forEach({ (str) in
            datas[str] = (datas[str] ?? 0)+1
        })
        
        datas.forEach({ (key, value) in
            if value > 1 {
                Log.i("注意"+key+"没有释放")
            }
        })

    #endif
}

```
在BaseController的deinit方法里实现我们的pop函数
```
deinit {
    CheckWselfHelper.shared.popVc(self)
}
```
由于我是tabba的ui架构，所以在点击tabbar的时候会push很多tabbar和navigation的controller。所以在点击和初始化的时候清空下我们的单例数组。

```
func clearVcs(_ addVc: UIViewController?) {
    #if DEBUG

    vcs.removeAll()
    
    guard let addVc = addVc else {return}
    
    pushVc(addVc)
        
    #endif
}

```

在tabbar里面的实现如下

初始化的时候清空
```
func initViews() {
    
    let tvc = JYTicketViewController()
    let ticketVc: JYNavigationViewController = JYNavigationViewController(rootViewController: tvc)
    let mallVc: JYNavigationViewController = JYNavigationViewController(rootViewController: JYMallViewController())
    let mineVc: JYNavigationViewController = JYNavigationViewController(rootViewController: JYMineViewController())
    
    ticketVc.tabBarItem.title = "购票"
    ticketVc.tabBarItem.image = UIImage.init(named: "ticket_normal")?.withRenderingMode(.alwaysOriginal)
    ticketVc.tabBarItem.selectedImage = UIImage.init(named: "ticket_selected")?.withRenderingMode(.alwaysOriginal)
    
    mallVc.tabBarItem.title = "商城"
    mallVc.tabBarItem.image = UIImage.init(named: "mall_normal")?.withRenderingMode(.alwaysOriginal)
    mallVc.tabBarItem.selectedImage = UIImage.init(named: "mall_selected")?.withRenderingMode(.alwaysOriginal)
    
    mineVc.tabBarItem.title =  "我的"
    mineVc.tabBarItem.image = UIImage.init(named: "mine_normal")?.withRenderingMode(.alwaysOriginal)
    mineVc.tabBarItem.selectedImage = UIImage.init(named: "mine_selected")?.withRenderingMode(.alwaysOriginal)

    self.tabBar.barTintColor = UIColor.white
    self.tabBar.tintColor = AppConfig.mainColor
    
    let lists = [ticketVc, mallVc, mineVc]
    self.viewControllers = lists
    self.delegate = self
    self.selectedIndex = 0
    ticketVc.navigationBar.isTranslucent = false
    mallVc.navigationBar.isTranslucent = false
    mineVc.navigationBar.isTranslucent = false
    
    CheckWselfHelper.shared.clearVcs(tvc)//这里初始化默认选中的controller

}
```

点击tabbar的时候清空
```
// MARK: UITabBarControllerDelegate
func tabBarController(_ tabBarController: UITabBarController, shouldSelect viewController: UIViewController) -> Bool {
    CheckWselfHelper.shared.clearVcs(viewController.childViewControllers.first)
    
    return true
}
```

### CheckWselfHelper.swift
https://gist.github.com/jackyshan/7a084291a03ae55815631697be1ae995