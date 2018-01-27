---
title: iOS设备iBeacon扫描总结
date: 2017-07-19 19:51:02
tags: BLE
---

![image](http://upload-images.jianshu.io/upload_images/301129-70278ebff12be093.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

iBeacon是苹果公司提出的“一种可以让附近手持电子设备检测到的一种新的低功耗、低成本信号传送器”的一套可用于室内定位系统的协议。 这种技术可以使一个智能手机或其他装置在一个iBeacon基站的感应范围内执行相应的命令

### 初始化CLLocationManager和CLBeaconRegion

    let locationManager = CLLocationManager()
    
    let beaconRegion: CLBeaconRegion = {
        let region = CLBeaconRegion(proximityUUID: NSUUID(UUIDString: "7B1C1C64-077E-4D23-9F49-7E644A13B5A9")!, identifier: "7B1C1C64-077E-4D23-9F49-7E644A13B5A9")
        region.notifyEntryStateOnDisplay = true
        return region
    }()

### 开始扫描

    func start() {
        stop()
        
        locationManager.delegate = self
        locationManager.startRangingBeaconsInRegion(beaconRegion)
    }
    
### 实现CLLocationManagerDelegate代理方法

打印授权状态

    func locationManager(manager: CLLocationManager, didChangeAuthorizationStatus status: CLAuthorizationStatus) {
        if status == .AuthorizedAlways {
            print("Location Access (Always) granted!")
        } else if status == .AuthorizedWhenInUse {
            print("Location Access (When In Use) granted!")
        } else if status == .Denied || status == .Restricted {
            print("Location Access (When In Use) denied!")
        }
    }
    
扫描到设备，accuracy实际距离

    func locationManager(manager: CLLocationManager, didRangeBeacons beacons: [CLBeacon], inRegion region: CLBeaconRegion) {
        print(">>>发现设备: becons: \(beacons)")
        print("------------------------------")
        for bc in beacons {
            print (bc.accuracy)
        }
    }

### 停止扫描
    
    func stop() {
        locationManager.stopRangingBeaconsInRegion(beaconRegion)
        locationManager.delegate = nil
        
    }

代码参考
https://github.com/jackyshan/bleIbeaconscanner/blob/master/IBeaconScanner.swift