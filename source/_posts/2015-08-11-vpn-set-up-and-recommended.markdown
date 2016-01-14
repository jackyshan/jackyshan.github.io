---
layout: post
title: "VPN网络加速器和搭建"
date: 2015-08-11 22:48:24 +0800
comments: true
categories: summary
---

如果一个人使用vpn，经济点的方法就直接去买网络加速器。或者，如果有几个人使用，一起买个服务器，自己搭建vpn，几个人平摊上网也比较划算。

##推荐

* [蓝灯](https://getlantern.org/)`网络加速，类似shadowsocks`**重点推荐**

>
优点：免费
>
缺点：不能修改代理配置文件，没有iOS端

* [红杏](http://honx.in/_VLu0DM6vDwo3sn9N)`网络加速器`
	
>
优点：浏览器代理网络加速，只能浏览器上网，速度很快稳定
>  
缺点：只限于浏览器，要在浏览器上装个红杏插件

* [GreenVpn](http://gjsq.me/5204478)`vpn代理`

>
优点：vpn代理，支持iOS，Android，PC客户端，价格适中
>
缺点：不是很稳定，有时会断开，断开可以重连

* [天涯vpn](http://www.tianyavpn.org/)`vpn代理`

>
优点：支持iOS，Android，PC，速度相对快点，价格比greenvpn贵点
>
缺点：断线之后要等几分钟以后才能重连


##搭建VPN&ShadowSocks

* [购买linode服务器](https://www.linode.com/?r=b39c08dca47d86cf1f09ab57c897e1a4f59a4b68)
 			
       购买Linode 1GB的那种，一个月10$，换算成人民币大概也就60元
 	   能够凑齐6个人，每个人每月10元，比购买网络加速器还要便宜

* 地点选择Fremont，CA `速度还可以`

* 服务器使用ubuntu

>
[UBUNTU、CENTOS搭建IPSEC/IKEV2 VPN服务器全攻略](http://quericy.me/blog/512)
>
[各平台VPS快速搭建SHADOWSOCKS及优化总结](http://quericy.me/blog/495)


    
  
