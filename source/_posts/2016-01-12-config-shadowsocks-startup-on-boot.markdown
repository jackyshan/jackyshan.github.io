---
layout: post
title: "shadowsocks安装及开机启动配置"
date: 2016-01-12 13:52:55 +0800
comments: true
categories: summary
---
## 安装

	git clone https://github.com/jackyshan/shadowsocks
	cd shadowsocks/
	python setup.py install

## 配置
### 编辑/etc/shadowsocks.json
	vi /etc/shadowsocks.json
	
	{
	
		"server":"0.0.0.0",
	
		"timeout":300,
	
		"method":"aes-256-cfb",
	
		"port_password":
	
		{
	
		"8383":"password1",
	
		"8384":"password2",
	
		"8385":"password3"
	
		}
	
	}


### 编辑shadowsocks.conf
	vi /etc/init/shadowsocks.conf
	
	#shadowscks - Shadowsocks Server Service
	#
	
	start on runlevel [2345]
	stop on runlevel [!2345]
	
	respawn

	exec /usr/local/bin/ssserver -c /etc/shadowsocks.json


## 启动
	/usr/local/bin/ssserver -c /etc/shadowsocks.json




