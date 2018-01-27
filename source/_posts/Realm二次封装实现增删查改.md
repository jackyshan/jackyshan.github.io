---
title: Realm二次封装实现增删查改
date: 2017-07-19 19:58:13
tags: Realm
---

![image](http://upload-images.jianshu.io/upload_images/301129-07248be050f001b0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Realm 是 SQLite 和 Core Data 的替代者，得益于其零拷贝的设计，Realm 比任何 ORM 都要快很多。几分钟内就能学会使用 Realm。

封装了基础类`DBBaseObject`，基础类实现了增删查改的方法，所有数据Object都继承DBBaseObject
新建`Dog`类，继承DBBaseObject

	class Dog: DBBaseObject {
	    dynamic var name = ""
	    dynamic var age = 0
	}

### 增
	let dog = Dog()
	dog.name = "xiaogou"
	dog.add()

### 查
	let dogs = Dog.query(Dog.self, "name = 'xiaogou'")
	dogs.count

### 改
	if let dog = dogs.first {
		Dog.update {
			dog.name = "dagou"
		}
	}

### 删
	if let dog = dogs.first {
		dog.delete()
	}

### DBBaseObject实现
	//
	//  DBBaseObject.swift
	//  DianZiCheng
	//
	//  Created by jackyshan on 2017/5/28.
	//  Copyright © 2017年 jackyshan. All rights reserved.
	//

	import UIKit

	class DBBaseObject: Object {
	    dynamic var note1: String?//备份
	    dynamic var note2: String?//备份
	    
	    func add() {//增
	        let realm = try! Realm()
	        try! realm.write {
	            realm.add(self)
	        }
	    }
	    
	    func delete() {//删
	        let realm = try! Realm()
	        try! realm.write {
	            realm.delete(self)
	        }
	    }
	    
	    static func query<T>(_ type: T.Type, _ filterStr: String? = nil) -> Results<T> {//查
	        let realm = try! Realm()
	        
	        if let str = filterStr {
	            return realm.objects(type).filter(str)
	        }
	        else {
	            return realm.objects(type)
	        }
	    }
	    
	    //多线程查询
	    static func query<T>(_ type: T.Type, _ filterStr: String? = nil, block:@escaping ((_ results: Results<T>)->Void)) {
	        DispatchQueue(label: "background").async {
	            let realm = try! Realm()
	            
	            if let str = filterStr {
	                let objects = realm.objects(type).filter(str)
	                block(objects)
	            }
	            else {
	                let objects = realm.objects(type)
	                block(objects)
	            }
	        }
	    }
	    
	    static func update(_ block: (() throws -> Swift.Void)) {//改
	        let realm = try! Realm()
	        try! realm.write(block)
	    }
	}