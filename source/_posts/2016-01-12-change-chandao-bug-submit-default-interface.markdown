---
layout: post
title: "禅道bug提交界面更改默认选中"
date: 2016-01-12 13:59:11 +0800
comments: true
categories: summary
---

### 目的
**实现解决方案和解决版本两个选择框默认选中一个值**

### 解决前
![](/images/chandao_bug_summit_unresolve.jpg)

### 解决后
![](/images/chandao_bug_summit_resolve.jpg)

### 编辑文件zentaopms/module/bug/lang/zh-cn.php
	第251 行
	// $lang->bug->resolutionList['']           = ‘'; 注释掉


### 编辑文件zentaopms/module/bug/view/resolve.html.php
	第36行
	原来是
	<td><?php echo html::select('resolvedBuild', $builds, '', "class='form-control chosen'");?></td> 
	改为
	<td><?php echo html::select('resolvedBuild', $builds, $bug->openedBuild, "class='form-control chosen'");?></td> 




