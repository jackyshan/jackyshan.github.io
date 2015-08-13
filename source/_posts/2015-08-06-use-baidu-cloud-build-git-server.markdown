---
layout: post
title: "利用百度云搭建Git服务器同步代码"
date: 2015-08-06 21:34:00 +0800
comments: true
categories: summary
---

* 在百度云目录新建test.git文件夹
  
  ![](/images/test_git_init.png)
  
* 在test.git目录shell下执行git --bare init命令

  ![](/images/test_git_bare.png)

* clone百度云下的test.git

  ![](/images/test_git_clone.png)

* 利用sourcetree上传代码

  ![](/images/test_git_commit.png)
  