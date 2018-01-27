---
title: iOS蓝牙扫描总结
date: 2017-07-19 19:43:36
tags: BLE
---

![image](http://upload-images.jianshu.io/upload_images/301129-eb3ed120ab90c15c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

蓝牙低能耗(BLE)技术是低成本、短距离、可互操作的鲁棒性无线技术，工作在免许可的2.4GHz ISM射频频段。它从一开始就设计为超低功耗(ULP)无线技术。它利用许多智能手段最大限度地降低功耗。蓝牙低能耗技术采用可变连接时间间隔，这个间隔根据具体应用可以设置为几毫秒到几秒不等。另外，因为BLE技术采用非常快速的连接方式，因此平时可以处于“非连接”状态（节省能源），此时链路两端相互间只是知晓对方，只有在必要时才开启链路，然后在尽可能短的时间内关闭链路。

### 初始化CBCentralManager
    func start() {
          stop()
          centerManager = CBCentralManager(delegate: self, queue: dispatch_get_main_queue())
    }

### 实现CBCentralManagerDelegate代理方法
检测到蓝牙打开，开始扫描设备

    // MARK: CBCentralManagerDelegate
    func centralManagerDidUpdateState(central: CBCentralManager) {
        if central.state != .PoweredOn {
            switch central.state {
            case .Unknown:
                print("蓝牙未知原因")
                break
            case .Resetting:
                print("蓝牙连接超时")
                break
            case .Unsupported:
                print("不支持蓝牙4.0")
                break
            case .Unauthorized:
                print("连接失败")
                break
            case .PoweredOff:
                print("蓝牙未开启")
                break
            
            default:
                break
            }
            
            print(">>>设备不支持BLE或者未打开")
            bleOpenFail()
            stop()
        }
        else {
            print(">>>BLE状态正常")
            bleOpenSucc()
            centerManager!.scanForPeripheralsWithServices([CBUUID(string: "60000-B5F3-F3F3-F0A9-E50F24DCCF9E")], options: [CBCentralManagerScanOptionAllowDuplicatesKey:false])
    //            centerManager!.scanForPeripheralsWithServices(nil, options: [CBCentralManagerScanOptionAllowDuplicatesKey:false])
          }
      }

扫描到设备，开始连接设备

      func centralManager(central: CBCentralManager, didDiscoverPeripheral peripheral: CBPeripheral, advertisementData: [String : AnyObject], RSSI: NSNumber)  {
        print(">>>>扫描周边设备 .. 设备id:\(peripheral.identifier.UUIDString), rssi: \(RSSI.stringValue), advertisementData: \(advertisementData), peripheral: \(peripheral)")
      peripheral.delegate = self
      centerManager?.connectPeripheral(peripheral, options: nil)
    }

### 实现CBPeripheralDelegate代理方法

连接到设备，开始扫描设备上的服务

    func centralManager(central: CBCentralManager, didConnectPeripheral peripheral: CBPeripheral) {
        print("和周边设备连接成功。")
        peripheral.discoverServices(nil)
        print("扫描周边设备上的服务..")
    }
    
发现设备上的服务，开始扫描服务携带的特性

    func peripheral(peripheral: CBPeripheral, didDiscoverServices error: NSError?) {
        if error != nil {
            print("发现服务时发生错误: \(error)")
            return
        }
        
        print("发现服务 ..")
        
        for service in peripheral.services! {
            peripheral.discoverCharacteristics(nil, forService: service)
        }
    }
    
发现设备特性，所以信息获得完毕

    func peripheral(peripheral: CBPeripheral, didDiscoverCharacteristicsForService service: CBService, error: NSError?) {
        print("发现服务 \(service.UUID), 特性数: \(service.characteristics?.count)")
        
        for c in service.characteristics! {
            if let data = c.value {
                peripheral.readValueForCharacteristic(c)
                print("特性值byte： \(data.bytes)")
                print("特性值string： \(String(data: data, encoding: NSUTF8StringEncoding))")
            }
        }
    }
    
### 停止蓝牙扫描

    func stop() {
        bleViewerPerArr.removeAll()
        centerManager?.stopScan()
        centerManager = nil
    }

代码参考
https://github.com/jackyshan/bleIbeaconscanner/blob/master/BLEScanner.swift
