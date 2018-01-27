---
title: 使用RxSwift开发登录注册忘记密码模块总结
date: 2018-01-27 20:19:06
tags: RxSwift
---

![img](http://upload-images.jianshu.io/upload_images/301129-c0b63bc3d92bd864.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


刚好有新的项目，就把登录这一块的逻辑全部用RxSwift去写了，用很少的代码量实现了所有的逻辑，这就是RxSwift的魅力吧，下面是项目登录注册模块的演示。

![gif](http://upload-images.jianshu.io/upload_images/301129-602672650d962bfa.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 登录

当前登录页面的逻辑分为密码登录和验证码登录，点击上面tab进行切换。上面的tab是我自定义的控件，切换的值通过block传回来。

忘记密码控件: 密码登录的时候需要显示，验证码登录需要隐藏。
获取验证码控件：密码登录的时候需要隐藏，验证码登录需要显示。
输入密码图标在验证码登录切换的时候变成验证码的图标。

基于以上的逻辑，我增加一个全局的响应`bool`变量`isPasswdLoginSeg`，让这个变量的的值跟着切换的值变换，然后把响应变量的`bool`值绑定到相应的控件上，实现相关控件的隐藏和显示。

---

默认是密码登录
```
let isPasswdLoginSeg: Variable<Bool> = Variable(true)
```

这个是自定义的控件，切换的值是0代表是密码登录，1代表验证码登录
```
jySegmentDayNightView.clickDayNightBlock = { [weak self] idx in
    self?.verifyPasswdFd.text = ""
    self?.verifyPasswdFd.sendActions(for: .valueChanged)
    self?.isPasswdLoginSeg.value = idx == 0
    self?.verifyPasswdFd.resignFirstResponder()
}
```

`isPasswdLoginSeg`作为Observable发送值的变化，然后传送bool值，绑定到其他控件的rx属性，或者相关控件根据发送的bool值进行改变
```
isPasswdLoginSeg.asObservable().bind(to: verifyBtn.rx.isHidden).addDisposableTo(disposeBag)
isPasswdLoginSeg.asObservable().map({!$0}).bind(to: forgetPasswdBtn.rx.isHidden).addDisposableTo(disposeBag)

isPasswdLoginSeg.asObservable().subscribe(onNext: { [weak self] (isPasswd) in
    self?.verifyPasswdImgV.image = isPasswd ? #imageLiteral(resourceName: "passwd_login") : #imageLiteral(resourceName: "verifycode_login")
    self?.verifyPasswdFd.placeholder = isPasswd ? "请输入密码" : "请输入验证码"
    self?.verifyPasswdFd.keyboardType = isPasswd ? .default : .numberPad
    self?.verifyPasswdFd.isSecureTextEntry = isPasswd
}).addDisposableTo(disposeBag)
```
---

登录btn控件的enable属性需要跟着电话号码和密码或验证码的输入进行变化。enable属性的颜色提前设置好。验证码控件也是根据电话号码输入框的值是否符合要求改变它的enable值。


`Observable.combineLatest`结合`phoneFd `和`verifyPasswdFd `控件的rx值绑定到loginBtn的enable属性。
```
Observable.combineLatest(phoneFd.rx.text.orEmpty.asObservable(), verifyPasswdFd.rx.text.orEmpty.asObservable()){ $0.count == 11 && $1.count > 0 }.bind(to: loginBtn.rx.isEnabled).addDisposableTo(disposeBag)

```

`phoneFd`变化绑定到验证码控件的enable，通过map进行输入的判断转换bool值，下面是判断输入是否满足11位，这个比较简单，也可以加个正则判断输入
```
phoneFd.rx.text.orEmpty.asObservable().distinctUntilChanged().map({ $0.count == 11 }).bind(to: verifyBtn.rx.isEnabled).addDisposableTo(disposeBag)
```

__下面有注册和忘记密码的rx写法，就不解释了。主要是对逻辑的封装，用rx的Observable进行值的绑定和转换，然后完成相应的业务。__

### 注册

```
let disposeBag = DisposeBag()
func initLinstener() {
    Observable.combineLatest(phoneFd.rx.text.orEmpty.asObservable(), verifyFd.rx.text.orEmpty.asObservable(), passwdFd.rx.text.orEmpty.asObservable()){ $0.count == 11 && $1.count > 0 && ($2.count >= 6 && $2.count <= 14) }.bind(to: registerBtn.rx.isEnabled).addDisposableTo(disposeBag)
    
    phoneFd.rx.text.orEmpty.asObservable().distinctUntilChanged().map({ $0.count == 11 }).bind(to: verifyBtn.rx.isEnabled).addDisposableTo(disposeBag)

}
```

### 忘记密码

```
let disposeBag = DisposeBag()
func initLinstener() {
    let phoneValid = phoneFd.rx.text.orEmpty.distinctUntilChanged().asObservable().map({$0.count == 11})
    let veryCodeValid = verifyFd.rx.text.orEmpty.distinctUntilChanged().asObservable().map({$0.count > 0})
    let passwdValid = passwdFd.rx.text.orEmpty.distinctUntilChanged().asObservable().map({$0.count >= 6 && $0.count <= 14})
    let confirmPasswdValid = passwdAgainFd.rx.text.orEmpty.distinctUntilChanged().asObservable().map({$0.count >= 6 && $0.count <= 14})
    let passwdEqualValid = Observable.combineLatest(passwdFd.rx.text.orEmpty.distinctUntilChanged().asObservable(), passwdAgainFd.rx.text.orEmpty.distinctUntilChanged()).map({$0 == $1})
    
    phoneValid.bind(to: verifyBtn.rx.isEnabled).addDisposableTo(disposeBag)
    
    Observable.combineLatest(phoneValid, veryCodeValid, passwdValid, confirmPasswdValid, passwdEqualValid).map({$0 && $1 && $2 && $3 && $4}).bind(to: forgetBtn.rx.isEnabled).addDisposableTo(disposeBag)
    
}

```

### 完整代码
https://github.com/jackyshan/RxSwiftLoginDemo