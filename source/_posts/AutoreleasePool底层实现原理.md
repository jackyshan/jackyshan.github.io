---
title: AutoreleasePool底层实现原理
date: 2018-05-23 20:59:52
tags: AutoreleasePool
---

![](https://upload-images.jianshu.io/upload_images/301129-da9595ea4430a011.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> AutoreleasePool（自动释放池）是OC中的一种内存自动回收机制，它可以延迟加入AutoreleasePool中的变量release的时机。在正常情况下，创建的变量会在超出其作用域的时候release，但是如果将变量加入AutoreleasePool，那么release将延迟执行。


### AutoreleasePool创建和释放

* App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。
* 第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。
* 第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。
* 在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

也就是说AutoreleasePool创建是在一个RunLoop事件开始之前(push)，AutoreleasePool释放是在一个RunLoop事件即将结束之前(pop)。
AutoreleasePool里的Autorelease对象的加入是在RunLoop事件中，AutoreleasePool里的Autorelease对象的释放是在AutoreleasePool释放时。

### AutoreleasePool实现原理


在终端中使用clang -rewrite-objc命令将下面的OC代码重写成C++的实现：

```
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    
    @autoreleasepool {
        NSLog(@"Hello, World!");
    }
 
    return 0;
}
```

在cpp文件代码中我们找到main函数代码如下：

```
int main(int argc, const char * argv[]) {

    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_kb_06b822gn59df4d1zt99361xw0000gn_T_main_d39a79_mi_0);
    }

    return 0;
}
```

可以看到苹果通过声明一个__AtAutoreleasePool类型的局部变量__autoreleasepool实现了@autoreleasepool{}。
`__AtAutoreleasePool`的定义如下:

```
extern "C" __declspec(dllimport) void * objc_autoreleasePoolPush(void);
extern "C" __declspec(dllimport) void objc_autoreleasePoolPop(void *);

struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

根据构造函数和析构函数的特点（自动局部变量的构造函数是在程序执行到声明这个对象的位置时调用的，而对应的析构函数是在程序执行到离开这个对象的作用域时调用），我们可以将上面两段代码简化成如下形式：

```
int main(int argc, const char * argv[]) {

    /* @autoreleasepool */ {
        void *atautoreleasepoolobj = objc_autoreleasePoolPush();
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_kb_06b822gn59df4d1zt99361xw0000gn_T_main_d39a79_mi_0);
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }

    return 0;
}
```

至此，我们可以分析出，单个自动释放池的执行过程就是`objc_autoreleasePoolPush()` —> `[object autorelease]` —> `objc_autoreleasePoolPop(void *)`。

来看一下objc_autoreleasePoolPush 和 objc_autoreleasePoolPop 的实现：

```
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```

上面的方法看上去是对 AutoreleasePoolPage 对应静态方法 push 和 pop 的封装。
下面分析一下AutoreleasePoolPage的实现，揭开AutoreleasePool的实现原理。

### AutoreleasePoolPage实现

#### AutoreleasePoolPage介绍

AutoreleasePoolPage 是一个 C++ 中的类，它在 NSObject.mm 中的定义是这样的：

```
class AutoreleasePoolPage {
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

#   define POOL_BOUNDARY nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);

    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};
```

* magic 检查校验完整性的变量
* next 指向新加入的autorelease对象
* thread page当前所在的线程，AutoreleasePool是按线程一一对应的（结构中的thread指针指向当前线程）
* parent 父节点 指向前一个page
* child 子节点 指向下一个page
* depth 链表的深度，节点个数
* hiwat high water mark 数据容纳的一个上限
* EMPTY_POOL_PLACEHOLDER 空池占位
* POOL_BOUNDARY 是一个边界对象 nil,之前的源代码变量名是 POOL_SENTINEL哨兵对象,用来区别每个page即每个 AutoreleasePoolPage 边界
* PAGE_MAX_SIZE = 4096, 为什么是4096呢？其实就是虚拟内存每个扇区4096个字节,[4K对齐](https://zh.wikipedia.org/zh-hans/4K%E5%AF%B9%E9%BD%90)的说法。
* COUNT 一个page里对象数

#### 双向链表

AutoreleasePool并没有单独的结构，而是由若干个AutoreleasePoolPage以双向链表的形式组合而成的栈结构（分别对应结构中的parent指针和child指针）

![](https://upload-images.jianshu.io/upload_images/3672149-7ea8e16cedb69d1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/526)

parent和child就是用来构造双向链表的指针。parent指向前一个page, child指向下一个page。
一个AutoreleasePoolPage的空间被占满时，会新建一个AutoreleasePoolPage对象，连接链表，后来的autorelease对象在新的page加入。

#### objc_autoreleasePoolPush

![](https://upload-images.jianshu.io/upload_images/3672149-25e788aede2fe5d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

每当自动释放池调用objc_autoreleasePoolPush时都会把边界对象放进栈顶,然后返回边界对象,用于释放。

```
atautoreleasepoolobj = objc_autoreleasePoolPush();
```

atautoreleasepoolobj就是返回的边界对象(POOL_BOUNDARY)

push实现如下：

```
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}
```

它调用AutoreleasePoolPage的类方法push：

```
static inline void *push() {
   return autoreleaseFast(POOL_BOUNDARY);
}
```

在这里会进入一个比较关键的方法autoreleaseFast，并传入边界对象(POOL_BOUNDARY)：

```
static inline id *autoreleaseFast(id obj)
{
   AutoreleasePoolPage *page = hotPage();
   if (page && !page->full()) {
       return page->add(obj);
   } else if (page) {
       return autoreleaseFullPage(obj, page);
   } else {
       return autoreleaseNoPage(obj);
   }
}
```

上述方法分三种情况选择不同的代码执行：

* 有 hotPage 并且当前 page 不满，调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中
* 有 hotPage 并且当前 page 已满，调用 autoreleaseFullPage 初始化一个新的页，调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中
* 无 hotPage，调用 autoreleaseNoPage 创建一个 hotPage，调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中

最后的都会调用 page->add(obj) 将对象添加到自动释放池中。
hotPage 可以理解为当前正在使用的 AutoreleasePoolPage。

#### AutoreleasePoolPage::autorelease(id obj)

autorelease方法的实现，先来看一下方法的调用栈：

```
- [NSObject autorelease]
└── id objc_object::rootAutorelease()
    └── id objc_object::rootAutorelease2()
        └── static id AutoreleasePoolPage::autorelease(id obj)
            └── static id AutoreleasePoolPage::autoreleaseFast(id obj)
                ├── id *add(id obj)
                ├── static id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
                │   ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                │   └── id *add(id obj)
                └── static id *autoreleaseNoPage(id obj)
                    ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                    └── id *add(id obj)
```

在autorelease方法的调用栈中，最终都会调用上面提到的 autoreleaseFast方法，将当前对象加到AutoreleasePoolPage 中。

这一小节中这些方法的实现都非常容易，只是进行了一些参数上的检查，最终还要调用autoreleaseFast方法：

```
inline id objc_object::rootAutorelease() {
    if (isTaggedPointer()) return (id)this;
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;

    return rootAutorelease2();
}

__attribute__((noinline,used)) id objc_object::rootAutorelease2() {
    return AutoreleasePoolPage::autorelease((id)this);
}

static inline id autorelease(id obj) {
   id *dest __unused = autoreleaseFast(obj);
   return obj;
}
```

autorelease函数和push函数一样，关键代码都是调用autoreleaseFast函数向自动释放池的链表栈中添加一个对象，
不过push函数的入栈的是一个边界对象，而autorelease函数入栈的是需要加入autoreleasepool的对象。

#### objc_autoreleasePoolPop

![](https://upload-images.jianshu.io/upload_images/3672149-8c3a6a9f0f117355.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/618)

自动释放池释放是传入 push 返回的边界对象,

```
objc_autoreleasePoolPop(atautoreleasepoolobj);
```

然后将边界对象指向的这一页 AutoreleasePoolPage 内的对象释放
atautoreleasepoolobj就是返回的边界对象(POOL_BOUNDARY)

AutoreleasePoolPage::pop()实现：

```
static inline void pop(void *token)   // token指针指向栈顶的地址
{
    AutoreleasePoolPage *page;
    id *stop;

    page = pageForPointer(token);   // 通过栈顶的地址找到对应的page
    stop = (id *)token;
    if (DebugPoolAllocation  &&  *stop != POOL_SENTINEL) {
        // This check is not valid with DebugPoolAllocation off
        // after an autorelease with a pool page but no pool in place.
        _objc_fatal("invalid or prematurely-freed autorelease pool %p; ", 
                    token);
    }

    if (PrintPoolHiwat) printHiwat();   // 记录最高水位标记

    page->releaseUntil(stop);   // 从栈顶开始操作出栈，并向栈中的对象发送release消息，直到遇到第一个哨兵对象

    // memory: delete empty children
    // 删除空掉的节点
    if (DebugPoolAllocation  &&  page->empty()) {
        // special case: delete everything during page-per-pool debugging
        AutoreleasePoolPage *parent = page->parent;
        page->kill();
        setHotPage(parent);
    } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
        // special case: delete everything for pop(top) 
        // when debugging missing autorelease pools
        page->kill();
        setHotPage(nil);
    } 
    else if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

该过程主要分为两步：

* page->releaseUntil(stop)，对栈顶（page->next）到stop地址（POOL_SENTINEL）之间的所有对象调用objc_release()，进行引用计数减1
* 清空page对象page->kill()，有两句注释

```
// hysteresis: keep one empty child if this page is more than half full

// special case: delete everything for pop(0)
除非是pop(0)方式调用，这样会清理掉所有page对象；
否则，在当前page存放的对象大于一半时，会保留一个空的子page，
这样估计是为了可能马上需要新建page节省创建page的开销。
```

#### 小结

* 自动释放池是一个个 AutoreleasePoolPage 组成的一个page是4096字节大小,每个 AutoreleasePoolPage 以双向链表连接起来形成一个自动释放池
* 当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中
* pop 时是传入边界对象,然后对page 中的对象发送release 的消息
