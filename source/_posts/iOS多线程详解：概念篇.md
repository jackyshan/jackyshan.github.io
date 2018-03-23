---
title: iOS多线程详解：概念篇
date: 2018-03-23 20:12:25
tags: iOS多线程
---


![](http://upload-images.jianshu.io/upload_images/301129-b23b2a2ecc8c16be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

讲多线程这个话题，就免不了先了解多线程相关的技术概念。本文涉及到的技术概念有CPU、进程、线程、同异步、队列等概念。
也可能讲的不全或者不足的地方，后续再加以补充，最近一直使用Swift进行开发，本文所有代码例子都会Swift4进行演示。

### CPU

#### CPU是什么

> 引自维基百科[CPU](https://zh.wikipedia.org/wiki/%E4%B8%AD%E5%A4%AE%E5%A4%84%E7%90%86%E5%99%A8)中央处理器 （英语：Central Processing Unit，缩写：CPU），是计算机的主要设备之一，功能主要是解释计算机指令以及处理计算机软件中的数据。
> 
> 计算机的可编程性主要是指对中央处理器的编程。
>
> 中央处理器、内部存储器和输入／输出设备是现代电脑的三大核心部件。
>
> 1970年代以前，中央处理器由多个独立单元构成，后来发展出由集成电路制造的中央处理器，这些高度收缩的组件就是所谓的微处理器，其中分出的中央处理器最为复杂的电路可以做成单一微小功能强大的单元。

`CPU`主要由运算器、控制器、寄存器三部分组成，从字面意思看就是运算就是起着运算的作用，控制器就是负责发出`CPU`每条指令所需要的信息，寄存器就是保存运算或者指令的一些临时文件，这样可以保证更高的速度。
`CPU`有着处理指令、执行操作、控制时间、处理数据四大作用，打个比喻来说，`CPU`就像我们的大脑，帮我们完成各种各样的生理活动。因此如果没有`CPU`，那么电脑就是一堆废物，无法工作。

#### 多核CPU与多个CPU多核

> 引自[知乎](https://www.zhihu.com/question/20998226/answer/18659825)架构可以千变万化，面向需求、综合考量是王道。
>
> 来，简单举个例子。假设现在我们要设计一台计算机的处理器部分的架构。现在摆在我们面前的有两种选择，多个单核CPU和单个多核CPU。如果我们选择多个单核CPU，那么每一个CPU都需要有较为独立的电路支持，有自己的Cache，而他们之间通过板上的总线进行通信。假如在这样的架构上，我们要跑一个多线程的程序（常见典型情况），不考虑超线程，那么每一个线程就要跑在一个独立的CPU上，线程间的所有协作都要走总线，而共享的数据更是有可能要在好几个Cache里同时存在。这样的话，总线开销相比较而言是很大的，怎么办？那么多Cache，即使我们不心疼存储能力的浪费，一致性怎么保证？如果真正做出来，还要在主板上占多块地盘，给布局布线带来更大的挑战，怎么搞定？如果我们选择多核单CPU，那么我们只需要一套芯片组，一套存储，多核之间通过芯片内部总线进行通信，共享使用内存。
>
> 在这样的架构上，如果我们跑一个多线程的程序，那么线程间通信将比上一种情形更快。如果最终实现出来，对板上空间的占用较小，布局布线的压力也较小。看起来，多核单CPU完胜嘛。可是，如果需要同时跑多个大程序怎么办？假设俩大程序，每一个程序都好多线程还几乎用满cache，它们分时使用CPU，那在程序间切换的时候，光指令和数据的替换就要费多大事情啊！所以呢，大部分一般咱们使用的电脑，都是单CPU多核的，比如我们配的Dell T3600，有一颗Intel Xeon E5-1650，6核，虚拟为12个逻辑核心。少部分高端人士需要更强的多任务并发能力，就会搞一个多颗多核CPU的机子，Mac Pro就可以有两颗。

__一个核心同时只能处理一个线程，单核CPU只能实现并发，而不是并行__。如果有2个线程，双核`CPU`，那这两个线程是并行的，如果有三个线程，那么就还是并发的。下面会讲到并发与并行的区别。

### 进程

#### 进程是什么

> 引自维基百科[进程](https://zh.wikipedia.org/wiki/%E8%A1%8C%E7%A8%8B)（英语：process），是计算机中已运行程序的实体。进程为曾经是分时系统的基本运作单位。在面向进程设计的系统（如早期的UNIX，Linux 2.4及更早的版本）中，进程是程序的基本执行实体；在面向线程设计的系统（如当代多数操作系统、Linux 2.6及更新的版本）中，进程本身不是基本运行单位，而是线程的容器。
> 
> 程序本身只是指令、数据及其组织形式的描述，进程才是程序（那些指令和数据）的真正运行实例。
> 
> 若干进程有可能与同一个程序相关系，且每个进程皆可以同步（循序）或异步（平行）的方式独立运行。现代计算机系统可在同一段时间内以进程的形式将多个程序加载到内存中，并借由时间共享（或称时分复用），以在一个处理器上表现出同时（平行性）运行的感觉。
> 
> 同样的，使用多线程技术（多线程即每一个线程都代表一个进程内的一个独立执行上下文）的操作系统或计算机架构，同样程序的平行线程，可在多CPU主机或网络上真正同时运行（在不同的CPU上）。


在`iOS`系统中，一个`APP`的运行实体代表一个进程。一个进程有独立的内存空间、系统资源、端口等。在进程中可以生成多个线程、这些线程可以共享进程中的资源。

打个比方，`CPU`好比是一个工厂，进程是一个车间，线程是车间里面的工人。车间的空间是工人们共享的，比如许多房间是每个工人都可以进出的。这象征一个进程的内存空间是共享的，每个线程都可以使用这些共享内存。

#### 进程间通信

搜集了一下资料，`iOS`大概有8种进程间的通信方式，可能不全，后续补充。

`iOS`系统是相对封闭的系统，`App`各自在各自的沙盒（sandbox）中运行，每个`App`都只能读取`iPhone`上`iOS`系统为该应用程序程序创建的文件夹`AppData`下的内容，不能随意跨越自己的沙盒去访问别的`App`沙盒中的内容。
所以`iOS`的系统中进行`App`间通信的方式也比较固定，常见的`App`间通信方式以及使用场景总结如下。

* 1、Port (local socket)

上层封装为`NSMachPort` : `Foundation`层
中层封装为`CFMachPort` ： `Core Foundation`层
下层封装为`Mach Ports` : `Mach`内核层（线程、进程都可使用它进行通信）

一个`App1`在本地的端口`port1234`进行`TCP`的`bind`和`listen`，另外一个`App2`在同一个端口`port1234`发起`TCP`的`connect`连接，这样就可以建立正常的`TCP`连接，进行`TCP`通信了，那么就想传什么数据就可以传什么数据了。但是有一个限制，就是要求两个`App`进程都在活跃状态，而没有被后台杀死。尴尬的一点是`iOS`系统会给每个`TCP`在后台`600`秒的网络通信时间，`600`秒后`APP`会进入休眠状态。

* 2、URL Scheme

这个是`iOS` `App`通信最常用到的通信方式，`App1`通过`openURL`的方法跳转到`App2`，并且在`URL`中带上想要的参数，有点类似`http`的`get`请求那样进行参数传递。这种方式是使用最多的最常见的，使用方法也很简单只需要源`App1`在`info.plist`中配置`LSApplicationQueriesSchemes`，指定目标`App2`的`scheme`；然后在目标`App2`的`info.plist`中配置好`URL types`，表示该`App`接受何种`URL Scheme`的唤起。

![](http://upload-images.jianshu.io/upload_images/301129-35e624848548a619.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

典型的使用场景就是各开放平台`SDK`的分享功能，如分享到微信朋友圈微博等，或者是支付场景。比如从滴滴打车结束行程跳转到微信进行支付。

![](http://upload-images.jianshu.io/upload_images/301129-20283f3bcbd7e406.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 3、Keychain

`iOS`系统的`Keychain`是一个安全的存储容器，它本质上就是一个`sqllite`数据库，它的位置存储在`/private/var/Keychains/keychain-2.db`，不过它所保存的所有数据都是经过加密的，可以用来为不同的`App`保存敏感信息，比如用户名，密码等。`iOS`系统自己也用`Keychain`来保存`VPN`凭证和`Wi-Fi`密码。它是独立于每个`App`的沙盒之外的，所以即使`App`被删除之后，`Keychain`里面的信息依然存在。

基于安全和独立于`App`沙盒的两个特性，`Keychain`主要用于给`App`保存登录和身份凭证等敏感信息，这样只要用户登录过，即使用户删除了`App`重新安装也不需要重新登录。

那`Keychain`用于`App`间通信的一个典型场景也和`App`的登录相关，就是统一账户登录平台。使用同一个账号平台的多个`App`，只要其中一个`App`用户进行了登录，其他app就可以实现自动登录不需要用户多次输入账号和密码。一般开放平台都会提供登录`SDK`，在这个`SDK`内部就可以把登录相关的信息都写到`Keychain`中，这样如果多个`App`都集成了这个`SDK`，那么就可以实现统一账户登录了。

`Keychain`的使用比较简单，使用`iOS`系统提供的类`KeychainItemWrapper`，并通过`Keychain access groups`就可以在应用之间共享`Keychain`中的数据的数据了。

```
import Security
// MARK: - 保存和读取UUID

class func saveUUIDToKeyChain() {
    var keychainItem = KeychainItemWrapper(account: "Identfier", service: "AppName", accessGroup: nil)
    var string = (keychainItem[(kSecAttrGeneric as! Any)] as! String)
    if (string == "") || !string {
        keychainItem[(kSecAttrGeneric as! Any)] = self.getUUIDString()
    }
}

class func readUUIDFromKeyChain() -> String {
    var keychainItemm = KeychainItemWrapper(account: "Identfier", service: "AppName", accessGroup: nil)
    var UUID = (keychainItemm[(kSecAttrGeneric as! Any)] as! String)
    return UUID
}

class func getUUIDString() -> String {
    var uuidRef = CFUUIDCreate(kCFAllocatorDefault)
    var strRef = CFUUIDCreateString(kCFAllocatorDefault, uuidRef)
    var uuidString = (strRef as! String).replacingOccurrencesOf("-", withString: "")
    CFRelease(strRef)
    CFRelease(uuidRef)
    return uuidString
}

```

* 4、UIPasteboard

顾名思义， `UIPasteboard`是剪切板功能，因为`iOS`的原生控件`UITextView`，`UITextField` 、`UIWebView`，我们在使用时如果长按，就会出现复制、剪切、选中、全选、粘贴等功能，这个就是利用了系统剪切板功能来实现的。而每一个App都可以去访问系统剪切板，所以就能够通过系统剪贴板进行`App`间的数据传输了。 

```
//创建系统剪贴板
let pasteboard = UIPasteboard.general
//往剪贴板写入淘口令
pasteboard.string = "复制这条信息￥rkUy0Mz97CV￥后打开👉手淘👈"

//淘宝从后台切到前台，读取淘口令进行展示
let alert = UIAlertView.init(title: "淘口令", message: "发现一个宝贝，口令是rkUy0Mz97CV", delegate: self, cancelButtonTitle: "取消", otherButtonTitles: "查看")
alert.show()

```

`UIPasteboard`典型的使用场景就是淘宝跟微信/QQ的链接分享。由于腾讯和阿里的公司战略，腾讯在微信和QQ中都屏蔽了淘宝的链接。那如果淘宝用户想通过QQ或者微信跟好友分享某个淘宝商品，怎么办呢？ 阿里的工程师就巧妙的利用剪贴板实现了这个功能。首先淘宝`App`中将链接自定义成淘口令，引导用户进行复制，并去QQ好友对话中粘贴。然后QQ好友收到消息后再打开自己的淘宝`App`，淘宝`App`每次从后台切到前台时，就会检查系统剪切板中是否有淘口令，如果有淘口令就进行解析并跳转到对于的商品页面。

![](http://upload-images.jianshu.io/upload_images/301129-6b3e0f17dc3acc1e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

微信好友把淘口令复制到淘宝中，就可以打开好友分享的淘宝链接了。

* 5、UIDocumentInteractionController

`UIDocumentInteractionController`主要是用来实现同设备上`App`之间的共享文档，以及文档预览、打印、发邮件和复制等功能。它的使用非常简单.

首先通过调用它唯一的类方法`interactionControllerWithURL:`，并传入一个`URL(NSURL)`，为你想要共享的文件来初始化一个实例对象。然后`UIDocumentInteractionControllerDelegate`，然后显示菜单和预览窗口。

```
let url = Bundle.main.url(forResource: "test", withExtension: "pdf")
if url != nil {
    let documentInteractionController = UIDocumentInteractionController.init(url: url!)
    documentInteractionController.delegate = self
    documentInteractionController.presentOpenInMenu(from: self.view.bounds, in: self.view, animated: true)
}

```
效果如下图
![](http://upload-images.jianshu.io/upload_images/301129-29eacd86401453d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 6、AirDrop

> 引自维基百科[AirDrop](https://zh.wikipedia.org/wiki/%E9%9A%94%E7%A9%BA%E6%8A%95%E9%80%81)是苹果公司的MacOS和iOS操作系统中的一个随建即连网络，自Mac OS X Lion（Mac OS X 10.7）和iOS 7起引入，允许在支持的麦金塔计算机和iOS设备上传输文件，无需透过邮件或大容量存储设备。
>
> 在OS X Yosemite（OS X 10.10）之前，OS X 中的隔空投送协议不同于iOS的隔空投送协议，因此不能互相传输[2]。但是，OS X Yosemite或更新版本支持iOS的隔空投送协议（使用Wi-Fi和蓝牙），这适用于一台Mac与一台iOS设备以及两台2012年或更新版本的Mac计算机之间的传输。[3][4]使用旧隔空投送协议（只使用Wi-Fi）的旧模式在两台2012年或更早的Mac计算机之间传输也是可行的。[4]
>
> 隔空投送所容纳的文件大小没有限制。苹果用户报告称隔空投送能传输小于10GB的视频文件。

`iOS`并没有直接提供`AirDrop`的实现接口，但是使用`UIActivityViewController`的方法唤起`AirDrop`，进行数据交互。

* 7、UIActivityViewController

`UIActivityViewController`类是一个标准的`ViewController`，提供了几项标准的服务，比如复制项目至剪贴板，把内容分享至社交网站，以及通过`Messages`发送数据等等。在`iOS 7 SDK`中，`UIActivityViewController`类提供了内置的`AirDrop`功能。
 
如果你有一些数据一批对象需要通过`AirDrop`进行分享，你所需要的是通过对象数组初始化`UIActivityViewController`，并展示在屏幕上：

```
UIActivityViewController *controller = [[UIActivityViewController alloc] initWithActivityItems:objectsToShare applicationActivities:nil]; 
[self presentViewController:controller animated:YES completion:nil]; 

```

效果图如下
![](http://upload-images.jianshu.io/upload_images/301129-5e072e5d2df5b389.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 8、App Groups

`App Group`用于同一个开发团队开发的`App`之间，包括`App`和`Extension`之间共享同一份读写空间，进行数据共享。同一个团队开发的多个应用之间如果能直接数据共享，大大提高用户体验。

实现细节参考[App之间的数据共享——`App Group`的配置](https://www.jianshu.com/p/94d4106b9298)

### 线程

#### 线程是什么
>引自维基百科[线程](https://zh.wikipedia.org/wiki/%E7%BA%BF%E7%A8%8B)（英语：thread）是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。
>
>一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。在Unix System V及SunOS中也被称为轻量进程（lightweight processes），但轻量进程更多指内核线程（kernel thread），而把用户线程（user thread）称为线程。

讲线程就不能不提任务，任务是什么的，通俗的说任务就是就一件事情或一段代码，线程其实就是去执行这件事情。

> 线程（thread），指的是一个独立的代码执行路径，也就是说线程是代码执行路径的最小分支。在 iOS 中，线程的底层实现是基于 POSIX threads API 的，也就是我们常说的 pthreads ；

#### 超线程技术
>引自维基百科[超线程](https://zh.wikipedia.org/wiki/%E8%B6%85%E5%9F%B7%E8%A1%8C%E7%B7%92)（HT, Hyper-Threading）超线程技术就是利用特殊的硬件指令，把一个物理内核模拟成两个逻辑内核，让单个处理器都能使用线程级并行计算，进而兼容多线程操作系统和软件，减少了CPU的闲置时间，提高了CPU的运行速度。 采用超线程即是可在同一时间里，应用程序可以使用芯片的不同部分。

>引自[知乎](https://www.zhihu.com/question/30313735/answer/48006772)超线程这个东西并不是开了就一定比不开的好。因为每个CPU核心里ALU，FPU这些运算单元的数量是有限的，而超线程的目的之一就是在一个线程用运算单元少的情况下，让另外一个线程跑起来，不让运算单元闲着。但是如果当一个线程整数，浮点运算各种多，当前核心运算单元没多少空闲了，这时候你再塞进了一个线程，这下子资源就紧张了。两线程就会互相抢资源，拖慢对方速度。至于，超线程可以解决一个线程cache miss，另外一个可以顶上，但是如果两个线程都miss了，那就只有都在等了。这个还是没有GPU里一个SM里很多warp，超多线程同时跑来得有效果。所以，如果你的程序是单线程，关了超线程，免得别人抢你资源，如果是多线程，每个线程运算不大，超线程比较有用。

#### 线程间通信

线程间通信的表现为：一个线程传递数据给另一个线程；在一个线程中执行完特定任务后，转到另一个线程继续执行任务。

下面主要是介绍__其他线程执行耗时任务，在主线程进行UI的刷新__，也是业务中比较常用的一种。

* 1、NSThread线程间通信

__NSThread是用Swift语言里的Thread__

`NSThread`这套方案是经过苹果封装后，并且完全面向对象的。所以你可以直接操控线程对象，非常直观和方便。不过它的生命周期还是需要我们手动管理，所以实际上使用也比较少，使用频率较多的是`GCD`以及`NSOperation`。

当然，`NSThread`还可以用来做线程间通讯，比如下载图片并展示为例，将下载耗时操作放在子线程，下载完成后再切换回主线程在`UI`界面对图片进行展示

```
func onThread() {
    let urlStr = "http://tupian.aladd.net/2015/7/2941.jpg"
    self.performSelector(inBackground: #selector(downloadImg(_:)), with: urlStr)
}

@objc func downloadImg(_ urlStr: String) {
    //打印当前线程
    print("下载图片线程", Thread.current)
    
    //获取图片链接
    guard let url = URL.init(string: urlStr) else {return}
    //下载图片二进制数据
    guard let data = try? Data.init(contentsOf: url) else {return}
    //设置图片
    guard let img = UIImage.init(data: data) else {return}
    
    //回到主线程刷新UI
    self.performSelector(onMainThread: #selector(downloadFinished(_:)), with: img, waitUntilDone: false)
}

@objc func downloadFinished(_ img: UIImage) {
    //打印当前线程
    print("刷新UI线程", Thread.current)
}
```

> 下载图片线程 <NSThread: 0x1c4464a00>{number = 5, name = (null)}
刷新UI线程 <NSThread: 0x1c007a400>{number = 1, name = main}

有的小伙伴应该会有疑问，__为什么执行的`NSObject`的方法实现的线程，怎么变成了`NSThread`的线程呢。__
其实这个方法是`NSObject`对`NSThread`的封装，方便快速实现线程的方法，下个断点验证下看看。

![](http://upload-images.jianshu.io/upload_images/301129-799eea88ecb805c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 2、GCD线程间通信

`GCD（Grand Central Dispatch）`伟大的中央调度系统，是苹果为多核并行运算提出的C语言并发技术框架。
`GCD`会自动利用更多的`CPU`内核。
会自动管理线程的生命周期（创建线程，调度任务，销毁线程等）。
程序员只需要告诉`GCD`想要如何执行什么任务，不需要编写任何线程管理代码。

```
func onThread() {
    let urlStr = "http://tupian.aladd.net/2015/7/2941.jpg"

    let dsp = DispatchQueue.init(label: "com.jk.thread")
    dsp.async {
        self.downloadImg(urlStr)
    }
}

@objc func downloadImg(_ urlStr: String) {
    //打印当前线程
    print("下载图片线程", Thread.current)
    
    //获取图片链接
    guard let url = URL.init(string: urlStr) else {return}
    //下载图片二进制数据
    guard let data = try? Data.init(contentsOf: url) else {return}
    //设置图片
    guard let img = UIImage.init(data: data) else {return}
    
    //回到主线程刷新UI
    DispatchQueue.main.async {
        self.downloadFinished(img)
    }
}

@objc func downloadFinished(_ img: UIImage) {
    //打印当前线程
    print("刷新UI线程", Thread.current)
}

```

> 下载图片线程 <NSThread: 0x1c426b9c0>{number = 4, name = (null)}
刷新UI线程 <NSThread: 0x1c0263140>{number = 1, name = main}


* 3、NSOperation线程间通信

__NSOperation是用Swift语言里的Operation__

`NSOperation`是苹果推荐使用的并发技术，它提供了一些用`GCD`不是很好实现的功能。相比`GCD`，`NSOperation`的使用更加简单。`NSOperation`是一个抽象类，也就是说它并不能直接使用，而是应该使用它的子类。__`Swift`里面可以使用`BlockOperation`和自定义继承自`Operation`的子类。__

`NSOperation`的使用常常是配合`NSOperationQueue`来进行的。只要是使用`NSOperation`的子类创建的实例就能添加到`NSOperationQueue`操作队列之中，一旦添加到队列，操作就会自动异步执行（注意是异步）。如果没有添加到队列，而是使用`start`方法，则会在当前线程执行。

```
func onThread() {
    let urlStr = "http://tupian.aladd.net/2015/7/2941.jpg"

    let que = OperationQueue.init()
    que.addOperation {
        self.downloadImg(urlStr)
    }
}

@objc func downloadImg(_ urlStr: String) {
    //打印当前线程
    print("下载图片线程", Thread.current)
    
    //获取图片链接
    guard let url = URL.init(string: urlStr) else {return}
    //下载图片二进制数据
    guard let data = try? Data.init(contentsOf: url) else {return}
    //设置图片
    guard let img = UIImage.init(data: data) else {return}
    
    //回到主线程刷新UI
    OperationQueue.main.addOperation {
        self.downloadFinished(img)
    }
}

@objc func downloadFinished(_ img: UIImage) {
    //打印当前线程
    print("刷新UI线程", Thread.current)
}
```

> 下载图片线程 <NSThread: 0x1c4271d80>{number = 3, name = (null)}
刷新UI线程 <NSThread: 0x1c006cac0>{number = 1, name = main}

`OperationQueue`的对象执行`addOperation`的方法，其实是生成了一个`BlockOperation`对象，异步执行当前任务。
下个断点，可以看到`BlockOperation`的执行过程。

![](http://upload-images.jianshu.io/upload_images/301129-b874dc5bcb560226.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 线程池

> [线程池](https://zh.wikipedia.org/zh-hans/线程池)（英语：thread pool）：一种线程使用模式。 线程过多会带来调度开销，进而影响缓存局部性和整体性能。 而线程池维护着多个线程，等待着监督管理者分配可并发执行的任务。 这避免了在处理短时间任务时创建与销毁线程的代价。

线程池的执行流程如：
首先，启动若干数量的线程，并让这些线程处于睡眠状态
其次，当客户端有新的请求时，线程池会唤醒某一个睡眠线程，让它来处理客户端的请求
最后，当请求处理完毕，线程又处于睡眠状态

__所以在并发的时候，同时能有多少线程在跑是有线程池的线程缓存数量决定的。__

* 1、GCD

`GCD`有一个底层线程池，这个池中存放的是一个个的线程。之所以称为“池”，很容易理解出这个“池”中的线程是可以重用的，当一段时间后这个线程没有被调用的话，这个线程就会被销毁。池是系统自动来维护，不需要我们手动来维护。

那`GCD`底层线程池的缓存数到底有多少个的，写段代码跑一下看看。

```
@IBAction func onThread() {
    let dsp = DispatchQueue.init(label: "com.jk.thread", attributes: .concurrent)
    for i in 0..<10000 {
        dsp.async {
            //打印当前线程
            print("\(i)当前线程", Thread.current)
            //耗时任务
            Thread.sleep(forTimeInterval: 5)
        }
    }
}

```

![](http://upload-images.jianshu.io/upload_images/301129-a8e6c8c2841dc9b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这段代码是生成了一个并发队列，`for`循环`10000`次，执行异步任务，相当于会生成`10000`条线程，由于异步执行任务，会立即执行而且不会等待任务的结束，所以在生成线程的时候，线程打印就立即执行了。从打印的结果来看，__一次打印总共有`64`行，从而可以得出`GCD`的线程池缓存数量是`64`条__。
当然如果想实现更多线程的并发执行的话，可以使用开源的[YYDispatchQueuePool](https://github.com/ibireme/YYDispatchQueuePool)。

* 2、NSOperation

`NSOperationQueue`提供了一套类似于线程池的机制，通过它可以更加方便的进行多线程的并发操作，构造一个线程池并添加任务对象到线程池中，线程池会分配线程，调用任务对象的`main`方法执行任务。

下面写段代码，分配`maxConcurrentOperationCount`为`3`个，看看效果

```
@IBAction func onThread() {
    let opq = OperationQueue.init()
    opq.maxConcurrentOperationCount = 3
    for i in 0..<10 {
        opq.addOperation({
            //打印当前线程
            print("\(i)当前线程", Thread.current)
            //耗时任务
            Thread.sleep(forTimeInterval: 5)
        })
    }
}
```

> 1当前线程 <NSThread: 0x1c0274a80>{number = 4, name = (null)}
0当前线程 <NSThread: 0x1c4673900>{number = 6, name = (null)}
2当前线程 <NSThread: 0x1c4674040>{number = 7, name = (null)}

可以看到，规定线程池缓存为`3`个，一次就打印`3`个线程，当这`3`个线程回收到线程池里，又会再打印`3`个，当然__如果其中一个线程先执行完，他就会先被回收__。

`NSOperationQueue`一次能够并发执行多少线程呢，跑一下下面代码

```
@IBAction func onThread() {
    let opq = OperationQueue.init()
    opq.maxConcurrentOperationCount = 300
    for i in 0..<100 {
        opq.addOperation({
            //打印当前线程
            print("\(i)当前线程", Thread.current)
            //耗时任务
            Thread.sleep(forTimeInterval: 5)
        })
    }
}

```

![](http://upload-images.jianshu.io/upload_images/301129-a8e6c8c2841dc9b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到也是`64`个，也就是__`NSOperationQueue`多了可以操作线程数量的接口，但是最大的线程并发数量还是`64`个__。


#### 多线程同步

##### 1、多线程同步是什么

> 引自百度百科[同步](https://baike.baidu.com/item/%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5)就是协同步调，按预定的先后次序进行运行。可理解为线程A和B一块配合，A执行到一定程度时要依靠B的某个结果，于是停下来，示意B运行；B依言执行，再将结果给A；A再继续操作。

线程同步其实是对于并发队列说的，串行队列的任务是依次执行的，本身就是同步的。

![](http://upload-images.jianshu.io/upload_images/301129-5782718a2b6e3a04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 2、多线程同步用途

__结果传递__：A执行到一定程度时要依靠B的某个结果，于是停下来，示意B运行；B依言执行，再将结果给A；A再继续操作。例子，小明、小李有三个西瓜，这个三个西瓜可以同时切开（并发），全部切完之后放到冰箱里冰好（同步），小明、小李吃冰西瓜（并发）。

__资源竞争__：是指多个线程同时访问一个资源时可能存在竞争问题提供的解决方案，使多个线程可以对同一个资源进行操作，比如线程A为数组M添加了一个数据，线程B可以接收到添加数据后的数组M。线程同步就是线程之间相互的通信。例子，购买火车票，多个窗口卖票（并发），票卖出去之后要把库存减掉（同步），多个窗口出票成功（并发）。


##### 3、多线程同步实现

多线程同步实现的方式有很多信号量(`DispatchSemaphore`)、锁(`NSLock`)、`@synchronized`、`dispatch_barrier_async`、`addDependency`、`pthread_mutex_t`，下面用这个方式实现火车票的去库存的情况。

* DispatchSemaphore

> GCD中信号量，也可以解决资源抢占问题，支持信号通知和信号等待。每当发送一个信号通知，则信号量+1；每当发送一个等待信号时信号量-1；如果信号量为0则信号会处于等待状态，直到信号量大于0开始执行。

简单地说就是洗手间只有一个坑位，外面进来一个人把门关上，其他人排队，这个人把门打开出去之后，可以再进来一个人。代码例子如下。

```
var tickets: [Int] = [Int]()

@IBAction func onThread() {
    let que = DispatchQueue.init(label: "com.jk.thread", attributes: .concurrent)
    
    //生成100张票
    for i in 0..<100 {
        tickets.append(i)
    }
    
    //北京卖票窗口
    que.async {
        self.saleTicket()
    }
    
    //上海卖票窗口
    que.async {
        self.saleTicket()
    }
}

//存在一个坑位
let semp = DispatchSemaphore.init(value: 1)

func saleTicket() {
    while true {
        //占坑，坑位减一
        semp.wait()
        
        if tickets.count > 0 {
            print("剩余票数", tickets.count, "卖票窗口", Thread.current)
            tickets.removeLast()
            Thread.sleep(forTimeInterval: 0.2)
        }
        else {
            print("票已经卖完了")
            
            //释放占坑，坑位加一
            semp.signal()
            
            break
        }
        
        //释放占坑，坑位加一
        semp.signal()
    }
}

```

> 打印结果：
> 剩余票数 100 卖票窗口 <NSThread: 0x1c0472e40>{number = 6, name = (null)}
剩余票数 99 卖票窗口 <NSThread: 0x1c027b540>{number = 4, name = (null)}
剩余票数 98 卖票窗口 <NSThread: 0x1c0472e40>{number = 6, name = (null)}
剩余票数 97 卖票窗口 <NSThread: 0x1c027b540>{number = 4, name = (null)}
剩余票数 96 卖票窗口 <NSThread: 0x1c0472e40>{number = 6, name = (null)}
剩余票数 95 卖票窗口 <NSThread: 0x1c027b540>{number = 4, name = (null)}
> .................
> 剩余票数 4 卖票窗口 <NSThread: 0x1c0472e40>{number = 6, name = (null)}
剩余票数 3 卖票窗口 <NSThread: 0x1c027b540>{number = 4, name = (null)}
剩余票数 2 卖票窗口 <NSThread: 0x1c0472e40>{number = 6, name = (null)}
剩余票数 1 卖票窗口 <NSThread: 0x1c027b540>{number = 4, name = (null)}
票已经卖完了
票已经卖完了


在不使用信号量的情况下，运行一段时间就会崩溃，这是多线程同事操作`tickets`票池的`removeLast`去库存的方法引起的，这样显然不符合我们的需求，所以我们需要考虑线程安全问题。

* NSLock

> 锁的概念，锁是最常用的同步工具。一段代码段在同一个时间只能允许被一个线程访问，比如一个线程A进入加锁代码之后由于已经加锁，另一个线程B就无法访问，只有等待前一个线程A执行完加锁代码后解锁，B线程才能访问加锁代码。
不要将过多的其他操作代码放到里面，否则一个线程执行的时候另一个线程就一直在等待，就无法发挥多线程的作用了。

在`Cocoa`程序中`NSLock`中实现了一个简单的互斥锁，实现了`NSLocking Protocol`。实现代码如下：

```
var tickets: [Int] = [Int]()

@IBAction func onThread() {
    let que = DispatchQueue.init(label: "com.jk.thread", attributes: .concurrent)
    
    //生成100张票
    for i in 0..<100 {
        tickets.append(i)
    }
    
    //北京卖票窗口
    que.async {
        self.saleTicket()
    }
    
    //上海卖票窗口
    que.async {
        self.saleTicket()
    }
}

//生成一个锁
let lock = NSLock.init()

func saleTicket() {
    while true {
        //关门，执行任务
        lock.lock()
        
        if tickets.count > 0 {
            print("剩余票数", tickets.count, "卖票窗口", Thread.current)
            tickets.removeLast()
            Thread.sleep(forTimeInterval: 0.2)
        }
        else {
            print("票已经卖完了")
            
            //开门，让其他任务可以执行
            lock.unlock()
            
            break
        }
        
        //开门，让其他任务可以执行
        lock.unlock()
    }
}

```

> 打印结果：
> 剩余票数 100 卖票窗口 <NSThread: 0x1c467d300>{number = 6, name = (null)}
剩余票数 99 卖票窗口 <NSThread: 0x1c4862380>{number = 7, name = (null)}
剩余票数 98 卖票窗口 <NSThread: 0x1c467d300>{number = 6, name = (null)}
剩余票数 97 卖票窗口 <NSThread: 0x1c4862380>{number = 7, name = (null)}
剩余票数 96 卖票窗口 <NSThread: 0x1c467d300>{number = 6, name = (null)}
剩余票数 95 卖票窗口 <NSThread: 0x1c4862380>{number = 7, name = (null)}
> .................
> 剩余票数 4 卖票窗口 <NSThread: 0x1c467d300>{number = 6, name = (null)}
剩余票数 3 卖票窗口 <NSThread: 0x1c4862380>{number = 7, name = (null)}
剩余票数 2 卖票窗口 <NSThread: 0x1c467d300>{number = 6, name = (null)}
剩余票数 1 卖票窗口 <NSThread: 0x1c4862380>{number = 7, name = (null)}
票已经卖完了
票已经卖完了

* @synchronized

在`Objective-C`中，我们可以用`@synchronized`关键字来修饰一个对象，并为其自动加上和解除互斥锁。
但是在Swift中，没有与之对应的方法，即`@synchronized`在`Swift`中已经（或者是暂时）不存在了。其实`@synchronized`在幕后做的事情是调用了`objc_sync`中的`objc_sync_enter`和`objc_sync_exit`方法，我可以直接调用这两个方法去实现。

```
var tickets: [Int] = [Int]()
    
@IBAction func onThread() {
    let que = DispatchQueue.init(label: "com.jk.thread", attributes: .concurrent)
    
    //生成100张票
    for i in 0..<100 {
        tickets.append(i)
    }
    
    //北京卖票窗口
    que.async {
        self.saleTicket()
    }
    
    //上海卖票窗口
    que.async {
        self.saleTicket()
    }
}

func saleTicket() {
    while true {
        //加锁，关门，执行任务
        objc_sync_enter(self)
        
        if tickets.count > 0 {
            print("剩余票数", tickets.count, "卖票窗口", Thread.current)
            tickets.removeLast()
            Thread.sleep(forTimeInterval: 0.2)
        }
        else {
            print("票已经卖完了")
            
            //开锁，开门，让其他任务可以执行
            objc_sync_exit(self)
            
            break
        }
        
        //开锁，开门，让其他任务可以执行
        objc_sync_exit(self)
    }
}
```

> 打印结果：
> 剩余票数 100 卖票窗口 <NSThread: 0x1c04697c0>{number = 6, name = (null)}
剩余票数 99 卖票窗口 <NSThread: 0x1c44706c0>{number = 4, name = (null)}
剩余票数 98 卖票窗口 <NSThread: 0x1c04697c0>{number = 6, name = (null)}
剩余票数 97 卖票窗口 <NSThread: 0x1c44706c0>{number = 4, name = (null)}
剩余票数 96 卖票窗口 <NSThread: 0x1c04697c0>{number = 6, name = (null)}
> .................
> 剩余票数 3 卖票窗口 <NSThread: 0x1c44706c0>{number = 4, name = (null)}
剩余票数 2 卖票窗口 <NSThread: 0x1c04697c0>{number = 6, name = (null)}
剩余票数 1 卖票窗口 <NSThread: 0x1c44706c0>{number = 4, name = (null)}
票已经卖完了
票已经卖完了

* GCD 栅栏方法：dispatch_barrier_async

> 我们有时需要异步执行两组操作，而且第一组操作执行完之后，才能开始执行第二组操作。这样我们就需要一个相当于 栅栏 一样的一个方法将两组异步执行的操作组给分割起来，当然这里的操作组里可以包含一个或多个任务。这就需要用到dispatch_barrier_async方法在两个操作组间形成栅栏。
dispatch_barrier_async函数会等待前边追加到并发队列中的任务全部执行完毕之后，再将指定的任务追加到该异步队列中。然后在dispatch_barrier_async函数追加的任务执行完毕之后，异步队列才恢复为一般动作，接着追加任务到该异步队列并开始执行。

```
var tickets: [Int] = [Int]()

@IBAction func onThread() {
    let que = DispatchQueue.init(label: "com.jk.thread", attributes: .concurrent)
    
    //生成100张票
    for i in 0..<100 {
        tickets.append(i)
    }
    
    for _ in 0..<51 {
        //北京卖票窗口
        que.async {
            self.saleTicket()
        }
        
        //GCD 栅栏方法，同步去库存
        que.async(flags: .barrier) {
            if self.tickets.count > 0 {
                self.tickets.removeLast()
            }
        }
        
        //上海卖票窗口
        que.async {
            self.saleTicket()
        }
        
        //GCD 栅栏方法，同步去库存
        que.async(flags: .barrier) {
            if self.tickets.count > 0 {
                self.tickets.removeLast()
            }
        }
    }
}

func saleTicket() {
    if tickets.count > 0 {
        print("剩余票数", tickets.count, "卖票窗口", Thread.current)
        Thread.sleep(forTimeInterval: 0.2)
    }
    else {
        print("票已经卖完了")
    }
}

```

> 打印结果：
> 剩余票数 100 卖票窗口 <NSThread: 0x1c0463c40>{number = 3, name = (null)}
剩余票数 99 卖票窗口 <NSThread: 0x1c0463c40>{number = 3, name = (null)}
剩余票数 98 卖票窗口 <NSThread: 0x1c0463c40>{number = 3, name = (null)}
剩余票数 97 卖票窗口 <NSThread: 0x1c0463c40>{number = 3, name = (null)}
> .................
> 剩余票数 59 卖票窗口 <NSThread: 0x1c0670100>{number = 6, name = (null)}
剩余票数 58 卖票窗口 <NSThread: 0x1c0670100>{number = 6, name = (null)}
剩余票数 57 卖票窗口 <NSThread: 0x1c0670100>{number = 6, name = (null)}
剩余票数 56 卖票窗口 <NSThread: 0x1c0670100>{number = 6, name = (null)}
> .................
> 剩余票数 3 卖票窗口 <NSThread: 0x1c0463c40>{number = 3, name = (null)}
剩余票数 2 卖票窗口 <NSThread: 0x1c0463c40>{number = 3, name = (null)}
剩余票数 1 卖票窗口 <NSThread: 0x1c0463c40>{number = 3, name = (null)}
票已经卖完了
票已经卖完了

* addDependency(操作依赖)

> NSOperation、NSOperationQueue最吸引人的地方是它能添加操作之间的依赖关系。通过操作依赖，我们可以很方便的控制操作之间的执行先后顺序。下面使用操作依赖实现多线程同步，代码如下。

```
var tickets: [Int] = [Int]()

@IBAction func onThread() {
    let que = OperationQueue.init()//并发队列
    que.maxConcurrentOperationCount = 1
    
    //生成100张票
    for i in 0..<100 {
        tickets.append(i)
    }
    
    for _ in 0..<51 {
        //addDependency方法，同步去库存
        let sync1 = BlockOperation.init(block: {
            if self.tickets.count > 0 {
                self.tickets.removeLast()
            }
        })
        
        //北京卖票窗口
        let bj = BlockOperation.init(block: {
            self.saleTicket()
        })
        bj.addDependency(sync1)//等待去库存
        
        //addDependency方法，同步去库存
        let sync2 = BlockOperation.init(block: {
            if self.tickets.count > 0 {
                self.tickets.removeLast()
            }
        })
        
        //上海卖票窗口
        let sh = BlockOperation.init(block: {
            self.saleTicket()
        })
        sh.addDependency(sync2)//等待去库存
        
        que.addOperation(sync1)
        que.addOperation(bj)
        que.addOperation(sync2)
        que.addOperation(sh)
    }
}

func saleTicket() {
    if tickets.count > 0 {
        print("剩余票数", tickets.count, "卖票窗口", Thread.current)
        Thread.sleep(forTimeInterval: 0.2)
    }
    else {
        print("票已经卖完了")
    }
}

```

> 打印结果：
> 剩余票数 99 卖票窗口 <NSThread: 0x1c42672c0>{number = 4, name = (null)}
剩余票数 98 卖票窗口 <NSThread: 0x1c06731c0>{number = 5, name = (null)}
剩余票数 97 卖票窗口 <NSThread: 0x1c06731c0>{number = 5, name = (null)}
剩余票数 96 卖票窗口 <NSThread: 0x1c06731c0>{number = 5, name = (null)
> .................
> 剩余票数 54 卖票窗口 <NSThread: 0x1c42672c0>{number = 4, name = (null)}
剩余票数 53 卖票窗口 <NSThread: 0x1c42672c0>{number = 4, name = (null)}
剩余票数 52 卖票窗口 <NSThread: 0x1c42672c0>{number = 4, name = (null)}
> .................
> 剩余票数 2 卖票窗口 <NSThread: 0x1c42672c0>{number = 4, name = (null)}
剩余票数 1 卖票窗口 <NSThread: 0x1c42672c0>{number = 4, name = (null)}
票已经卖完了
票已经卖完了
票已经卖完了


* 使用POSIX互斥锁

> POSIX互斥锁在很多程序里面很容易使用。为了新建一个互斥锁，你声明并初始化一个pthread_mutex_t的结构。为了锁住和解锁一个互斥锁，你可以使用pthread_mutex_lock和pthread_mutex_unlock函数。列表4-2显式了要初始化并使用一个POSIX线程的互斥锁的基础代码。当你用完一个锁之后，只要简单的调用pthread_mutex_destroy来释放该锁的数据结构。


```
var tickets: [Int] = [Int]()

@IBAction func onThread() {
    let que = DispatchQueue.init(label: "com.jk.thread", attributes: .concurrent)
    
    mutex()
    
    //生成100张票
    for i in 0..<100 {
        tickets.append(i)
    }
    
    //北京卖票窗口
    que.async {
        self.saleTicket()
    }
    
    //上海卖票窗口
    que.async {
        self.saleTicket()
    }
}

//生成一个锁
var lock = pthread_mutex_t.init()

func mutex() {
    //设置属性
    var attr: pthread_mutexattr_t = pthread_mutexattr_t()
    pthread_mutexattr_init(&attr)
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE)
    let err = pthread_mutex_init(&self.lock, &attr)
    pthread_mutexattr_destroy(&attr)
    
    switch err {
    case 0:
        // Success
        break
        
    case EAGAIN:
        fatalError("Could not create mutex: EAGAIN (The system temporarily lacks the resources to create another mutex.)")
        
    case EINVAL:
        fatalError("Could not create mutex: invalid attributes")
        
    case ENOMEM:
        fatalError("Could not create mutex: no memory")
        
    default:
        fatalError("Could not create mutex, unspecified error \(err)")
    }
}


func saleTicket() {
    while true {
        //关门，执行任务
        pthread_mutex_lock(&lock)
        
        if tickets.count > 0 {
            print("剩余票数", tickets.count, "卖票窗口", Thread.current)
            tickets.removeLast()
            Thread.sleep(forTimeInterval: 0.2)
        }
        else {
            print("票已经卖完了")
            
            //开门，让其他任务可以执行
            pthread_mutex_unlock(&lock)
            
            break
        }
        
        //开门，让其他任务可以执行
        pthread_mutex_unlock(&lock)
    }
}

deinit {
    pthread_mutex_destroy(&lock)
}
```

> 打印结果：
> 剩余票数 100 卖票窗口 <NSThread: 0x1c446f7c0>{number = 5, name = (null)}
剩余票数 99 卖票窗口 <NSThread: 0x1c0472900>{number = 6, name = (null)}
剩余票数 98 卖票窗口 <NSThread: 0x1c446f7c0>{number = 5, name = (null)}
剩余票数 97 卖票窗口 <NSThread: 0x1c0472900>{number = 6, name = (null)}
剩余票数 96 卖票窗口 <NSThread: 0x1c446f7c0>{number = 5, name = (null)}
> .................
> 剩余票数 35 卖票窗口 <NSThread: 0x1c0472900>{number = 6, name = (null)}
剩余票数 34 卖票窗口 <NSThread: 0x1c446f7c0>{number = 5, name = (null)}
剩余票数 33 卖票窗口 <NSThread: 0x1c0472900>{number = 6, name = (null)}
剩余票数 32 卖票窗口 <NSThread: 0x1c446f7c0>{number = 5, name = (null)}
> .................
> 剩余票数 2 卖票窗口 <NSThread: 0x1c446f7c0>{number = 5, name = (null)}
剩余票数 1 卖票窗口 <NSThread: 0x1c0472900>{number = 6, name = (null)}
票已经卖完了
票已经卖完了

可以看到使用`pthread_mutex_t`完成了多线程同步，当然实现锁的类型有很多，比如还有`NSRecursiveLock`(递归锁)、`NSConditionLock`(条件锁)、`NSDistributedLock`(分布式锁)、`OSSpinLock`(自旋锁)等方式，就不用代码一一实现了。


### 同异步和队列

#### 同异步是什么

> 步和异步操作的主要区别在于是否等待操作执行完成，亦即是否阻塞当前线程。同步操作会等待操作执行完成后再继续执行接下来的代码，而异步操作则恰好相反，它会在调用后立即返回，不会等待操作的执行结果。


| | 同步 | 异步 |
|:-:|:-:|:-:|
| 是否阻塞当前线程 | 是 | 否 |
| 是否等待任务执行完成 | 是 | 否 |


#### 队列是什么

* 队列含义

> 引自维基百科[队列](https://zh.wikipedia.org/wiki/%E9%98%9F%E5%88%97)，又称为伫列（queue），是先进先出（FIFO, First-In-First-Out）的线性表。在具体应用中通常用链表或者数组来实现。队列只允许在后端（称为rear）进行插入操作，在前端（称为front）进行删除操作。
队列的操作方式和堆栈类似，唯一的区别在于队列只允许新数据在后端进行添加。

![](http://upload-images.jianshu.io/upload_images/301129-194805ad2e7c51c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 串行队列和并发队列

串行队列一次只能执行一个任务，而并发队列则可以允许多个任务同时执行。`iOS`系统就是使用这些队列来进行任务调度的，它会根据调度任务的需要和系统当前的负载情况动态地创建和销毁线程，而不需要我们手动地管理。

__注意这个并发多个任务同时执行的`同时`在二字指的是同一时间内(由于`CPU`执行很快，感觉几乎同时)__

这里会有一个经常性的疑问，串行队列一次执行一个任务，任务按顺序执行，先进先出，这个好理解。__那并发几个任务同时执行也是先进先出，这个怎么理解呢。因为并发执行任务，先进去的任务并不一定先执行完，但是即使后面的任务先执行完，也是要等前面的任务退出。这是由队列的性质决定的。__

| | 串行队列 | 并发队列 |
|:-:|:-:|:-:|
| 同步执行 | 当前线程，一个接着一个地执行，顺序执行，一个任务执行完毕后，再执行下一个任务 | 当前线程，一个接着一个地执行，顺序执行，一个任务执行完毕后，再执行下一个任务 |
| 异步执行 | 其他线程，一个接着一个地执行 | 多个任务，多个线程，多个任务并发执行 |


![](http://upload-images.jianshu.io/upload_images/301129-6d802467e0d7e547.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/301129-8a3924f342170ccd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

__注意这里说的是队列执行时间，并不是先进先出的示例。__


* 并发队列和并行队列

> 并发的来历
> 在过去单CPU时代，单任务在一个时间点只能执行单一程序。之后发展到多任务阶段，计算机能在同一时间点并行执行多任务或多进程。虽然并不是真正意义上的“同一时间点”，而是多个任务或进程共享一个CPU，并交由操作系统来完成多任务间对CPU的运行切换，以使得每个任务都有机会获得一定的时间片运行。
> 并行的来历
> 多线程比多任务更加有挑战。多线程是在同一个程序内部并行执行，因此会对相同的内存空间进行并发读写操作。这可能是在单线程程序中从来不会遇到的问题。其中的一些错误也未必会在单CPU机器上出现，因为两个线程从来不会得到真正的并行执行。然而，更现代的计算机伴随着多核CPU的出现，也就意味着__不同的线程能被不同的CPU核得到真正意义的并行执行__。

并发的关键是你有处理多个任务的能力，不一定要同时。
并行的关键是你有同时处理多个任务的能力。
__并发是一种能力，处理多个任务的能力。并行是状态，多个任务同时执行的状态。__
可以看到并发和并行并不是同一类概念，所以不具有比较性，并发包含并行，就比如水果是包含西瓜一样。并发的不一定是并行，并行的一定是并发。

> 一个CPU的`核心`同时只能处理一个线程。
> 单核CPU一个线程，当前是并行(两个以上才叫并行，为了理解，暂且这样叫吧)。单核CPU两个线程，当前是并发。双核CPU两个线程，当前是并行。双核CPU四个线程，当前是并发。


![](http://upload-images.jianshu.io/upload_images/301129-fd60408209a40942.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/301129-057641b244b01cce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
