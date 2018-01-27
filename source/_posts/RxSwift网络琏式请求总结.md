---
title: RxSwift网络琏式请求总结
date: 2017-11-02 21:47:15
tags: RxSwift
---

![img](http://upload-images.jianshu.io/upload_images/301129-4a564e26f450160f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[RxSwift](https://github.com/ReactiveX/RxSwift)是在 Apple 推出 [Swift](https://swift.org/) 后， [ReactiveX](http://reactivex.io/) 推出 Reactive Extensions 系列一个实现库。下面介绍工作中使用RxSwift解决异步网络请求场景的实践。

![img](http://upload-images.jianshu.io/upload_images/301129-6b84c6fed13e445f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/301129-40324ccaf01621c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

业务场景为根据当前定位位置请求附近公交、地铁、水巴数据。根据右侧的筛选项筛选出不同的数据，进行请求。

### 创建数据事件源Observable

* 请求地铁网络数据


    func searchSubwayData(_ location: CLLocationCoordinate2D, _ isUserLocation: Bool = false) -> Observable<Bool> {
        return Observable<Bool>.create { [weak self] (observer) -> Disposable in
            let send = SubwaySendModel()
            send.latitude = location.latitude
            send.longitude = location.longitude
            send.range = 500
            SubwayNetwork.This.doTask(SubwayNetwork.CMD_getByCoord, data: send, controller: nil, success: {[weak self] (response: SubwayResponseModel?) in
                observer.onCompleted()
                guard let resp = response else {return}
                guard resp.line.count > 0 else {return}
                if isUserLocation == true {
                    self?.subWayResponseModel = nil
                    self?.subWayResponseModel = response
                }
                guard resp.station.count > 0 else {return}
                guard let stations: [SubwayStationModel] = (resp.station as NSArray).jsonArray() else {return}
                
                for model in stations {
                    let annotation = RyHomeMAPointAnnotation(.subway)
                    annotation.title = model.name
                    annotation.coordinate = JZLocationConverter.wgs84(toGcj02: CLLocationCoordinate2D.init(latitude: CLLocationDegrees(model.latitude), longitude: CLLocationDegrees(model.longitude)))
                    let nearStationModel = RyHomeNearStationModel()
                    nearStationModel.name = model.name
                    nearStationModel.desc = model.lines
                    let nlocation = CLLocation.init(latitude: annotation.coordinate.latitude, longitude: annotation.coordinate.longitude)
                    nearStationModel.sid = "\(model.id)"
                    nearStationModel.mtype = .subway
                    if let userLocation = self?.mapView.userLocation.location {
                        nearStationModel.locationDistance = userLocation.distance(from: nlocation)
                    }
                    nearStationModel.location = nlocation
                    annotation.mStationModel = nearStationModel
                    self?.nearStationModelArr.append(nearStationModel)
                    self?.subwayAnnotations.append(annotation)
                }
                self?.mapView.addAnnotations(self?.subwayAnnotations)
                
                }, error: { [weak self] (err, msg) in
                    self?.subWayResponseModel = nil
                    observer.onError(TestError.test)
                }, com: nil, showWait: false)
            
            return Disposables.create()
        }
    }

* 请求水巴网络数据


    func searchBoatData(_ location: CLLocationCoordinate2D, _ isUserLocation: Bool = false) -> Observable<Bool> {
        return Observable<Bool>.create { [weak self] (observer) -> Disposable in
            let send = WaterBusSendModel()
            send.latitude = location.latitude
            send.longitude = location.longitude
            send.range = 500
            
            WaterBusNetWork.This.doArrayTask(WaterBusNetWork.CMD_getByCoord, data: send, controller: nil, success: { [weak self] (response: [ShuiBaStationModel]?) in
                observer.onCompleted()
                guard let stations = response else {return}
                guard stations.count > 0 else {return}
                if isUserLocation == true {
                    self?.shuibaResponseArr = nil
                    self?.shuibaResponseArr = stations
                }
                
                for model in stations {
                    let annotation = RyHomeMAPointAnnotation(.boat)
                    annotation.title = model.n
                    annotation.coordinate = JZLocationConverter.wgs84(toGcj02: CLLocationCoordinate2D.init(latitude: CLLocationDegrees(model.la), longitude: CLLocationDegrees(model.lo)))
                    let nearStationModel = RyHomeNearStationModel()
                    nearStationModel.name = model.n
                    nearStationModel.desc = "途径\(model.rcount ?? "0")条线路"
                    let nlocation = CLLocation.init(latitude: annotation.coordinate.latitude, longitude: annotation.coordinate.longitude)
                    nearStationModel.sid = model.i
                    nearStationModel.mtype = .boat
                    if let userLocation = self?.mapView.userLocation.location {
                        nearStationModel.locationDistance = userLocation.distance(from: nlocation)
                    }
                    nearStationModel.location = nlocation
                    annotation.mStationModel = nearStationModel
                    self?.nearStationModelArr.append(nearStationModel)
                    self?.boatAnnotations.append(annotation)
                }
                self?.mapView.addAnnotations(self?.boatAnnotations)
                
                }, error: { [weak self] (_, _) in
                    self?.shuibaResponseArr = nil
                    observer.onError(TestError.test)
                }, com: nil, showWait: false)
            
            return Disposables.create()
        }
    }


* 请求公交站网络数据


    func searchBusStationData(_ location: CLLocationCoordinate2D) -> Observable<Bool> {
        return Observable<Bool>.create { [weak self] (observer) -> Disposable in
            let send = SendGetFullStationByCoord()
            send.latitude = location.latitude
            send.longitude = location.longitude
            send.range = 500
            send.withLCheck = true
            BusNetWork.This.doArrayTask(BusNetWork.CMD_getByCoord, data: send, controller: self, success: { (responseObj:[GetFullStationByCoord]?) in
                observer.onCompleted()
                guard let stations = responseObj else {return}
                guard stations.count > 0 else {return}
                
                for model in stations {
                    let annotation = RyHomeMAPointAnnotation(.busStation)
                    annotation.title = model.n
                    annotation.coordinate = JZLocationConverter.wgs84(toGcj02: CLLocationCoordinate2D.init(latitude: CLLocationDegrees(model.la), longitude: CLLocationDegrees(model.lo)))
                    let nearStationModel = RyHomeNearStationModel()
                    nearStationModel.name = model.n
                    nearStationModel.desc = "途径\(model.c)条线路"
                    let nlocation = CLLocation.init(latitude: annotation.coordinate.latitude, longitude: annotation.coordinate.longitude)
                    nearStationModel.sid = model.i
                    nearStationModel.mtype = .busStation
                    nearStationModel.linec = Int(model.c)
                    if let userLocation = self?.mapView.userLocation.location {
                        nearStationModel.locationDistance = userLocation.distance(from: nlocation)
                    }
                    nearStationModel.location = nlocation
                    annotation.mStationModel = nearStationModel
                    self?.nearStationModelArr.append(nearStationModel)
                    self?.busStationAnnotations.append(annotation)
                }
                self?.mapView.addAnnotations(self?.busStationAnnotations)
                
            }, error: { (_, _) in
                observer.onError(TestError.test)
            }, com: nil, showWait: false)
            
            return Disposables.create()
        }
        
    }

代码中`Observable<Bool>.create {  }`是创建一个被观察者，`observer.onCompleted() `是网络数据请求回来，当前观察者发送完成信号，开始处理下一个事件，`observer.onError(TestError.test) `是网络请求失败，发送失败信号，使整个事件序列重新开始请求。

Observable就是可被观察的，是事件源，可以被订阅，上面创建了三个Observable，下面我们可以通过concat，把创建的三个事件联合起来一起被订阅。

### 联合订阅事件源

    let disposeBag = DisposeBag()
    func getData(_ isUserLocation: Bool = false) {
        ryHomeRightCategoryView.updateFilterIcon(filterType)
        
        guard let location = movedLocationCoordinate else {return}
        
        mapView.removeOverlays(mapView.overlays)
        addOverlay(location)
        mapView.removeAnnotations(subwayAnnotations)
        mapView.removeAnnotations(boatAnnotations)
        mapView.removeAnnotations(busStationAnnotations)
        subwayAnnotations.removeAll()
        boatAnnotations.removeAll()
        busStationAnnotations.removeAll()
        nearStationModelArr.removeAll()
        
        let wgsLocation = JZLocationConverter.gcj02(toWgs84: location)
        
        let symbol1 = searchSubwayData(wgsLocation)
        let symbol2 = searchBoatData(wgsLocation)
        let symbol3 = searchBusStationData(wgsLocation)
        
        var symbols: Observable<Observable<Bool>>? = nil
        if filterType == .subway {
            symbols = Observable.of(symbol1)
        }
        else if filterType == .boat {
            symbols = Observable.of(symbol2)
        }
        else if filterType == .bus {
            symbols = Observable.of(symbol3)
        }
        else if filterType == .all {
            symbols = Observable.of(symbol1, symbol2, symbol3)
        }
        symbols?.concat().retry(2).subscribe().addDisposableTo(disposeBag)
    }

这个方法，通过concat把几个事件源联合起来进行订阅，就实现了多个网络的链式顺序请求。
* 创建了地铁、水巴、公交站台三个被观察者的对象，这样方便后面进行筛选数据。


    let symbol1 = searchSubwayData(wgsLocation)
    let symbol2 = searchBoatData(wgsLocation)
    let symbol3 = searchBusStationData(wgsLocation)

* 创建事件源序列对象`var symbols: Observable<Observable<Bool>>? = nil ` 根据当前的筛选类型`filterType`，给`symbols`进行赋值


    var symbols: Observable<Observable<Bool>>? = nil
    if filterType == .subway {
        symbols = Observable.of(symbol1)
    }
    else if filterType == .boat {
        symbols = Observable.of(symbol2)
    }
    else if filterType == .bus {
        symbols = Observable.of(symbol3)
    }
    else if filterType == .all {
        symbols = Observable.of(symbol1, symbol2, symbol3)
    }

* 联合订阅事件，把当前事件添加到释放池`disposeBag `，方便当前页面被释放后，被订阅的事件可以被释放。


    symbols?.concat().retry(2).subscribe().addDisposableTo(disposeBag)

### 参考

*  [RxSwift入坑解读-你所需要知道的各种概念](http://www.codertian.com/2016/11/27/RxSwift-ru-keng-ji-read-document/)
* [RxSwift 入坑手册 Part0 - 基础概念](https://blog.callmewhy.com/2015/09/21/rxswift-getting-started-0/)