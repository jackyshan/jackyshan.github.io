---
layout: post
title: "设置GoAgent在Mac上开机自启动"
date: 2015-08-05 23:07:34 +0800
comments: true
categories: summary
---

##Automator
1. 打开Automator，选择应用程序
	
	![](/images/automator_app.png)

2. 选择运行脚本

	```	
	输入python /Users/apple/Downloads/goagent/local/proxy.py
	python 后跟goagent的路径
	```
	
	![](/images/automator_choise_shell.png)
	
3. 保存为GoAgent.App
	
	```
	保存在应用程序目录
	```
	
	![](/images/automator_goagent.png)
	
##系统设置
1. 打开用户和群组
	
	![](/images/add_app_users.png)
	
2. 点击左下角+号，选取GoAgent应用程序，添加
	
	![](/images/add_goagent_in_users.png)