---
title: iOS通过SSH远程管理macOS服务器文件
date: 2018-03-23 20:11:19
tags: iOS
---

![](http://upload-images.jianshu.io/upload_images/301129-fc4b1dd3e805c682.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以前写过一个iOS客户端播放视频的App，我在macOS搭建了web服务器实现目录浏览功能，把下载的视频资源放到web服务器的目录，这样iOS客户端可以通过webview获取到macOS服务端的视频资源的链接了，从而实现iOS上播放电脑上视频资源的功能。
但是用久了之后，发现我看完的电影资源还在那里，每次看视频去找新的没看过的就很麻烦。这样就想能不能写个程序，当我看完这个电影之后，直接删掉这个资源，或者把这个资源移动到其他收藏目录。通过这个想法，找了一些实现方法，感觉还是使用ssh去做比较快和方便。

![](http://upload-images.jianshu.io/upload_images/301129-d0f1d0a3f617f07d.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 使用分析blink
本文ssh的功能主要是通过开源工具[blink](https://github.com/blinksh/blink)实现，窗口实现的逻辑大部分在`TermController`里面

* 创建终端显示器
```
- (void)createPTY
{
  pipe(_pinput);
  _termout = fterm_open(_terminal, 0);
  _termerr = fterm_open(_terminal, 0);
  _termin = fdopen(_pinput[0], "r");
  _termsz = malloc(sizeof(struct winsize));
}
```
`_terminal `是一个`UIView`，他做了两件事，一个是创建`WKWebView`用来渲染我们的输入和输出数据，一个是监听键盘事件，把监听到的数据回调给`TermController `实现数据流的写入。

* 监听键盘事件
```
  // If the key is a special key, we do not apply modifiers.
  if (text.length > 1) {
    // Check if we have a function key
    NSRange range = [text rangeOfString:@"FKEY"];
    if (range.location != NSNotFound) {
      NSString *value = [text substringFromIndex:(range.length)];
      [_delegate write:[CC FKEY:[value integerValue]]];
    } else {
      [_delegate write:[CC KEY:text MOD:0 RAW:_raw]];
    }
  } else {
    NSUInteger modifiers = [[_smartKeys view] modifiers];
    if (modifiers & KbdCtrlModifier) {
      [_delegate write:[CC CTRL:text]];
    } else if (modifiers & KbdAltModifier) {
      [_delegate write:[CC ESC:text]];
    } else {
      [_delegate write:[CC KEY:text MOD:0 RAW:_raw]];
    }
  }
```
* 实现写入数据

```
- (void)write:(NSString *)input
{
  // Trasform the string and write it, with the correct sequence
  const char *str = [input UTF8String];
  write(_pinput[1], str, [input lengthOfBytesUsingEncoding:NSUTF8StringEncoding]);
}
```

* 创建session会话
```
  - (void)startSession
{
  // Until we are able to duplicate the streams, we have to recreate them.
  TermStream *stream = [[TermStream alloc] init];
  stream.in = _termin;
  stream.out = _termout;
  stream.err = _termerr;
  stream.control = self;
  stream.sz = _termsz;

  _session = [[MCPSession alloc] initWithStream:stream];
  _session.delegate = self;
  [_session executeWithArgs:@""];
}  
```
有了`MCPSession `就完成了整个会话的过程，如图
![](http://upload-images.jianshu.io/upload_images/301129-5d7debb9271a0ee3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* `WKWebView `渲染功能，是通过`js`加载方法渲染页面
```
// Write data to terminal control
- (void)write:(NSString *)data
{
  [_delegate receiveData:data];
  
  NSData *jsonData = [NSJSONSerialization dataWithJSONObject:@[ data ] options:0 error:nil];
  NSString *jsString = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
  NSString *jsScript = [NSString stringWithFormat:@"write_to_term(%@[0])", jsString];
  
  dispatch_async(dispatch_get_main_queue(), ^{
    [_webView evaluateJavaScript:jsScript completionHandler:nil];
  });
}
```

### 实现ssh输入输出
`blink`的ssh功能是在`MCPSession`里面实现了调用`SSHSession`，他们都是继承了`Session`类。`SSHSession`里面主要是对`libssh2`的framework提供的api实现了一套ssh的封装，比如登录中密码的判断、公私钥的验证等

[libssh2](https://www.libssh2.org/) 是一个使用 C 语言编写的以实现 SSH2 协议的代码库。

SSH 协议的全称是 [Secure Shell](https://link.jianshu.com?t=https://zh.wikipedia.org/wiki/Secure_Shell)，顾名思义就是为操作系统提供一个安全的 Shell 使得用户在和远程主机进行交互时的数据不易被第三方窃取。除了这个基本的功能之外，它还包含了很多其他的功能，比如 SFTP，端口转发等等。

* 实现ssh登录判断
```
- (void)ssh_login:(NSArray *)ids to:(struct sockaddr *)addr port:(int)port user:(const char *)user timeout:(int)timeout error:(NSError **)error
```
登录成功之后实现`SSHSession `的输入输出
在`- (int)ssh_client_loop`方法里面有个循环一直在处理输入输出数据流的解析工作

* 从`stream`获取流数据，`libssh2_channel_read `读取字符数据
```
  // Wait for stream->in or socket while not ready for reading
  do {
    if (!pfds[0].events || pfds[0].revents & (POLLIN)) {
      // Read from socket
      do {
	rc = libssh2_channel_read(_channel, inputbuf, BUFSIZ);
	if (rc > 0) {
	  fwrite(inputbuf, rc, 1, _stream.out);
	  pfds[0].events = 0;
	} 
```
* 从`stream`获取流数据，`libssh2_channel_write`写入字符数据
```
    // Input from stream
    if (pfds[1].revents & POLLIN) {
      towrite = fread(streambuf, 1, BUFSIZ, _stream.in);
      rc = 0;
      do {
	rc = libssh2_channel_write(_channel, streambuf + rc, towrite);
	if (rc > 0) {
	  towrite -= rc;
	}
```
### 实现文件列表和删除
在连上ssh之后，就可以通过命令实现列表和删除，命令太麻烦，那就写个界面实现操作。

* 创建`FileCommandDelegate `代理和`FileCommandViewController`类
```
@protocol FileCommandDelegate <NSObject>

- (void)excuteCommand:(NSString *)command;

@end

@interface FileCommandViewController : UIViewController

@property (weak) id<FileCommandDelegate> delegate;

@end
```

在`TermController`实现代理，执行命令
```
#pragma mark FileCommandDelegate
- (void)excuteCommand:(NSString *)command {
  [self write:command];
}
```
连上ssh，代理出去，弹出文件管理页面
```
- (void)sshConnected:(BOOL)isConnected {
  if (isConnected == false) {
    return;
  }
  
  dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 3 * NSEC_PER_SEC), dispatch_get_main_queue(), ^{
    FileCommandViewController *vc = [[FileCommandViewController alloc] init];
    vc.delegate = self;
    UINavigationController *navi = [[UINavigationController alloc] initWithRootViewController:vc];
    [self presentViewController:navi animated:YES completion:nil];
  });
  
}
```
* 在`FileCommandViewController`实现数据输入输出的通知
```
  [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(receiveData:) name:@"kCommandReceived" object:nil];
```
* 过滤获取的数据，实现列表
```
- (void)receiveData:(NSNotification * _Nonnull)note {
  
  //    NSLog(@"%@", note.object);
  
  NSString *checkString = note.object;
  //1.创建正则表达式，[0-9]:表示‘0’到‘9’的字符的集合
  NSString *pattern = @"\\[1m\\[36m.+\\[";
  //1.1将正则表达式设置为OC规则
  NSRegularExpression *regular = [[NSRegularExpression alloc] initWithPattern:pattern options:NSRegularExpressionCaseInsensitive error:nil];
  //2.利用规则测试字符串获取匹配结果
  NSArray *results = [regular matchesInString:checkString options:0 range:NSMakeRange(0, checkString.length)];
  
  if (results.count > 0) {
    //      NSLog(@"%@", results);
    NSMutableArray *reguslars = [NSMutableArray array];
    [results enumerateObjectsUsingBlock:^(NSTextCheckingResult * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
      NSString *subStr = [checkString substringWithRange:obj.range];
      NSString *checkString = subStr;
      //1.创建正则表达式，[0-9]:表示‘0’到‘9’的字符的集合
      NSString *pattern = @"\\[1m\\[36m";
      //1.1将正则表达式设置为OC规则
      NSRegularExpression *regular = [[NSRegularExpression alloc] initWithPattern:pattern options:NSRegularExpressionCaseInsensitive error:nil];
      //2.利用规则测试字符串获取匹配结果
      NSArray *results = [regular matchesInString:checkString options:0 range:NSMakeRange(0, checkString.length)];
      [results enumerateObjectsUsingBlock:^(NSTextCheckingResult *  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        NSString *subStr = [checkString substringWithRange:NSMakeRange(obj.range.location+obj.range.length, checkString.length-obj.range.length-obj.range.location)];
        NSString *checkString = subStr;
        //1.创建正则表达式，[0-9]:表示‘0’到‘9’的字符的集合
        NSString *pattern = @"(?=\\[).+";
        //1.1将正则表达式设置为OC规则
        NSRegularExpression *regular = [[NSRegularExpression alloc] initWithPattern:pattern options:NSRegularExpressionCaseInsensitive error:nil];
        //2.利用规则测试字符串获取匹配结果
        NSArray *results = [regular matchesInString:checkString options:0 range:NSMakeRange(0, checkString.length)];
        
        if (results.count > 0) {
          NSTextCheckingResult *res = results[0];
          NSString *sst = [checkString substringWithRange:NSMakeRange(0, res.range.location-1)];
          NSLog(@"%@", sst);
          [reguslars addObject:sst];
        }
        
      }];
      
    }];
    
    _datas = reguslars;
  }
  
  //1.创建正则表达式，[0-9]:表示‘0’到‘9’的字符的集合
  NSString *pattern1 = @"\\s\\w+(\\.\\w+)";
  //1.1将正则表达式设置为OC规则
  NSRegularExpression *regular1 = [[NSRegularExpression alloc] initWithPattern:pattern1 options:NSRegularExpressionCaseInsensitive error:nil];
  //2.利用规则测试字符串获取匹配结果
  NSArray *results1 = [regular1 matchesInString:checkString options:0 range:NSMakeRange(0, checkString.length)];
  if (results1.count > 0) {
    NSMutableArray *reguslars = [NSMutableArray array];
    [results1 enumerateObjectsUsingBlock:^(NSTextCheckingResult * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
      NSString *subStr = [[checkString substringWithRange:obj.range] stringByReplacingOccurrencesOfString:@" " withString:@""];
      [reguslars addObject:subStr];
      
    }];
    [reguslars addObjectsFromArray:_datas];
    _datas = reguslars;
  }

  if (_datas.count > 0) {
    NSLog(@"%@", checkString);
    dispatch_async(dispatch_get_main_queue(), ^{
      [_tableView reloadData];
    });
  }
  
}

```

* 点击刷新获取列表数据
```
- (void)refreshList {
  [_delegate excuteCommand:@"ls\n"];
}
```
![](http://upload-images.jianshu.io/upload_images/301129-87064f5c87272ce2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 左滑删除文件
```
- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
  
  [_delegate excuteCommand:[[NSString alloc] initWithFormat:@"rm %@\n", _datas[indexPath.row]]];
}
```
![](http://upload-images.jianshu.io/upload_images/301129-e96bf9a505cf0d14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 点击进入目录

这里判断是不是文件，如果是文件就直接return，不跳转
```
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
  [tableView deselectRowAtIndexPath:indexPath animated:YES];
  
  NSString *checkString = _datas[indexPath.row];
  //1.创建正则表达式，[0-9]:表示‘0’到‘9’的字符的集合
  NSString *pattern = @"\\.\\w+";
  //1.1将正则表达式设置为OC规则
  NSRegularExpression *regular = [[NSRegularExpression alloc] initWithPattern:pattern options:NSRegularExpressionCaseInsensitive error:nil];
  //2.利用规则测试字符串获取匹配结果
  NSArray *results = [regular matchesInString:checkString options:0 range:NSMakeRange(0, checkString.length)];
  
  if (results.count > 0) {
    return;
  }
  
  [_delegate excuteCommand:[[NSString alloc] initWithFormat:@"cd %@\n", checkString]];

  FileCommandViewController *vc = [[FileCommandViewController alloc] init];
  vc.delegate = _delegate;
  [self.navigationController pushViewController:vc animated:true];
}
```

### 代码
现在功能主要实现了列表和删除，出现一个问题，就是中文字符ssh返回的都是`???`，待研究解决。

https://github.com/jackyshan/iOSTerminalFromBlinkSSHFileManage
