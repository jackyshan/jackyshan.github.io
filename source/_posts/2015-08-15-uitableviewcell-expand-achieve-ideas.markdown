---
layout: post
title: "UITableViewCell通讯录展开四级实现思路"
date: 2015-08-15 22:17:46 +0800
comments: true
categories: iOS
---

![](/images/contacts_expand.png)

##Model
```
@interface StreetArea : NSObject

@property (nonatomic, strong) NSString *street;//街道名称
@property (nonatomic, strong) NSString *areaId;//区id

@end

@protocol StreetArea

@end

@interface DistrictArea : NSObject

@property (nonatomic, strong) NSString *district;//区域名称
@property (nonatomic, strong) NSArray<StreetArea> *streetList;//街道列表
@property (nonatomic, assign) BOOL selected;//被选中

@end

@protocol DistrictArea

@end

@interface CityArea : NSObject

@property (nonatomic, strong) NSString *city;//城市名称
@property (nonatomic, strong) NSArray<DistrictArea> *districtList;//区域列表
@property (nonatomic, assign) BOOL selected;//被选中

@end

@protocol CityArea

@end

@interface ProvinceArea : NSObject

@property (nonatomic, strong) NSString *province;//省级名称
@property (nonatomic, strong) NSArray<CityArea> *cityList;//城市列表
@property (nonatomic, assign) BOOL selected;//被选中

@end
```

__省市区街道__

省ProvinceArea的model包含城市列表cityList，市CityArea的model包含区域列表districtList

##实现
```
_contactsArr = [NSMutableArray array];
```
初始化一个可变数组


```
if (cityModel.selected) {//被选中了，哈哈
            if (cityModel.districtList.count <= 0) {
                cityModel.selected = !cityModel.selected;
                NSLog(@"无区级列表!");return;
            }
            [_contactsArr insertObjects:cityModel.districtList atIndexes:sets];
            [tableView beginUpdates];
            [tableView insertRowsAtIndexPaths:paths withRowAnimation:UITableViewRowAnimationMiddle];
            [tableView endUpdates];
            [tableView reloadRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationFade];
            [tableView scrollToRowAtIndexPath:paths[cityModel.districtList.count-1]
                             atScrollPosition:UITableViewScrollPositionBottom animated:YES];
        }
        else //没有选中，😄
        {
            [_contactsArr removeObjectsAtIndexes:sets];
            [tableView beginUpdates];
            [tableView deleteRowsAtIndexPaths:paths withRowAnimation:UITableViewRowAnimationMiddle];
            [tableView endUpdates];
            [tableView reloadRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationFade];
        }
```
每个model加了一个selected字段，判断该model是否selected，_contactsArr数组根据model的selected相应删除或添加model的list数组。

`层数越多，判断逻辑越复杂`

`上面的一串的代码只是最简单的两层折叠`

##源码
<https://github.com/jackyshan/ContactsExpand>