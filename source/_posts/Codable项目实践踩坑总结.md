---
title: Codable项目实践踩坑总结
date: 2018-06-03 15:54:48
tags: iOS
---

![](https://upload-images.jianshu.io/upload_images/301129-60f9048f7f5c4135.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 项目情况

在Swift2.3的时候就已经开始项目的整体Swift实现了。因为当时没有比较好用的Model，就使用OC的JSONModel实现Model的转换，Model还是用Swift建立，继承JSONModel实现字典转模型、数组转模型等一系列的序列化操作。

现在项目升级到Swift4.1，由于Swift4的语言特性造成了这样的Swift写的Model继承OC，已经无法使用了。现在市面上各种Model序列化的第三方库也慢慢发展成熟了，所以也打算整体更换成全部Swift实现的Model了。因为Codable是苹果自己的，所以率先选择了这个框架使用Model的整体迁移。

### 封装Codable

苹果也是为了自己推出的序列化比较安全，常用接口的实现还比较缓慢，各种常用功能也都支持的不是很完全，比如字典转模型、模型转字典，要写好几行代码来实现，这对于工程项目是不可接受的，所以下面的代码封装了JBaseModel，实现了常用的这些接口。

```
//
//  JBaseModel.swift
//  renttravel
//
//  Created by jackyshan on 2018/5/30.
//  Copyright © 2018年 GCI. All rights reserved.
//

protocol JBaseModel: Codable {
    
}

extension JBaseModel {
    //模型转字典
    func toDictionary() -> [String:Any] {
        guard let data = try? JSONEncoder().encode(self) else {
            return [String:Any]()
        }
        
        guard let dict = (try? JSONSerialization.jsonObject(with: data, options: .allowFragments)) as? [String: Any] else {
            return [String:Any]()
        }
        
        return dict
    }
    
    //字典转模型
    static func initt<T:JBaseModel>(dictionary dict:[String:Any],_ type:T.Type) throws -> T {
        var newDict = dict
        if dict is [String: AnyCodable] {
            let dd: [String: AnyCodable] = dict as! [String : AnyCodable]
            newDict = Dictionary.init(uniqueKeysWithValues: dd.map({key, value in (key, value.value)}))
        }
        
        guard let JSONString = newDict.toJSONString() else {
            print("MapError.dictToJsonFail")
            throw MapError.dictToJsonFail
        }
        guard let jsonData = JSONString.data(using: .utf8) else {
            print(MapError.jsonToDataFail)
            throw MapError.jsonToDataFail
        }

        let decoder = JSONDecoder()
//        if let obj = try? decoder.decode(type, from: jsonData) {
//            return obj
//        }
//        print(MapError.jsonToModelFail)
//        throw MapError.jsonToModelFail
        
        return try! decoder.decode(type, from: jsonData)
    }
    
    //Json转模型
    static func initt<T:JBaseModel>(string jsonSting: String, error: Error?, _ type:T.Type) throws -> T {
        guard let jsonData = jsonSting.data(using: .utf8) else {
            print(MapError.jsonToDataFail)
            throw MapError.jsonToDataFail
        }
        let decoder = JSONDecoder()
//        if let obj = try? decoder.decode(type, from: jsonData) {
//            return obj
//        }
//        print(MapError.jsonToModelFail)
//        throw MapError.jsonToModelFail
        
        return try! decoder.decode(type, from: jsonData)
    }
    
    //model转json 字符串
    func toJSONString() -> String {
        guard let data = try? JSONEncoder().encode(self) else {
            return ""
        }
        
        guard let str = String.init(data: data, encoding: .utf8)?.replacingOccurrences(of: "\\/", with: "/") else {
            return ""
        }
        
        return str
    }
}

enum MapError: Error {
    case jsonToModelFail    //json转model失败
    case jsonToDataFail     //json转data失败
    case dictToJsonFail     //字典转json失败
    case jsonToArrFail      //json转数组失败
    case modelToJsonFail    //model转json失败
}

extension Dictionary {
    func toJSONString() -> String? {
        if !JSONSerialization.isValidJSONObject(self) {
            print("dict 转 json 失败")
            return nil
        }
        if let newData:Data = try? JSONSerialization.data(withJSONObject: self, options: []) {
            guard let JSONString = String.init(data: newData, encoding: .utf8)?.replacingOccurrences(of: "\\/", with: "/") else {
                return ""
            }
            
            return JSONString
        }
        print("dict 转 json 失败")
        return nil
    }
}

extension Array {
    func toJSONString() -> String? {
        if !JSONSerialization.isValidJSONObject(self) {
            print("dict转json失败")
            return nil
        }
        if let newData : Data = try? JSONSerialization.data(withJSONObject: self, options: []) {
            guard let JSONString = String.init(data: newData, encoding: .utf8)?.replacingOccurrences(of: "\\/", with: "/") else {
                return ""
            }
            
            return JSONString
        }
        print("dict转json失败")
        return nil
    }
    static func initt<T:Decodable>(string jsonString:String,_ type:[T].Type) throws -> Array<T> {
        guard let JSonData = jsonString.data(using: .utf8) else {
            print(MapError.jsonToDataFail)
            throw MapError.jsonToDataFail
        }
        let decoder = JSONDecoder()
//        if let obj = try? decoder.decode(type, from: JSonData) {
//            return obj
//        }
//        print(MapError.jsonToArrFail)
//        throw MapError.jsonToArrFail
        
        return try! decoder.decode(type, from: JSonData)
    }
}

extension String {
    func toDictionary() -> [String:Any]? {
        guard let jsonData:Data = self.data(using: .utf8) else {
            print("json转dict失败")
            return nil
        }
        if let dict = try? JSONSerialization.jsonObject(with: jsonData, options: .mutableContainers) {
            return dict as? [String : Any] ?? ["":""]
        }
        print("json转dict失败")
        return nil
    }
}
```

上面代码中引用到了[AnyCodable](https://github.com/Flight-School/AnyCodable)，需要另行下载。

### 总结

根据踩坑经验，Codable有下面几种常见的踩坑情况，周知：
* 继承Codable的类，属性都要实现Codable，所以Any不能用

```
class UserModel: Codable {
    var name: Any?
}
```
Any是没有实现Codable的，可以根据`mattt`大神封装的[AnyCodable](https://github.com/Flight-School/AnyCodable)写成下面这样

```
class UserModel: Codable {
    var name: AnyCodable?
}
```

AnyCodable是继承Codable的，实现常用类型的Codable协议

* Codable不能实现动态类型转换

例如后台返回json如下：

```
"user": "{"price": 100}"
```
由于这个price我是用来显示的，不需要进行计算的操作，很多人会吧price声明称String类型

```
class User: Codable {
      var price: String?
}
```

这样写Codable解析Model是会失败的，因为根据返回的数据，price应该声明为Int类型

```
class User: Codable {
      var price: Int?
}
```

* 利用AnyCodable实现Dictionary转JSON，要进行二次过滤

```
let dict: [String: AnyEncodable] = [
    "boolean": true,
    "integer": 1,
    "double": 3.14159265358979323846,
    "string": "string",
    "array": [1, 2, 3],
    "nested": [
        "a": "alpha",
        "b": "bravo",
        "c": "charlie"
    ]
]
```

判断dictionary是否可以JSON解析

```
JSONSerialization.isValidJSONObject(dict)
```
返回false

解决方法

```
var newDict = dict
if dict is [String: AnyCodable] {
    let dd: [String: AnyCodable] = dict as! [String : AnyCodable]
    newDict = Dictionary.init(uniqueKeysWithValues: dd.map({key, value in (key, value.value)}))
}
```

我在AnyCodable的官方库上面提出了toJSON的问题，并给出了自己的[答案](https://github.com/Flight-School/AnyCodable/issues/7)，具体的解决方法实现细节，可以参考这个答案。

