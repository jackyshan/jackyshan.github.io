---
title: NSDictionary底层实现原理
date: 2018-05-23 20:57:08
tags: iOS
---

![](https://upload-images.jianshu.io/upload_images/301129-ff337872b5a8a2f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### NSDictionary介绍

> NSDictionary（字典）是使用 hash表来实现key和value之间的映射和存储的， hash函数设计的好坏影响着数据的查找访问效率。数据在hash表中分布的越均匀，其访问效率越高。而在Objective-C中，通常都是利用NSString 来作为键值，其内部使用的hash函数也是通过使用 NSString对象作为键值来保证数据的各个节点在hash表中均匀分布。

### NSDictionary内部结构

NSDictionary使用NSMapTable实现，NSMapTable同样是一个key－value的容器。

```
typedef struct {
   NSMapTable        *table;
   NSInteger          i;
   struct _NSMapNode *j;
} NSMapEnumerator;
```

上述结构体描述了遍历一个NSMapTable时的一个指针对象，其中包含table对象自身的指针，计数值，和节点指针。

```
typedef struct {
   NSUInteger (*hash)(NSMapTable *table,const void *);
   BOOL (*isEqual)(NSMapTable *table,const void *,const void *);
   void (*retain)(NSMapTable *table,const void *);
   void (*release)(NSMapTable *table,void *);
   NSString  *(*describe)(NSMapTable *table,const void *);
   const void *notAKeyMarker;
} NSMapTableKeyCallBacks;
```

上述结构体中存放的是几个函数指针，用于计算key的hash值，判断key是否相等，retain，release操作。

```
typedef struct {
   void       (*retain)(NSMapTable *table,const void *);
   void       (*release)(NSMapTable *table,void *);
   NSString  *(*describe)(NSMapTable *table, const void *);
} NSMapTableValueCallBacks;
```

上述存放的三个函数指针，定义在对NSMapTable插入一对key－value时，对value对象的操作。

```
@interface NSMapTable : NSObject {
   NSMapTableKeyCallBacks   *keyCallBacks;
   NSMapTableValueCallBacks *valueCallBacks;
   NSUInteger             count;
   NSUInteger             nBuckets;
   struct _NSMapNode  **buckets;
}
```

可以看出来NSMapTable是一个哈希＋链表的数据结构，因此在NSMapTable中插入或者删除一对对象时，
寻找的时间是O（1）＋O（m），m最坏时可能为n。

* O（1）：为对key进行hash得到bucket的位置
* O（m）：遍历该bucket后面冲突的value，通过链表连接起来。

因此NSDictionary中的Key-Value遍历时是无序的，至如按照什么样的顺序，跟hash函数相关。NSMapTable使用NSObject的哈希函数。

```
- (NSUInteger)hash {
   return (NSUInteger)self>>4;
}
```

上述是NSObject的哈希值的计算方式，简单通过移位实现。右移4位，左边补0。因为对象大多存于堆中，地址相差4位应该很正常。

### NSDictionary使用

#### NSDictionary的Key值

NSDictionary中最常用的一个方法原型：

```
- (void)setObject:(id)anObject forKey:(id <NSCopying>)aKey;  
```

从这个方法中可以知道，要作为Key值，必须遵循NSCopying协议。也就是说在NSDictionary内部，会对aKey对象Copy一份新的。而anObject 对象在其内部是作为强引用（retain或strong)。所以在MRC下，向该方法发送消息之后，我们会向anObject发送release消息进行释放。

既然知道了作为key值，必须遵循NSCopying协议，说明除了NSString对象之外，我们还可以使用其他类型对象来作为NSDictionary的 key值。不过这还不够，作为key值，该类型还必须继承于NSObject并且要重载一下两个方法：

```
- (NSUInteger)hash;  
- (BOOL)isEqual:(id)object;  
```

其中，hash 方法是用来计算该对象的 hash 值，最终的 hash 值决定了该对象在 hash 表中存储的位置。所以同样，如果想重写该方法，我们尽量设计一个能让数据分布均匀的 hash 函数。

__所以如果对象key的hash值相同，那在hash表里面的对应的value值是相同的__(value值被更新了)

isEqual方法是为了通过hash值来找到对象在hash表中的位置。

KeyObject.h文件：

```
//
//  KeyObject.h
//  testD
//
//  Created by jackyshan on 2018/5/22.
//  Copyright © 2018年 GCI. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface KeyObject : NSObject<NSCopying>//实现Copying协议

- (instancetype)initWithHashI:(NSUInteger)i;

@end
```

KeyObject.m文件

```
//
//  KeyObject.m
//  testD
//
//  Created by jackyshan on 2018/5/22.
//  Copyright © 2018年 GCI. All rights reserved.
//

#import "KeyObject.h"

@interface KeyObject() {
    NSUInteger _hashValue;
}

@end

@implementation KeyObject

- (instancetype)initWithHashI:(NSUInteger)i {
    if (self = [super init]) {
        _hashValue = i;
    }
    
    return self;
}

#pragma mark - overload method

- (BOOL)isEqual:(id)object {
    //根据hash值判断是否是同一个键值
    return ([self hashKeyValue] == [(typeof(self))object hashKeyValue]);
}

- (NSUInteger)hash {
    //返回hash值
    return [self hashKeyValue];
}

#pragma mark - NSCopying
- (id)copyWithZone:(NSZone *)zone {
    KeyObject *obj = [KeyObject allocWithZone:zone];
    obj->_hashValue = _hashValue;
    return obj;
}

#pragma mark - private method

- (NSUInteger)hashKeyValue {
    return _hashValue % 3;
}

@end
```

测试

```
//
//  main.m
//  testD
//
//  Created by jackyshan on 2018/5/22.
//  Copyright © 2018年 GCI. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "KeyObject.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        
        NSMutableDictionary *dic = [NSMutableDictionary dictionary];
        
        KeyObject *key1 = [[KeyObject alloc] initWithHashI:1];
        [dic setObject:@"key1value" forKey:key1];
        
        KeyObject *key2 = [[KeyObject alloc] initWithHashI:4];
        [dic setObject:@"key2value" forKey:key2];
        
        KeyObject *key3 = [[KeyObject alloc] initWithHashI:8];
        [dic setObject:@"key3value" forKey:key3];
        
        NSLog(@"key1--->%@", [dic objectForKey:key1]);
        NSLog(@"key1--->%@", [dic objectForKey:key2]);
        NSLog(@"key1--->%@", [dic objectForKey:key3]);
        
        NSLog(@"allKeys--->%@", [dic allKeys]);
        NSLog(@"allValues--->%@", [dic allValues]);
    }
    return 0;
}
```

> 打印结果：
> 2018-05-22 15:40:06.429379+0800 testD[6350:1441941] key1--->key2value
2018-05-22 15:40:06.454114+0800 testD[6350:1441941] key1--->key2value
2018-05-22 15:40:06.454137+0800 testD[6350:1441941] key1--->key3value
2018-05-22 15:40:06.454634+0800 testD[6350:1441941] allKeys--->(
    "<KeyObject: 0x100403440>",
    "<KeyObject: 0x100406b10>"
)
2018-05-22 15:40:06.454779+0800 testD[6350:1441941] allValues--->(
    key2value,
    key3value
)
Program ended with exit code: 0


从打印结果来看，key1和key2对象的hash值是一样的，所以他们value是相等的，key1的value被key2更新了。


#### NSDictionary的KVC实现

```
@implementation NSDictionary (NSKeyValueCoding)
- (id)valueForKey:(NSString*)key {
    if([key hasPrefix:@"@"])
        return [super valueForKey:[key substringFromIndex:1]];
    return [self objectForKey:key];
}

- (void)setValue:(id)value forKey:(NSString*)key {
    [NSException raise:NSInvalidArgumentException format:@"%@ called on immutable dictionary %@", NSStringFromSelector(_cmd), self];
}

@end


@implementation NSMutableDictionary (NSKeyValueCoding)

- (void)setValue:(id)value forKey:(NSString*)key {
    if(value)
        [self setObject:value forKey:key];
    else
        [self removeObjectForKey:key];
}

@end
```

* setValue和setObject的区别

```
- (void)setObject:(ObjectType)anObject forKey:(KeyType <NSCopying>)aKey;
- (void)setValue:(nullable ObjectType)value forKey:(NSString *)key;
```

`setObject: ForKey`:是NSMutableDictionary特有的；`setValue: ForKey:`是KVC的主要方法。


> (1) setValue: ForKey:的value是可以为nil的（但是当value为nil的时候，会自动调用removeObject：forKey方法）；
setObject: ForKey:的value则不可以为nil。
(2) setValue: ForKey:的key必须是不为nil的字符串类型；
setObject: ForKey:的key可以是不为nil的所有继承NSCopying的类型。