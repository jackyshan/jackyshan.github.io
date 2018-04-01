---
title: iOS内存管理详解
date: 2018-04-01 17:34:47
tags: iOS内存
---

![](https://upload-images.jianshu.io/upload_images/301129-2de59f3fc0ac2871.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图可以看到，栈里面存放的是值类型，堆里面存放的是对象类型。对象的引用计数是在堆内存中操作的。下面我们讲讲堆和栈怎么存放和操作数据， 还有`MRC`和`ARC`怎么管理引用计数。

### Heap(堆)和stack(栈)

#### 堆是什么

> 引自维基百科[堆](https://zh.wikipedia.org/wiki/%E5%A0%86_(%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84))（英语：Heap）是计算机科学中一类特殊的数据结构的统称。堆通常是一个可以被看做一棵树的数组对象。在队列中，调度程序反复提取队列中第一个作业并运行，因为实际情况中某些时间较短的任务将等待很长时间才能结束，或者某些不短小，但具有重要性的作业，同样应当具有优先权。堆即为解决此类问题设计的一种数据结构。

> 堆(Heap)又被为优先队列(priority queue)。尽管名为优先队列，但堆并不是队列。回忆一下，在队列中，我们可以进行的限定操作是dequeue和enqueue。dequeue是按照进入队列的先后顺序来取出元素。而在堆中，我们不是按照元素进入队列的先后顺序取出元素的，而是按照元素的优先级取出元素。

这就好像候机的时候，无论谁先到达候机厅，总是头等舱的乘客先登机，然后是商务舱的乘客，最后是经济舱的乘客。每个乘客都有头等舱、商务舱、经济舱三种个键值(key)中的一个。头等舱->商务舱->经济舱依次享有从高到低的优先级。

总的来说，堆是一种数据结构，数据的插入和删除是根据优先级定的，他有几个特性：
* 任意节点的优先级不小于它的子节点
* 每个节点值都小于或等于它的子节点
* 主要操作是插入和删除最小元素(元素值本身为优先级键值，小元素享有高优先级)

举个例子，就像叠罗汉，体重大(优先级低、值大)的站在最下面，体重小的站在最上面(优先级高，值小)。
为了让堆稳固，我们每次都让最上面的参与者退出堆，__也就是每次取出优先级最高的元素__。

![](https://upload-images.jianshu.io/upload_images/301129-f29cbaec8606bd0a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 栈是什么

> 引自维基百科[栈](https://zh.wikipedia.org/wiki/%E5%A0%86%E6%A0%88)是计算机科学中一种特殊的串列形式的抽象资料型别，其特殊之处在于只能允许在链接串列或阵列的一端（称为堆叠顶端指标，英语：top）进行加入数据（英语：push）和输出数据（英语：pop）的运算。另外栈也可以用一维数组或连结串列的形式来完成。堆叠的另外一个相对的操作方式称为伫列。
由于堆叠数据结构只允许在一端进行操作，因而按照后进先出（LIFO, Last In First Out）的原理运作。

举个例子，一把54式手枪的子弹夹，你往里面装子弹，最先射击出来的子弹肯定是最后装进去的那一个。
这就是栈的结构，后进先出。

![](https://upload-images.jianshu.io/upload_images/301129-8a44f3ede4d193de.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

栈中的每个元素称为一个`frame`。而最上层元素称为`top frame`。__栈只支持三个操作， `pop`, `top`, `push`。__

* pop取出栈中最上层元素(8)，栈的最上层元素变为早先进入的元素(9)。
* top查看栈的最上层元素(8)。
* push将一个新的元素(5)放在栈的最上层。

栈不支持其他操作。如果想取出元素12, 必须进行3次`pop`操作。

![](https://upload-images.jianshu.io/upload_images/301129-a8c2181c469689a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 内存分配中的栈和堆

堆栈空间分配

> 栈（操作系统）：由操作系统自动分配释放 ，存放函数的参数值，局部变量的值等。其操作方式类似于数据结构中的栈。
> 堆（操作系统）： 一般由程序员分配释放， 若程序员不释放，程序结束时可能由OS回收，分配方式倒是类似于链表。

堆栈缓存方式

> 栈使用的是一级缓存， 他们通常都是被调用时处于存储空间中，调用完毕立即释放。
> 堆则是存放在二级缓存中，生命周期由虚拟机的垃圾回收算法来决定（并不是一旦成为孤儿对象就能被回收）。所以调用这些对象的速度要相对来得低一些。

一般情况下程序存放在`Rom`（只读内存，比如硬盘）或`Flash`中，运行时需要拷到`RAM`（随机存储器RAM）中执行，`RAM`会分别存储不同的信息，如下图所示：

![](https://upload-images.jianshu.io/upload_images/301129-4965bcaf3749ddb7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

内存中的栈区处于相对较高的地址以地址的增长方向为上的话，栈地址是向下增长的。

栈中分配局部变量空间，堆区是向上增长的用于分配程序员申请的内存空间。另外还有静态区是分配静态变量，全局变量空间的；只读区是分配常量和程序代码空间的；以及其他一些分区。

也就是说，在`iOS`中，我们的值类型是放在栈空间的，内存分配和回收不需要我们关系，系统会帮我处理。在堆空间的对象类型就要有程序员自己分配，自己释放了。

### 引用计数

#### 引用计数是什么

> 引自维基百科[引用计数](https://zh.wikipedia.org/wiki/%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0)是计算机编程语言中的一种内存管理技术，是指将资源（可以是对象、内存或磁盘空间等等）的被引用次数保存起来，当被引用次数变为零时就将其释放的过程。使用引用计数技术可以实现自动资源管理的目的。同时引用计数还可以指使用引用计数技术回收未使用资源的垃圾回收算法。
当创建一个对象的实例并在堆上申请内存时，对象的引用计数就为1，在其他对象中需要持有这个对象时，就需要把该对象的引用计数加1，需要释放一个对象时，就将该对象的引用计数减1，直至对象的引用计数为0，对象的内存会被立刻释放。

正常情况下，当一段代码需要访问某个对象时，该对象的引用的计数加1；当这段代码不再访问该对象时，该对象的引用计数减1，表示这段代码不再访问该对象；当对象的引用计数为0时，表明程序已经不再需要该对象，系统就会回收该对象所占用的内存。

* 当程序调用方法名以`alloc`、`new`、`copy`、`mutableCopy`开头的方法来创建对象时，该对象的引用计数加`1`。
*  程序调用对象的`retain`方法时，该对象的引用计数加`1`。
*  程序调用对象的`release`方法时，该对象的引用计数减`1`。

`NSObject` 中提供了有关引用计数的如下方法：

*  —`retain`：将该对象的引用计数器加`1`。
*  —`release`：将该对象的引用计数器减`1`。
*  —`autorelease`：不改变该对象的引用计数器的值，只是将对象添加到自动释放池中。
*  —`retainCount`：返回该对象的引用计数的值。

#### 引用计数内存管理的思考方式

看到“引用计数”这个名称，我们便会不自觉地联想到“某处有某物多少多少”而将注意力放到计数上。但其实，更加客观、正确的思考方式：

* 自己生成的对象，自己持有。
* 非自己生成的对象，自己也能持有。
* 不再需要自己持有的对象时释放。
* 非自己持有的对象无法释放。

引用计数式内存管理的思考方式仅此而已。按照这个思路，完全不必考虑引用计数。
上文出现了“生成”、“持有”、“释放”三个词。而在`Objective-C`内存管理中还要加上“废弃”一词。各个词标书的`Objective-C`方法如下表。

| 对象操作 | Objective-C方法 |
| - | - |
| 生成并持有对象 | alloc/new/copy/mutableCopy等方法 |
| 持有对象 | retain方法 |
| 释放对象 | release方法 |
| 废弃对象 | dealloc方法 |

这些有关`Objective-C`内存管理的方法，实际上不包括在该语言中，而是包含在Cocoa框架中用于`macOS`、`iOS`应用开发。`Cocoa`框架中`Foundation`框架类库的`NSObject`类担负内存管理的职责。`Objective-C`内存管理中的`alloc/retain/release/dealloc`方法分别指代`NSObject`类的`alloc`类方法、`retain`实例方法、`release`实例方法和`dealloc`实例方法。

![](https://upload-images.jianshu.io/upload_images/301129-a5f25dd4c7a81260.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Cocoa框架、Foundation框架和NSObject类的关系

### MRC(手动管理引用计数)

顾名思义，`MRC`就是调用`Objective-C`的方法(`alloc/new/copy/mutableCopy/retain/release`等)实现引用计数的增加和减少。

下面通过`Objective-C`的方法实现内存管理的思考方式。

#### 自己生成的对象，自己持有

使用以下名称开头的方法名意味着自己生成的对象只有自己持有：

* alloc
* new
* copy
* mutableCopy

##### alloc的实现

```
// 自己生成并持有对象
id obj = [[NSObject alloc] init];
```

使用`NSObject`类的`alloc`方法就能自己生成并持有对象。指向生成并持有对象的指针被赋给变量`obj`。

##### new的实现

```
// 自己生成并持有对象
id obj = [NSObject new];
```

`[NSObject new]`与`[[NSObject alloc] init]`是完全一致的。

##### copy的实现

`copy`方法利用基于`NSCopying`方法约定，由各类实现的`copyWithZone:`方法生成并持有对象的副本。

```
#import "ViewController.h"

@interface Person: NSObject<NSCopying>

@property (nonatomic, strong) NSString *name;

@end

@implementation Person

- (id)copyWithZone:(NSZone *)zone {
    Person *obj = [[[self class] allocWithZone:zone] init];
    obj.name = self.name;
    return obj;
}

@end

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    //alloc生成并持有对象
    Person *p = [[Person alloc] init];
    p.name = @"testname";
    
    //copy生成并持有对象
    id obj = [p copy];
    
    //打印对象
    NSLog(@"p对象%@", p);
    NSLog(@"obj对象%@", obj);
}

@end
```

> 打印结果：
> 2018-03-28 23:01:32.321661+0800 ocram[4466:1696414] p对象<Person: 0x1c0003320>
2018-03-28 23:01:32.321778+0800 ocram[4466:1696414] obj对象<Person: 0x1c0003370>

从打印可以看到`obj`是`p`对象的副本。两者的引用计数都是`1`。

> 说明：在`- (id)copyWithZone:(NSZone *)zone`方法中，一定要通过`[self class]`方法返回的对象调用`allocWithZone:`方法。因为指针可能实际指向的是`Person`的子类。这种情况下，通过调用`[self class]`，就可以返回正确的类的类型对象。

##### mutableCopy的实现

与`copy`方法类似，`mutableCopy`方法利用基于`NSMutableCopying`方法约定，由各类实现的`mutableCopyWithZone:`方法生成并持有对象的副本。

```
#import "ViewController.h"

@interface Person: NSObject<NSMutableCopying>

@property (nonatomic, strong) NSString *name;

@end

@implementation Person

- (id)mutableCopyWithZone:(NSZone *)zone {
    Person *obj = [[[self class] allocWithZone:zone] init];
    obj.name = self.name;
    return obj;
}

@end

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    //alloc生成并持有对象
    Person *p = [[Person alloc] init];
    p.name = @"testname";
    
    //copy生成并持有对象
    id obj = [p mutableCopy];
    
    //打印对象
    NSLog(@"p对象%@", p);
    NSLog(@"obj对象%@", obj);
}

@end
```

> 打印结果：
> 2018-03-28 23:08:17.382538+0800 ocram[4476:1699096] p对象<Person: 0x1c4003c20>
2018-03-28 23:08:17.382592+0800 ocram[4476:1699096] obj对象<Person: 0x1c4003d70>

从打印可以看到`obj`是`p`对象的副本。两者的引用计数都是`1`。

`copy`和`mutableCopy`的区别在于，`copy`方法生成不可变更的对象，而`mutableCopy`方法生成可变更的对象。

##### 浅拷贝和深拷贝

既然讲到`copy`和`mutableCopy`，那就要谈一下深拷贝和浅拷贝的概念和实践。

###### 什么是浅拷贝、深拷贝？

> 简单理解就是，浅拷贝是拷贝了指向对象的指针， 深拷贝不但拷贝了对象的指针，还在系统中再分配一块内存，存放拷贝对象的内容。

浅拷贝：拷贝对象本身,返回一个对象,指向相同的内存地址。
深层复制：拷贝对象本身,返回一个对象,指向不同的内存地址。

###### 如何判断浅拷贝、深拷贝？

> 深浅拷贝取决于拷贝后的对象的是不是和被拷贝对象的地址相同，如果不同，则产生了新的对象，则执行的是深拷贝，如果相同，则只是指针拷贝，相当于retain一次原对象, 执行的是浅拷贝。

![](https://upload-images.jianshu.io/upload_images/655393-3639d66f6ecf3b97.png)

深拷贝和浅拷贝的判断要注意两点：

* 源对象类型是否是可变的
* 执行的拷贝是`copy`还是`mutableCopy`

###### 浅拷贝深拷贝的实现

* NSArray调用copy方法，浅拷贝

```
id obj = [NSArray array];
id obj1 = [obj copy];

NSLog(@"obj是%p", obj);
NSLog(@"obj1是%p", obj1);
```

> 打印结果：
> 2018-03-29 20:48:56.087197+0800 ocram[5261:2021415] obj是0x1c0003920
2018-03-29 20:48:56.087250+0800 ocram[5261:2021415] obj1是0x1c0003920

指针一样`obj`是浅拷贝。

* NSArray调用mutableCopy方法，深拷贝

```
id obj = [NSArray array];
id obj1 = [obj mutableCopy];

NSLog(@"obj是%p", obj);
NSLog(@"obj1是%p", obj1);
```

> 打印结果：
> 2018-03-29 20:42:16.508134+0800 ocram[5244:2018710] obj是0x1c00027d0
2018-03-29 20:42:16.508181+0800 ocram[5244:2018710] obj1是0x1c0453bf0

指针不一样`obj`是深拷贝。

* NSMutableArray调用copy方法，深拷贝

```
id obj = [NSMutableArray array];
id obj1 = [obj copy];

NSLog(@"obj是%p", obj);
NSLog(@"obj1是%p", obj1);
```

> 打印结果：
> 2018-03-29 20:50:36.936054+0800 ocram[5265:2022249] obj是0x1c0443f90
2018-03-29 20:50:36.936097+0800 ocram[5265:2022249] obj1是0x1c0018580

指针不一样`obj`是深拷贝。

* NSMutableArray调用mutableCopy方法，深拷贝

```
id obj = [NSMutableArray array];
id obj1 = [obj mutableCopy];

NSLog(@"obj是%p", obj);
NSLog(@"obj1是%p", obj1);
```

> 打印结果：
> 2018-03-29 20:52:30.057542+0800 ocram[5268:2023155] obj是0x1c425e6f0
2018-03-29 20:52:30.057633+0800 ocram[5268:2023155] obj1是0x1c425e180

指针不一样`obj`是深拷贝。

* 深拷贝的数组里面的元素依然是浅拷贝

```
id obj = [NSMutableArray arrayWithObject:@"test"];
id obj1 = [obj mutableCopy];

NSLog(@"obj是%p", obj);
NSLog(@"obj内容是%p", obj[0]);
NSLog(@"obj1是%p", obj1);
NSLog(@"obj1内容是%p", obj1[0]);
```

> 打印结果：
> 2018-03-29 20:55:18.196597+0800 ocram[5279:2025743] obj是0x1c0255120
2018-03-29 20:55:18.196647+0800 ocram[5279:2025743] obj内容是0x1c02551e0
2018-03-29 20:55:18.196665+0800 ocram[5279:2025743] obj1是0x1c0255210
2018-03-29 20:55:18.196682+0800 ocram[5279:2025743] obj1内容是0x1c02551e0

可以看到`obj`和`obj1`虽然指针是不一样的(深拷贝)，但是他们的元素的指针是一样的，__所以数组里的元素依然是浅拷贝__。

##### 其他实现

使用上述使用一下名称开头的方法，下面名称也意味着自己生成并持有对象。

* allocMyObject
* newThatObject
* copyThis
* mutableCopyYourObject

使用驼峰拼写法来命名。

```
#import "ViewController.h"

@interface Person: NSObject

@property (nonatomic, strong) NSString *name;

+ (id)allocObject;

@end

@implementation Person

+ (id)allocObject {
    //自己生成并持有对象
    id obj = [[Person alloc] init];
    
    return obj;
}

@end

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    //取得非自己生成并持有的对象
    Person *p = [Person allocObject];
    p.name = @"testname";
    
    NSLog(@"p对象%@", p);
}

@end
```

> 打印结果：
> 2018-03-28 23:33:37.044327+0800 ocram[4500:1706677] p对象<Person: 0x1c0013770>

`allocObject`名称符合上面的命名规则，因此它与用`alloc`方法生成并持有对象的情况完全相同，所以使用`allocObject`方法也意味着“自己生成并持有对象”。

#### 非自己生成的对象，自己也能持有

```
//非自己生成的对象，暂时没有持有
id obj = [NSMutableArray array];

//通过retain持有对象
[obj retain];
```
上述代码中`NSMutableArray`通过类方法`array`生成了一个对象赋给变量`obj`，但变量`obj`自己并不持有该对象。使用`retain`方法可以持有对象。

#### 不再需要自己持有的对象时释放

自己持有的对象，一旦不再需要，持有者有义务释放该对象。释放使用`release`方法。

##### 自己生成并持有对象的释放

```
// 自己生成并持有对象
id obj = [[NSObject alloc] init];

//释放对象
[obj release];
```

如此，用`alloc`方法由自己生成并持有的对象就通过`realse`方法释放了。自己生成而非自己所持有的对象，若用`retain`方法变为自己持有，也同样可以用`realse`方法释放。

##### 非自己生成的对象持有对象的释放

```
//非自己生成的对象，暂时没有持有
id obj = [NSMutableArray array];

//通过retain持有对象
[obj retain];

//释放对象
[obj release];
```

##### 非自己生成的对象本身的释放

像调用`[NSMutableArray array]`方法使取得的对象存在，但自己并不持有对象，是如何实现的呢？

```
+ (id)array {
    //生成并持有对象
    id obj = [[NSMutableArray alloc] init];
    
    //使用autorelease不持有对象
    [obj autorelease];
    
    //返回对象
    return obj;
}
```

上例中，我们使用了`autorelease`方法。用该方法，可以使取得的对象存在，但自己不持有对象。`autorelease`提供这样的功能，使对象在超出指定的生存范围时能够自动并正确的释放(调用`release`方法)。

![](https://upload-images.jianshu.io/upload_images/301129-20f13dc3018fcad6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在后面会对`autorelease`做更为详细的介绍。使用`NSMutableArray`类的`array`类方法等可以取得谁都不持有的对象，这些方法都是通过`autorelease`实现的。根据上文的命名规则，这些用来取得谁都不持有的对象的方法名不能以`alloc/new/copy/mutableCopy`开头，这点需要注意。

#### 非自己持有的对象无法释放

对于用`alloc/new/copy/mutableCopy`方法生成并持有的对象，或是用`retain`方法持有的对象，由于持有者是自己，所以在不需要该对象时需要将其释放。而由此以外所得到的对象绝对不能释放。倘若在程序中释放了非自己所持有的对象就会造成崩溃。

```
// 自己生成并持有对象
id obj = [[NSObject alloc] init];

//释放对象
[obj release];

//再次释放已经非自己持有的对象，应用程序崩溃
[obj release];
```

释放了非自己持有的对象，肯定会导致应用崩溃。因此绝对不要去释放非自己持有的对象。

#### autorelease

##### autorelease介绍

> 说到Objective-C内存管理，就不能不提autorelease。
> 顾名思义，autorelease就是自动释放。这看上去很像ARC，单实际上它更类似于C语言中自动变量（局部变量）的特性。

在C语言中，程序程序执行时，若局部变量超出其作用域，该局部变量将被自动废弃。

```
{
    int a;
}

//因为超出变量作用域，代码执行到这里，自动变量`a`被废弃，不可再访问。
```

`autorelease`会像`C`语言的局部变量那样来对待对象实例。当其超出作用域时，对象实例的`release`实例方法被调用。另外，同`C`语言的局部变量不同的是，编程人员可以设置变量的作用域。

`autorelease`的具体使用方法如下：

* 生成并持有`NSAutoreleasePool`对象。
* 调用已分配对象的`autorelease`实例方法。
* 废弃`NSAutoreleasePool`对象。

![](https://upload-images.jianshu.io/upload_images/301129-82a4644dbb5acb35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

id obj = [[NSObject alloc] init];

[obj autorelease];

[pool drain];
```

上述代码中最后一行的`[pool drain]`等同于`[obj release]`。

##### autorelease实现

调用`NSObject`类的`autorelease`实例方法。

```
[obj autorelease];
```

调用`autorelease`方法的内部实现

```
- (id) autorelease {
    [NSAutoreleasePool addObject: self];
}
```

`autorelease`实例方法的本质就是调用`NSAutoreleasePool`对象的`addObject`类方法。

##### autorelease注意

`autorelease`是`NSObject`的实例方法，`NSAutoreleasePool`也是继承`NSObject`的类。那能不能调用`autorelease`呢？

```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

[pool release];
```

运行结果发生崩溃。通常在使用`Objective-C`，也就是`Foundation`框架时，无论调用哪一个对象的`autorelease`实例方法，实现上是调用的都是`NSObject`类的`autorelease`实例方法。但是对于`NSAutoreleasePool`类，`autorelease`实例方法已被该类重载，因此运行时就会出错。

### ARC(自动管理引用计数)

#### ARC介绍

> 上面讲了“引用计数内存管理的思考方式”的本质部分在ARC中并没有改变。就像“自动引用计数”这个名称表示的那样，ARC只是自动地帮助我们处理“引用计数”的相关部分。

在编译单位上，可设置`ARC`有效或无效，即设置特定类是否启用`ARC`。
在`Project`里面找到`Build Phases`-`Compile Sources`，这里是所有你的编译文件。指定编译器属性为`-fobjc-arc`即为该文件使用`ARC`，指定编译器属性为`-fno-objc-arc`即为该文件不使用`ARC`，如下图所示。

![](https://upload-images.jianshu.io/upload_images/301129-44a7bd5fad885e51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编译器在编译时会帮我们自动插入，包括 `retain`、`release`、`copy`、`autorelease`、`autoreleasepool`

#### ARC有效的代码实现

##### 所有权修饰符

Objective-C编程中为了处理对象，可将变量类型定义为id类型或各种对象类型。 ARC中，id类型和对象类其类型必须附加所有权修饰符。

* __strong修饰符
* __weak修饰符
* __unsafe_unretained修饰符
* __autoreleasing修饰符

######  __strong修饰符

`__strong`修饰符是`id`类型和对象类型默认的所有权修饰符。也就是说，不写修饰符的话，默认对象前面被附加了`__strong`所有权修饰符。

```
id obj = [[NSObject alloc] init];
等同于
id __strong obj = [[NSObject alloc] init];
```

`__strong`修饰符的变量`obj`在超出其变量作用域时，即在该变量被废弃时，会释放其被赋予的对象。
`__strong`修饰符表示对对象的“强引用”。持有强引用的变量在超出其作用域时被废弃，随着强引用的失效，引用的对象会随之释放。

当然，`__strong`修饰符也可以用在Objective-C类成员变量和方法参数上。

```
@interface Test: NSObject
{
    id __strong obj_;
}

- (void)setObject:(id __strong)obj;

@end

@implementation Test

- (instancetype)init {
    self = [super init];
    return self;
}

- (void)setObject:(id __strong)obj {
    obj_ = obj
}

@end
```

无需额外的工作便可以使用于类成员变量和方法参数中。`__strong`修饰符和后面要讲的`__weak`修饰符和`__autoreleasing`修饰符一起，可以保证将附有这些修饰符的自动变量初始化为`nil`。

正如苹果宣称的那样，通过`__strong`修饰符再键入`retain`和`release`，完美地满足了“引用计数式内存管理的思考方式”。

######  __weak修饰符

通过`__strong`修饰符并不能完美的进行内存管理，这里会发生“循环引用”的问题。

![](https://upload-images.jianshu.io/upload_images/301129-1eeccd858df16410.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过上面的例子代码实现循环引用。

```
{
      id test0 = [[Test alloc] init];
      id test1 = [[Test alloc] init];
      [test0 setObject:test1];
      [test1 setObject:test0];
}
```

可以看到`test0`和`tets1`互相持有对方，谁也释放不了谁。

循环引用容易发生内存泄露。所谓内存泄露就是应当废弃的对象在超出其生命周期后继续存在。

`__weak`修饰符可以避免循环引用，与`__strong`修饰符相反，提供弱引用。弱引用不能持有对象实例，所以在超出其变量作用域时，对象即被释放。像下面这样将之前的代码修改，就可以避免循环引用了。

```
@interface Test: NSObject
{
    id __weak obj_;
}

- (void)setObject:(id __strong)obj;
```

使用`__weak`修饰符还有另外一个优点。在持有某对象的弱引用时，若该对象被废弃，则此弱引用将自动失效且处于nil赋值的状态(空弱引用)。

```
id __weak obj1 = nil;
{
    id __strong obj0 = [[NSObject alloc] init];
    
    obj1 = obj0;
    
    NSLog(@"%@", obj1);
}

NSLog(@"%@", obj1);
```

> 打印结果：
> 2018-03-30 21:47:50.603814+0800 ocram[51624:22048320] <NSObject: 0x60400001ac10>
2018-03-30 21:47:50.604038+0800 ocram[51624:22048320] (null)

可以看到因为`obj0`超出作用域就被释放了，弱引用也被至为`nil`状态。

######  __unsafe_unretained修饰符

`__unsafe_unretained`修饰符是不安全的修饰符，尽管`ARC`式的内存管理是编译器的工作，但附有`__unsafe_unretained`修饰符的变量不属于编译器的内存管理对象。`__unsafe_unretained`和`__weak`一样不能持有对象。

```
id __unsafe_unretained obj1 = nil;
{
    id __strong obj0 = [[NSObject alloc] init];
    
    obj1 = obj0;
    
    NSLog(@"%@", obj1);
}

NSLog(@"%@", obj1);
```

> 打印结果：
> 2018-03-30 21:58:28.033250+0800 ocram[51804:22062885] <NSObject: 0x604000018e80>

可以看到最后一个打印没有打印出来，程序崩溃了。这是因为超出了作用域，`obj1`已经变成了一个野指针，然后我们去操作野指针的时候会发生崩溃。

所以在使用`__unsafe_unretained`修饰符时，赋值给`__strong`修饰符的变量时有必要确保被赋值的对象确实存在。

######  __autoreleasing修饰符

在`ARC`中，我也可以使用`autorelease`功能。指定“`@autoreleasepool`块”来代替“`NSAutoreleasePool`类对象生成、持有以及废弃这一范围，使用附有`__autoreleasing`修饰符的变量替代`autorelease`方法。

![](https://upload-images.jianshu.io/upload_images/301129-96a7718c97b89c88.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实我们不用显示的附加 `__autoreleasing`修饰符，这是由于编译器会检查方法名是否以`alloc/new/copy/mutableCopy`开始，如果不是则自动将返回值的对象注册到`autoreleasepool`。

有时候`__autoreleasing`修饰符要和`__weak`修饰符配合使用。

```
id __weak obj1 = obj0;

id __autoreleasing tmp = obj1;
```

为什么访问附有`__weak`修饰符的变量时必须访问注册到`autoreleasepool`的对象呢？这是因为`__weak`修饰符只持有对象的弱引用，而在访问引用对象的过程中，该对象有可能被废弃。如果把访问的对象注册到`autoreleasepool`中，那么在`@autoreleasepool`块结束之前都能确保该对象存在。

###### 属性与所有权修饰符的对应关系

![](https://upload-images.jianshu.io/upload_images/301129-87c4982ef025d1bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上各种属性赋值给指定的属性中就相当于赋值给附加各属性对应的所有权修饰符的变量中。只有`copy`不是简单的赋值，它赋值的是通过`NSCopying`接口的`copyWithZone：`方法复制赋值源所生成的对象。


##### ARC规则

在`ARC`有效的情况下编译源代码，必须遵守一定的规则。

###### 不能使用`retain/release/retainCount/autorelease`

`ARC`有效时，实现`retain/release/retainCount/autorelease`会引起编译错误。代码会标红，编译不通过。

###### 不能使用`NSAllocateObject/NSDeallocateObject`

###### 须遵守内存管理的方法命名规则

`alloc`,`new`,`copy`,`mutableCopy`,`init`
以`init`开始的方法的规则要比`alloc`,`new`,`copy`,`mutableCopy`更严格。该方法必须是实例方法，并且要返回对象。返回的对象应为`id`类型或方法声明类的对象类型，抑或是该类的超类型或子类型。该返回对象并不注册到`autoreleasepool`上。基本上只是对`alloc`方法返回值的对象进行初始化处理并返回该对象。

```
//符合命名规则
- (id) initWithObject;

//不符合命名规则
- (void) initThisObject;
```

###### 不要显式调用`dealloc`

当对象的引用计数为`0`，所有者不持有该对象时，该对象会被废弃，同时调用对象的`dealloc`方法。`ARC`会自动对此进行处理，因此不必书写`[super dealloc]`。

###### 使用`@autoreleasepool`块替代`NSAutoreleasePool`

###### 不能使用区域(NSZone)

###### 对象型变量不能作为C语言结构体（struct、union）的成员

C语言结构体（struct、union）的成员中，如果存在Objective-C对象型变量，便会引起编译错误。

```
struct Data {
    NSMutableArray *array;
};
```

> 显示警告:
> ARC forbids Objective-C objects in struct

在`C`语言的规约上没有方法来管理结构体成员的生命周期。因为`ARC`把内存管理的工资分配给编译器，所以编译器必须能够知道并管理对象的生命周期。例如`C`语言的局部变量可使用该变量的作用域管理对象。但是对于`C`语言的结构体成员来说，这在标准上就是不可实现的。

要把对象类型添加到结构体成员中，可以强制转换为`void *`或是附加`__unsafe_unretained`修饰符。

```
struct Data {
    NSMutableArray __unsafe_unretained *array;
};
```

`__unsafe_unretained`修饰符的变量不属于编译器的内存管理对象。如果管理时不注意赋值对象的所有者，便可能遭遇内存泄露或者程序崩溃。

###### 显示转换`id`和`void *`

在MRC时，将`id`变量强制转换`void *`变量是可以的。

```
id obj = [[NSObject alloc] init];

void *p = obj;

id o = p;

[o release];
```

但是在ARC时就会编译报错，`id`型或对象型变量赋值给`void *`或者逆向赋值时都需要进行特定的转换。如果只想单纯的赋值，则可以使用“`__bridge`转换”

__bridge转换中还有另外两种转换，分部是“`__bridge_retained`”和“`__bridge_transfer`转换”
`__bridge_retained`转换与`retain`类似，`__bridge_transfer`转换与release类似。

```
void *p = (__bridge_retained void *)[[NSObject alloc] init];
NSLog(@"class = %@", [(__bridge id)p class]);
(void)(__bridge_transfer id)p;
```

### ARC内存的泄露和检测

#### ARC内存泄露常见场景

##### 对象型变量作为C语言结构体（struct、union）的成员

```
struct Data {
    NSMutableArray __unsafe_unretained *array;
};
```

`__unsafe_unretained`修饰符的变量不属于编译器的内存管理对象。如果管理时不注意赋值对象的所有者，便可能遭遇内存泄露或者程序崩溃。

##### 循环引用

循环引用常见有三种现象：

* 两个对象互相持有对象，这个可以设置弱引用解决。

```
@interface Test: NSObject
{
    id __weak obj_;
}

- (void)setObject:(id __strong)obj;
```

* block持有self对象，这个要在block块外面和里面设置弱引用和强引用。

```
__weak __typeof(self) wself = self;
obj.block = ^{
    __strong __typeof(wself) sself = wself;
    
    [sself updateSomeThing];
}
```

* NSTimer的target持有self

> NSTimer会造成循环引用，timer会强引用target即self，一般self又会持有timer作为属性，这样就造成了循环引用。
那么，如果timer只作为局部变量，不把timer作为属性呢？同样释放不了，因为在加入runloop的操作中，timer被强引用。而timer作为局部变量，是无法执行invalidate的，所以在timer被invalidate之前，self也就不会被释放。

##### 单例属性不释放

严格来说这个不算是内存泄露，主要就是我们在单例里面设置一个对象的属性，因为单例是不会释放的，所以单例会有一直持有这个对象的引用。

```
[Instanse shared].obj = self;
```

可以看到单例持有了当前对象`self`，这个`self`就不会释放了。

#### ARC内存泄露的检测

##### 使用Xcode自带工具Instrument

打开Xcode8自带的Instruments

![](https://upload-images.jianshu.io/upload_images/2818477-4c925fd56aa30c0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/443)

或者

![](https://upload-images.jianshu.io/upload_images/2818477-c3386cd774a4cfde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/193)

或者：长按运行按钮，然后出现如图所示列表，点击Profile.

![](https://upload-images.jianshu.io/upload_images/2818477-37a2090a6db18898.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/330)

按上面操作，build成功后跳出Instruments工具，选择Leaks选项

![](https://upload-images.jianshu.io/upload_images/2818477-c8179c9996436732.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

选择之后界面如下图:

到这里之后,我们前期的准备工作做完啦，下面开始正式的测试!

（有一个注意的点，最好选择真机进行测试，模拟器是运行在mac上的，mac跟手机还是有区别的嘛。）

1.选中Xcode先把程序（command + R）运行起来（如果Xcode左上角已经是instrument的图标就不用执行这一步了）

2.再选中Xcode，按快捷键（command + control + i）运行起来,此时Leaks已经跑起来了

3.由于Leaks是动态监测，所以我们需要手动操作APP,一边操作，一边观察Leaks的变化，当出现红色叉时，就监测到了内存泄露，点击左上角的第二个，进行暂停检测(也可继续检测).如图所示:

![](https://upload-images.jianshu.io/upload_images/2818477-8ebb314d64b4c207.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

4.下面就是定位修改了,此时选中有红色柱子的Leaks，下面有个"田"字方格，点开，选中Call Tree

![](https://upload-images.jianshu.io/upload_images/2818477-c1248053d6897fa9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/490)

显示如下图界面

![](https://upload-images.jianshu.io/upload_images/2818477-bb00b42e42279f2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

5.下面就是最关键的一步，在这个界面的右下角有若干选框，选中Invert Call Tree 和Hide System Libraries,（红圈范围内）显示如下：

![](https://upload-images.jianshu.io/upload_images/2818477-50539494769c5296.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/257)

如果右下角找不到此设置窗口，可以在底部点击Call Tree，显示如下：

![](https://upload-images.jianshu.io/upload_images/2818477-decf4632e5d23123.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/384)

到这里就算基本完成啦，这里显示的就是内存泄露代码部分，那么现在还差一步：定位!

6.选中显示的若干条中的一条，双击，会自动跳到内存泄露代码处，如图所示

![](https://upload-images.jianshu.io/upload_images/2818477-c4f169d6942022c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

在选择call tree后，可能你会发现查看不到源码从而无法定位内存泄漏的位置，只是显示16进制的数据。此时需要你在Xcode中检查是否有dSYM File生成，如下图所示选择第二项DWARF with dSYM File.

![](https://upload-images.jianshu.io/upload_images/2818477-523c269e5b63f8f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

##### 在对象dealloc中进行打印

我们生成的对象，在即将释放的时候，会调用`dealloc`方法。所以我们可以在`dealloc`打印当前对象已经释放的消息。如果没有释放，对象的`dealloc`方法就永远不会执行，此时我们知道发生了内存泄露。

通过这个思路，我写了一个小工具用来检查当前`controller`没有释放的，然后打印出来。

[写个简单的Swift检测Controller没有销毁的工具](http://jackyshan.github.io/2018/01/27/%E5%86%99%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84Swift%E6%A3%80%E6%B5%8BController%E6%B2%A1%E6%9C%89%E9%94%80%E6%AF%81%E7%9A%84%E5%B7%A5%E5%85%B7/)
