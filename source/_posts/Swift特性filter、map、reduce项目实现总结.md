---
title: Swift特性filter、map、reduce项目实现总结
date: 2017-07-19 19:56:54
tags: Swift
---

![image](http://upload-images.jianshu.io/upload_images/301129-38444aaf5a34490d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### filter
filter实现功能是，对符合条件的数组内的元素过滤出来，组成一个新的数组

`场景1`
扫描蓝牙设备，对蓝牙设备进行分类，major为1是站台，为0是公交，蓝牙设备返回模型为

	@interface IbeaconModel : NSObject

	@property (nonatomic,assign) int major;
	@property (nonatomic,assign) int minor;

	@end

使用filter过滤出站台和公交的设备

	let ls = self?.ibeaconArr?.filter({$0.major == 1})
	let lb = self?.ibeaconArr?.filter({$0.major == 0})

`场景2`
司机反馈界面，选中内容之后再次点击内容按钮反选内容，这样就要把append的模型进行删除掉，使用reduce可以很简单实现remove模型的效果

	checkedList = checkedList.filter({$0 !== model)

### map
map实现功能是，对数组的元素进行处理，返回一个新的数组

`场景`
把上面蓝牙过滤出的站台和公交数组的元素的minor转化成string数组

	let ls = searchFilterArr.filter({$0.major == 1}).map({String($0.minor)})
	let lb = searchFilterArr.filter({$0.major == 0}).map({String($0.minor)})

### reduce
`场景1`
司机反馈页面，对选中的反馈内容列表进行过滤id，传给后台所有选中的id并且以逗号分格，反馈内容的模型是

	@interface DriverFeedbackModel : BaseModel

	/** id */
	@property (nonatomic, strong) NSString *id;

	/** name */
	@property (nonatomic, strong) NSString *name;

	@end

使用reduce过滤出id，组成字符串

    func getTidString() -> String {
        var str:String = ""
        str = checkedList.reduce("", {$0 + $1.id + ","})
        
        return str
    }

`场景2`
对高德地铁线路数组和后台返回的地铁线路列表进行过滤，找出高德地铁第几条线路的index在后台返回的线路列表里的Index是第几个

	let bus = busstops![idx]
	let line = lineModel[0]
	let stations: [SubwayLineStationModel] = (line.stations as NSArray).jsonArray()!
	idx = stations.reduce(0) { (n, smodel) -> Int in
	    var m = n
	    if (bus.name as NSString).isEqual(to: smodel.station) {
	        m = stations.index(of: smodel)!
	    }
	    
	    return m
	}