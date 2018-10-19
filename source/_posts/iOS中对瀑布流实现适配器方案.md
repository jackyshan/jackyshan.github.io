---
title: iOS中对瀑布流实现适配器方案
date: 2018-10-19 22:42:14
tags: iOS
---

![](https://upload-images.jianshu.io/upload_images/301129-067a5a2f0888d6e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 介绍

适配器模式是就是把一种接口转换成另一种接口，统一给调用者提供简单好用的接口。
如上图所示，在工业设计上，苹果电脑为了美观取消了以太网接口，那我们的电脑如果要使用有线以太网，就需要这根线做转接，把USB转成以太网输出，这样苹果电脑就可以上有线网络了。

适配器方案在项目开发中能够大量节约开发和维护成本，Android系统框架中有实现好的适配器方案，对ListView实现的很友好，可以提高对ListView的开发和维护。

我封装了BaseTableViewAdapter和BaseCollectionViewAdapter

在Controller里实现TableView和CollectionView只需要下面这样写

```
import UIKit

class ViewController: UIViewController {
    
    @IBOutlet weak var tableView: UITableView!
    @IBOutlet weak var collectionView: UICollectionView!
    
    var cAdapter: MixCollectionViewAdapter?
    var tAdapter: MixTableViewAdapter?

    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view, typically from a nib.
        
        cAdapter = MixCollectionViewAdapter(collectionView)
        cAdapter?.dataSoure = [1, 1, 1, 1, 1, 1, 1]
        
        tAdapter = MixTableViewAdapter(tableView)
        tAdapter?.dataSoure = [1, 1, 1, 1, 1, 1, 1]
    }
}
```

运行结果如下

![](https://upload-images.jianshu.io/upload_images/301129-dd37744cb785d666.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


发现没有TableView和CollectionView的逻辑只需要写这么点代码。

下面对iOS中UITableView和UICollectionView实现适配器方案。

### Swift实现

使用Swift里面实现UITableView的适配器的封装，其中数据声明使用泛型，避免底层参与上层业务逻辑。

#### UITableView实现

##### BaseTableViewAdapter

实现`BaseTableViewAdapter`，基础Adapter，用于继承

```
class BaseTableViewAdapter<T>: NSObject, UITableViewDelegate, UITableViewDataSource {
    var cellClick:((_ obj:T)->Void)?
    var cellClickIndex:((_ obj:T,_ index:IndexPath)->Void)?
    
    var mTableView:UITableView?
    var mDataSource:[T]?
    var cellHeight:CGFloat = 60
    
    init(_ tableView: UITableView) {
        super.init()
        mTableView = tableView
        mTableView!.dataSource = self
        mTableView!.delegate = self

        onCreate()
    }
    
    var dataSoure:[T] = []{
        willSet{
            mDataSource = newValue
        }
        
        didSet{
            mTableView?.reloadData()
        }
    }
    
    func onCreate() {
        
    }
    
    func numberOfSections(in tableView: UITableView) -> Int {
        return 1
    }
    
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        self.mTableView!.deselectRow(at: indexPath, animated: true)
        self.cellClick?(self.mDataSource![indexPath.row])
        self.cellClickIndex?(self.mDataSource![indexPath.row], indexPath)
    }
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return self.mDataSource?.count ?? 0
    }
    
    func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
        return cellHeight
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let data: T = self.mDataSource![indexPath.row]
        return jkTableView(tableView, cellForRowAtIndexPath: indexPath, bingData: data)
    }
    
    func jkTableView(_ tableView: UITableView, cellForRowAtIndexPath indexPath: IndexPath, bingData: T) -> UITableViewCell {
        return UITableViewCell()
    }
    
    func cellOnClick(_ action:@escaping (T) -> Void){
        self.cellClick = action
    }
    
    func cellOnClickIndex(_ action:@escaping (_ obj:T,_ index:IndexPath)->Void){
        self.cellClickIndex = action
    }
    
}

```

下面讲解代码实现步骤

```
    init(_ tableView: UITableView) {
        super.init()
        mTableView = tableView
        mTableView!.dataSource = self
        mTableView!.delegate = self

        onCreate()
    }
```

把UITableView传进来，实现UITableView的代理方法

```
    var dataSoure:[T] = [] {
        willSet{
            mDataSource = newValue
        }
        
        didSet{
            mTableView?.reloadData()
        }
    }
```

dataSoure是数据源，调用reloadData，实现TableView代理数据刷新

```
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        self.mTableView!.deselectRow(at: indexPath, animated: true)
        self.cellClick?(self.mDataSource![indexPath.row])
        self.cellClickIndex?(self.mDataSource![indexPath.row], indexPath)
    }
```

实现点击，通过block接口传递出去，这样把代理方法通过block传递，此对象实现了大部分的UITableView初始化和代理的逻辑

```
BaseTableViewAdapter<T>
```

声明泛型T，前面说了传数据，那我们的数据类型是不固定的，通过泛型我们不需要知道数据类型，因为交给上层去处理就好了。

##### 实现Adapter

前面实现了BaseTableViewAdapter，下面我们利用继承BaseTableViewAdapter，实现业务。

实现MixTableViewAdapter，如下。

```
class MixTableViewAdapter: BaseTableViewAdapter<Int> {

    override func onCreate() {
        cellHeight = 60
        mTableView?.registerNib(MixVolumeTableViewCell.self)
    }
    
    override func jkTableView(_ tableView: UITableView, cellForRowAtIndexPath indexPath: IndexPath, bingData: Int) -> UITableViewCell {
        let cell: MixVolumeTableViewCell = tableView.dequeueReusableCell(indexPath: indexPath)
        
        //处理data
        //...
        
        return cell
    }
    
}
```

```
    override func onCreate() {
        cellHeight = 60
        mTableView?.registerNib(MixVolumeTableViewCell.self)
    }
```

onCreate里面进行一些TableView的初始化，行高、注册cell等。

```
    override func jkTableView(_ tableView: UITableView, cellForRowAtIndexPath indexPath: IndexPath, bingData: Int) -> UITableViewCell {
        let cell: MixVolumeTableViewCell = tableView.dequeueReusableCell(indexPath: indexPath)
        
        //处理data
        //...
        
        return cell
    }
```

```
class MixTableViewAdapter: BaseTableViewAdapter<Int>
```

声明泛型类型为Int，处理数据源，通过泛型传递，bingData是泛型传过来的Int。

#### UICollectionView实现

##### BaseCollectionViewAdapter

实现`BaseCollectionViewAdapter`，基础Adapter，用于继承

```
class BaseCollectionViewAdapter<T>: NSObject, UICollectionViewDelegate, UICollectionViewDataSource {
    
    var cellClick:((_ obj:T)->Void)?
    var mCollectionView: UICollectionView?
    var mDataSource: [T] = [T]()
    
    init(_ collectionView: UICollectionView) {
        super.init()
        
        collectionView.dataSource = self
        collectionView.delegate = self
        
        mCollectionView = collectionView
        onCreate()
    }
    
    func onCreate() {
        
    }

    var dataSoure: [T] = [] {
        willSet {
            mDataSource = newValue
        }
        
        didSet {
            mCollectionView?.reloadData()
        }
    }
    
    // MARK: UICollectionViewDataSource
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return mDataSource.count
    }
    
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let data: T = mDataSource[indexPath.item]
        return jkCollectionView(collectionView, cellForItemAt: indexPath, data: data)
    }
    
    func jkCollectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath, data: T) -> UICollectionViewCell {
        return UICollectionViewCell()
    }
    
    // MARK: UICollectionViewDelegate
    func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) {
        collectionView.deselectItem(at: indexPath, animated: true)
        
        cellClick?(mDataSource[indexPath.item])
    }
    
    func collectionView(_ collectionView: UICollectionView, canMoveItemAt indexPath: IndexPath) -> Bool {
        return false
    }
    
    func collectionView(_ collectionView: UICollectionView, moveItemAt sourceIndexPath: IndexPath, to destinationIndexPath: IndexPath) {
        
    }

    // MARK: - other
    func cellOnClick(_ action:@escaping (T) -> Void) {
        self.cellClick = action
    }

}
```

代码讲解

```
    init(_ collectionView: UICollectionView) {
        super.init()
        
        collectionView.dataSource = self
        collectionView.delegate = self
        
        mCollectionView = collectionView
        onCreate()
    }
```

传递UICollectionView，实现代理方法

```
    var dataSoure: [T] = [] {
        willSet {
            mDataSource = newValue
        }
        
        didSet {
            mCollectionView?.reloadData()
        }
    }
```

传递数据源，刷新CollectionView

```
    func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) {
        collectionView.deselectItem(at: indexPath, animated: true)
        
        cellClick?(mDataSource[indexPath.item])
    }
```

实现点击操作，通过block回调

```
class BaseCollectionViewAdapter<T>: NSObject, UICollectionViewDelegate, UICollectionViewDataSource
```

泛型传递

##### 实现Adapter

新建MixCollectionViewAdapter继承BaseCollectionViewAdapter，声明泛型为Int，代码如下。

```
import UIKit

class MixCollectionViewAdapter: BaseCollectionViewAdapter<Int> {

    override func onCreate() {
        let itemCountOnLine: Int = UIScreen.main.bounds.width > 320 ? 4 : 3
        mCollectionView?.collectionViewLayout = UICollectionViewFlowLayout.flowWithItemOnLine(itemCountOnLine, margin: 12)
    
        mCollectionView?.registerNib(MixItemCollectionViewCell.self)
    }
    
    override func jkCollectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath, data: Int) -> UICollectionViewCell {
        let cell: MixItemCollectionViewCell = collectionView.dequeueReusableCell(indexPath: indexPath)
        
        //处理data
        //...
        
        return cell
    }
}
```

在onCreate实现CollectionView的进一步初始化

```
    override func onCreate() {
        let itemCountOnLine: Int = UIScreen.main.bounds.width > 320 ? 4 : 3
        mCollectionView?.collectionViewLayout = UICollectionViewFlowLayout.flowWithItemOnLine(itemCountOnLine, margin: 12)
    
        mCollectionView?.registerNib(MixItemCollectionViewCell.self)
    }
```

回调处理数据

```
    override func jkCollectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath, data: Int) -> UICollectionViewCell {
        let cell: MixItemCollectionViewCell = collectionView.dequeueReusableCell(indexPath: indexPath)
        
        //处理data
        //...
        
        return cell
    }
```

### OC实现

因为在OC里面id可以代表一切类型，数据传递可以使用id，运行时直接解析成我们需要的数据类型就可以了。

#### UITableView实现

##### BaseTableViewAdapter

OC实现虽有区别，但是大体上是一样的，代码如下。

```
#import "BaseTableViewAdapter.h"

@implementation BaseTableViewAdapter

- (instancetype)initWithTableView:(UITableView *)tableView {
    if (self = [super init]) {
        _tableView = tableView;
        tableView.delegate = self;
        tableView.dataSource = self;
        [self onCreate];
    }
    
    return self;
}

- (void)onCreate {
    _cellHeight = 64;
}

- (void)setDataSource:(NSArray *)dataSource {
    _dataSource = dataSource;
    
    [_tableView reloadData];
}

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    return _cellHeight;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return _dataSource.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    return [self tableView:tableView cellForObj:_dataSource[indexPath.row]];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForObj:(id)obj {
    return [UITableViewCell new];
}

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:true];
    if (_cellBlock == nil) {
        return;
    }
    _cellBlock(_dataSource[indexPath.row]);
}

@end
```

初始化TableView，实现代理方法，传递点击事件。

##### 实现Adapter

继承BaseTableViewAdapter，实现业务逻辑，代码如下。

```
#import "MixTableViewAdapter.h"
#import "MixVolumeTableViewCell.h"

@implementation MixTableViewAdapter

- (void)onCreate {
    self.cellHeight = 60;
    [self.tableView registerNib:[MixVolumeTableViewCell class]];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForObj:(id)obj {
    MixVolumeTableViewCell *cell = [tableView dequeueReusableCell:[MixVolumeTableViewCell class]];
    return cell;
}

@end
```

实现进一步的初始化操作，可以在此实现cell的业务逻辑。

#### UICollectionView实现

OC里面UICollectionView的实现逻辑也是大同小异

##### BaseCollectionViewAdapter

```
#import "BaseCollectionViewAdapter.h"

@implementation BaseCollectionViewAdapter

- (instancetype)initWithCollectionView:(UICollectionView *)collectionView {
    if (self = [super init]) {
        _collectionView = collectionView;
        collectionView.delegate = self;
        collectionView.dataSource = self;
        [self onCreate];
    }
    
    return self;
}

- (void)onCreate {
    
}

- (void)setDataSource:(NSArray *)dataSource {
    _dataSource = dataSource;
    
    [_collectionView reloadData];
}

- (NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section {
    return _dataSource.count;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath {
    return [self collectionView:collectionView cellForObj:_dataSource[indexPath.item] andIndexPath:indexPath];
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForObj:(id)obj andIndexPath:(NSIndexPath *)indexPath {
    return [UICollectionViewCell new];
}

- (void)collectionView:(UICollectionView *)collectionView didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    [collectionView deselectItemAtIndexPath:indexPath animated:YES];
    if (_cellBlock == nil) {
        return;
    }
    _cellBlock(_dataSource[indexPath.item]);
}

@end
```

传递CollectionView，实现代理方法，传递数据源，刷新数据，传递点击事件。

##### 实现Adapter

继承BaseCollectionViewAdapter，继续业务逻辑，代码如下。

```
#import "MixCollectionViewAdapter.h"
#import "MixItemCollectionViewCell.h"

@implementation MixCollectionViewAdapter

- (void)onCreate {
    NSInteger itemCountOnLine = [UIScreen mainScreen].bounds.size.width > 320 ? 4 : 3;
    self.collectionView.collectionViewLayout = [UICollectionViewFlowLayout flowLayoutWithItemCountOnLine:itemCountOnLine forMargin:12];
    
    [self.collectionView registerNib:[MixItemCollectionViewCell class]];
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForObj:(id)obj andIndexPath:(nonnull NSIndexPath *)indexPath {
    MixItemCollectionViewCell *cell = [collectionView dequeueReusableCell:[MixItemCollectionViewCell class] forIdp:indexPath];
    cell.bgView.isChecked = YES;
    
    return cell;
}

@end
```

进一步初始化操作，实现cell的业务逻辑。

### 总结

iOS中各种MVX模式天天讨论，孰优孰虑？实际上平时的代码中，如果很好的使用设计模式，做代码的解耦，像适配器模式这样很好的使用，代码易读，开发维护成本降低，使用MVC开发绰绰有余了。

本文测试代码放到[GitHub](https://github.com/jackyshan/iOSAdapterTest)上了，有需要可以去查看。

### 关注我

欢迎关注公众号：jackyshan，技术干货首发微信，第一时间推送。


![](http://upload-images.jianshu.io/upload_images/301129-bdd8c22856a81ede?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
