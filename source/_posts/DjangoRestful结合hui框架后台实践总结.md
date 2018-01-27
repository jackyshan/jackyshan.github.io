---
title: DjangoRestful结合hui框架后台实践总结
date: 2017-11-02 21:46:21
tags: Django
---

![img](http://upload-images.jianshu.io/upload_images/301129-8d8c9a6495acd684.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Django REST framework
Django REST framework is a powerful and flexible toolkit for building Web APIs.

### [Project setup](http://www.django-rest-framework.org/tutorial/quickstart/#project-setup)
Create a new Django project named `tutorial`, then start a new app called `quickstart`

	# Create the project directory
	mkdir tutorial
	cd tutorial

	# Create a virtualenv to isolate our package dependencies locally
	virtualenv env
	source env/bin/activate  # On Windows use `env\Scripts\activate`

	# Install Django and Django REST framework into the virtualenv
	pip install django
	pip install djangorestframework

	# Set up a new project with a single application
	django-admin.py startproject tutorial .  # Note the trailing '.' character
	cd tutorial
	django-admin.py startapp quickstart
	cd ..

同步到数据库

    python manage.py migrate

创建超级管理员

    python manage.py createsuperuser

### [Serializers](http://www.django-rest-framework.org/tutorial/quickstart/#serializers)
创建一个新的文件`tutorial/quickstart/serializers.py `,用来渲染我们的数据

	from django.contrib.auth.models import User, Group
	from rest_framework import serializers


	class UserSerializer(serializers.HyperlinkedModelSerializer):
	    class Meta:
	        model = User
	        fields = ('url', 'username', 'email', 'groups')


	class GroupSerializer(serializers.HyperlinkedModelSerializer):
	    class Meta:
	        model = Group
	        fields = ('url', 'name')

### [Views](http://www.django-rest-framework.org/tutorial/quickstart/#views)
打开`tutorial/quickstart/views.py `，写入代码

    from django.contrib.auth.models import User, Group
    from rest_framework import viewsets
    from tutorial.quickstart.serializers import UserSerializer, GroupSerializer

    class UserViewSet(viewsets.ModelViewSet):
        """
        API endpoint that allows users to be viewed or edited.
        """
        queryset = User.objects.all().order_by('-date_joined')
        serializer_class = UserSerializer


    class GroupViewSet(viewsets.ModelViewSet):
        """
        API endpoint that allows groups to be viewed or edited.
        """
        queryset = Group.objects.all()
        serializer_class = GroupSerializer

### [URLs](http://www.django-rest-framework.org/tutorial/quickstart/#urls)
打开`tutorial/urls.py `，写入代码

	from django.conf.urls import url, include
	from rest_framework import routers
	from tutorial.quickstart import views

	router = routers.DefaultRouter()
	router.register(r'users', views.UserViewSet)
	router.register(r'groups', views.GroupViewSet)

	# Wire up our API using automatic URL routing.
	# Additionally, we include login URLs for the browsable API.
	urlpatterns = [
	    url(r'^', include(router.urls)),
	    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
	]

### [Settings](http://www.django-rest-framework.org/tutorial/quickstart/#settings)
控制API访问权限必须为admin用户，打开`tutorial/settings.py `，写入代码

    INSTALLED_APPS = [
        ...,
        'rest_framework',
    ]

    REST_FRAMEWORK = {
        'DEFAULT_PERMISSION_CLASSES': [
            'rest_framework.permissions.IsAdminUser',
        ],
        'PAGE_SIZE': 10
    }

### 测试API是否生效
开启web服务通过以下命令

    python manage.py runserver

执行过程

	Performing system checks...

	System check identified no issues (0 silenced).
	August 04, 2017 - 06:03:53
	Django version 1.11.4, using settings 'tutorial.settings'
	Starting development server at http://127.0.0.1:8000/
	Quit the server with CONTROL-C.

浏览器打开[http://127.0.0.1:8000/](http://127.0.0.1:8000/)，登录刚才创建的超级用户

![img](http://upload-images.jianshu.io/upload_images/301129-f9238a021c959481.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在DjangoRestful框架搭建完成，下面可以下载[h-ui](http://www.h-ui.net/Hui-overview.shtml)框架，进行后台部署

![img](http://upload-images.jianshu.io/upload_images/301129-daaa7c3bab136a7a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## H-ui.admin框架
[H-ui.admin](http://www.h-ui.net/H-ui.admin.shtml)是用H-ui前端框架开发的轻量级网站后台模版采用源生html语言，完全免费，简单灵活，兼容性好让您快速搭建中小型网站后台。

![img](http://upload-images.jianshu.io/upload_images/301129-0409e68d70d2f2de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### H-ui.admin部署
下载http://downs.h-ui.net/h-ui/H-ui.admin_v3.1.3.1.zip，解压H-ui.admin，重名为templates，拖动到目录`tutorial/quickstart`，打开`tutorial/quickstart/views.py`，添加`loginUser`方法

    from django.shortcuts import render

    def loginUser(request):
        if request.method == 'GET':
            return render(request, 'login.html')

打开`tutorial/urls.py`，输入login的url路由地址

    urlpatterns = [
        ...,
    	url(r'^login$', views.loginUser),
    ]

打开`tutorial/settings.py`，设置`TEMPLATES`地址

    TEMPLATES = [
        {
            ...,
            'DIRS': [BASE_DIR+"/tutorial/quickstart/templates",],
        },
    ]

执行`python manage.py runserver`，执行过程如下

    System check identified no issues (0 silenced).
    August 04, 2017 - 06:38:30
    Django version 1.11.4, using settings 'tutorial.settings'
    Starting development server at http://127.0.0.1:8000/
    Quit the server with CONTROL-C.
    [04/Aug/2017 06:38:32] "GET /login HTTP/1.1" 200 3887
    [04/Aug/2017 06:38:32] "GET /static/h-ui/css/H-ui.min.css HTTP/1.1" 404 1676
    Not Found: /lib/Hui-iconfont/1.0.8/iconfont.css
    Not Found: /lib/jquery/1.9.1/jquery.min.js
    [04/Aug/2017 06:38:33] "GET /static/h-ui.admin/css/H-ui.login.css HTTP/1.1" 404 1700
    [04/Aug/2017 06:38:33] "GET /lib/Hui-iconfont/1.0.8/iconfont.css HTTP/1.1" 404 4123
    [04/Aug/2017 06:38:33] "GET /lib/jquery/1.9.1/jquery.min.js HTTP/1.1" 404 4108
    [04/Aug/2017 06:38:33] "GET /static/h-ui/js/H-ui.min.js HTTP/1.1" 404 1670
    Not Found: /lib/jquery/1.9.1/jquery.min.js
    [04/Aug/2017 06:38:33] "GET /lib/jquery/1.9.1/jquery.min.js HTTP/1.1" 404 4108
    [04/Aug/2017 06:38:33] "GET /static/h-ui/js/H-ui.min.js HTTP/1.1" 404 1670

打开[http://127.0.0.1:8000/login](http://127.0.0.1:8000/login)，效果图

![img](http://upload-images.jianshu.io/upload_images/301129-c1a76ddbf42a1606.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到静态文件和样式文件404，打开`tutorial/settings.py`，设置静态文件路径

    STATIC_URL = '/static/'
    STATIC_ROOT = BASE_DIR+"/tutorial/quickstart/static"

    STATICFILES_DIRS = [
        # os.path.join(BASE_DIR, "static"),
        os.path.join(BASE_DIR, "tutorial/quickstart/temp"),
    ]

创建目录

    cd quickstart
    mkdir static
    static temp

打开`templates`目录，找到`lib`、`static`、`temp`目录移动到`tutorial/quickstart/temp`目录下

执行命令`python manage.py collectstatic`

打开`tutorial/quickstart/templates/login.html`，在静态链接那里前面加上`static`，例如

	<link href="`static`/static/h-ui/css/H-ui.min.css" rel="stylesheet" type="text/css" />

再次打开[http://127.0.0.1:8000/login](http://127.0.0.1:8000/login)，效果图

![img](http://upload-images.jianshu.io/upload_images/301129-5bfb6b920caa2231.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

页面效果出来了，我们可以编写登录逻辑了。

### 登录页面
打开`tutorial/quickstart/views.py`，设置`loginUser`的`POST`方法

    from django.contrib.auth import authenticate

    def loginUser(request):
        if request.method == 'GET':
            return render(request, 'login.html')

        print request.POST
        username = request.POST['username']
        password = request.POST['password']
        user = authenticate(username=username, password=password)
        if user is not None:
            if user.is_active:
                login(request, user)
                # Redirect to a success page.
                return render(request, 'index.html')
            else:
                return render(request, 'login.html', {'errmsg': 'disabled account'})
                # Return a 'disabled account' error message
        else:
            return render(request, 'login.html', {'errmsg': 'invalid login'})

打开`tutorial/quickstart/templates/login.html`，设置报错提示`Huialert-error`，设置`action`

	    <form class="form form-horizontal" action="/login" method="post">
	    {% csrf_token %}
	  {% if errmsg is not None %}
	      <div class="Huialert Huialert-error"><i class="icon-remove"></i>{{errmsg}}</div>
	  {% endif %} 

用户名密码输入错误报错

![img](http://upload-images.jianshu.io/upload_images/301129-c8233e83f933dcec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输入正确，打开`index.html`页面，编辑`tutorial/quickstart/templates/index.html`，静态链接前面加上'static'，打开之后显示

![img](http://upload-images.jianshu.io/upload_images/301129-7963e1b9a9313fda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 用户列表

打开`tutorial/quickstart/templates/index.html`，修改`welcome`为`userlist`

	<section class="Hui-article-box">
		<div id="Hui-tabNav" class="Hui-tabNav hidden-xs">
			<div class="Hui-tabNav-wp">
				<ul id="min_title_list" class="acrossTab cl">
					<li class="active">
						<span title="我的用户" data-href="/userlist">我的用户</span>
						<em></em></li>
			</ul>
		</div>
			<div class="Hui-tabNav-more btn-group"><a id="js-tabNav-prev" class="btn radius btn-default size-S" href="javascript:;"><i class="Hui-iconfont"></i></a><a id="js-tabNav-next" class="btn radius btn-default size-S" href="javascript:;"><i class="Hui-iconfont"></i></a></div>
	</div>
		<div id="iframe_box" class="Hui-article">
			<div class="show_iframe">
				<div style="display:none" class="loading"></div>
				<iframe scrolling="yes" frameborder="0" src="/userlist"></iframe>
		</div>
	</div>
	</section>

打开`tutorial/quickstart/views.py`，添加方法`userList`

    def userList(request):
        if request.method == 'GET':
            return render(request, 'user-list.html')

打开`tutorial/urls.py`，输入`userlist`的路由

	urlpatterns = [
		...,
	    url(r'^userlist$', views.userList),
	]

打开`tutorial/quickstart/templates/user-list.html`，修改静态文件链接

执行`python manage.py runserver`

![img](http://upload-images.jianshu.io/upload_images/301129-c9ecb3864a9ff577.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 资讯列表
打开`tutorial/quickstart/templates/index.html`，修改`article-list.html`为`/articlelist`

				<ul>
					<li><a data-href="/articlelist" data-title="资讯管理" href="javascript:void(0)">资讯管理</a></li>
			    </ul>

打开`tutorial/quickstart/views.py`，添加方法`articleList`

	def articleList(request):
	    if request.method == 'GET':
	        return render(request, 'article-list.html')

打开`tutorial/urls.py`，输入`articlelist`的路由

	urlpatterns = [
		...,
	    url(r'^articlelist$', views.articleList),
	]

打开`tutorial/quickstart/templates/article-list.html`，修改静态文件链接

实现资讯列表页面效果

![img](http://upload-images.jianshu.io/upload_images/301129-14f65a811e683909.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 数据渲染（结合RestfulApi）
之前的页面`我的用户`和`资讯管理`的数据都是写死的，现在把`资讯管理`拿出来，单独一个例子来进行数据渲染的实验。这里涉及到`models.py`、`serializers.py`、`views.py`、`urls.py`、`settings.py`、`apps.py`

#### Restful设置
建立Restful框架所需要的模型、序列、视图等

##### 模型
打开`tutorial/quickstart/models.py`，建立`Article`模型

	# -*- coding: utf-8 -*-
	from __future__ import unicode_literals

	from django.db import models

	# Create your models here.
	class Article(models.Model):
	    created = models.DateTimeField(auto_now_add=True)
	    aid = models.AutoField(primary_key=True) #id
	    title = models.CharField(max_length=100, blank=True, default='资讯标题') #标题
	    category = models.TextField(default='行业动态') #分类
	    source = models.TextField(default='H-ui') #来源
	    update_time = models.TextField(default='2017-8-6') #更新时间
	    see_times = models.TextField(default='1111') #浏览次数
	    publish_status = models.IntegerField(default=0) #发布状态 0未发布 1已发布 2草稿

	    class Meta:
	        ordering = ('created',)
##### 序列
打开`tutorial/quickstart/serializers.py`，建立`ArticleSerializer`序列

	from rest_framework import serializers
	from tutorial.quickstart.models import Article

	class ArticleSerializer(serializers.HyperlinkedModelSerializer):
	    class Meta:
	        model = Article
	        fields = ('url', 'aid', 'title', 'category', 'source', 'update_time', 'see_times', 'publish_status')

##### 视图
打开`tutorial/quickstart/views.py`，建立`ArticleViewSet`视图

    from tutorial.quickstart.serializers import ArticleSerializer
    from tutorial.quickstart.models import Article

    class ArticleViewSet(viewsets.ModelViewSet):
        """
        API endpoint that allows groups to be viewed or edited.
        """
        queryset = Article.objects.all()
        serializer_class = ArticleSerializer

##### 路由
打开`tutorial/urls.py`，添加`articles`Api的路由

    ...
    router.register(r'articles', views.ArticleViewSet)

##### 设置
打开`tutorial/settings.py`，添加`INSTALLED_APPS`

    INSTALLED_APPS = [
        ...,
        'tutorial.quickstart.apps.QuickstartConfig',
    ]

##### app内部配置
打开`tutorial/quickstart/apps.py`，调整`quickstart`的`name`

	# -*- coding: utf-8 -*-
	from __future__ import unicode_literals

	from django.apps import AppConfig


	class QuickstartConfig(AppConfig):
	    name = 'tutorial.quickstart'

##### 同步数据库

    python manage.py makemigrations
    python manage.py migrate

##### 完成
打开[http://127.0.0.1:8000/articles/](http://127.0.0.1:8000/articles/)

![img](http://upload-images.jianshu.io/upload_images/301129-cee05e1677af7a4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在还没有数据，添加(POST)一条数据上去

![img](http://upload-images.jianshu.io/upload_images/301129-990dfa31dcde78a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### H-ui设置
对H-ui的静态页面进行调整，渲染Api的数据

##### GET
打开`tutorial/quickstart/views.py`，调整方法`articleList`，加入参数数据`articles`

    def articleList(request):
        if request.method == 'GET':
            articles = Article.objects.all()
            return render(request, 'article-list.html', {'articles': articles})

打开`tutorial/quickstart/templates/article-list.html`，调整`tbody`，接收数据，进行渲染

			<tbody>
			{% for article in articles %}
				<tr class="text-c">
					<td><input type="checkbox" value="" name=""></td>
					<td>{{article.aid}}</td>
					<td class="text-l"><u style="cursor:pointer" class="text-primary" onClick="article_edit('查看','article-zhang.html','10001')" title="查看">{{article.title}}</u></td>
					<td>{{article.category}}</td>
					<td>{{article.source}}</td>
					<td>{{article.update_time}}</td>
					<td>{{article.see_times}}</td>
					<td class="td-status"><span class="label label-success radius">{% if article.publish_status == 1 %} {{'已发布' }} {% else %} {{'未发布' }} {% endif %}</span></td>
					<td class="f-14 td-manage"><a style="text-decoration:none" onClick="article_stop(this,'10001')" href="javascript:;" title="下架"><i class="Hui-iconfont"></i></a> <a style="text-decoration:none" class="ml-5" onClick="article_edit('资讯编辑','article-add.html','10001')" href="javascript:;" title="编辑"><i class="Hui-iconfont"></i></a> <a style="text-decoration:none" class="ml-5" onClick="article_del(this,'10001')" href="javascript:;" title="删除"><i class="Hui-iconfont"></i></a></td>
				</tr>
			{% endfor %}
			</tbody>

视图效果

![img](http://upload-images.jianshu.io/upload_images/301129-d0d98b205057e438.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### DELETE
打开`tutorial/quickstart/templates/article-list.html`，修改`article_del`方法

    onClick="article_del(this,'{{article.aid}}')"

jQuery实现`article_del`方法，请求Api

	/*资讯-删除*/
	function article_del(obj,id){
		layer.confirm('确认要删除吗？',function(index){
			var csrftoken = $.cookie('csrftoken');

			function csrfSafeMethod(method) {
			    // these HTTP methods do not require CSRF protection
			    return (/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
			}

			$.ajaxSetup({
			    beforeSend: function(xhr, settings) {
			        if (!csrfSafeMethod(settings.type) && !this.crossDomain) {
			            xhr.setRequestHeader("X-CSRFToken", csrftoken);
			        }
			    }
			});

			$.ajax({
		    url: "/articles/"+id,
		    type: 'DELETE',
		    success: function(result) {
		        // Do something with the result
			        $(obj).parents("tr").remove();
			        layer.msg('已删除!',{icon:1,time:1000});
			    }
			});		
		});
	}

服务器日志显示处理成功

    [07/Aug/2017 06:46:00] "DELETE /articles/0 HTTP/1.1" 301 0
    [07/Aug/2017 06:46:00] "DELETE /articles/0/ HTTP/1.1" 204 0

##### POST
打开`tutorial/quickstart/templates/article-list.html`，修改`添加资讯`的`data-href`

    <a class="btn btn-primary radius" data-title="添加资讯" data-href="/articleadd" onclick="Hui_admin_tab(this)" href="javascript:;">

打开`tutorial/urls.py`，增加`articleadd`路由

	urlpatterns = [
	    ...,
	    url(r'^articleadd$', views.articleAdd),
	]

打开`tutorial/quickstart/views.py`，添加`articleAdd`方法

    def articleAdd(request):
        if request.method == 'GET':
            return render(request, 'article-add.html')

刷下网页，打开`添加资讯`按钮

![img](http://upload-images.jianshu.io/upload_images/301129-d178e13ca9772385.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

页面效果已经出来了，可以去掉不用的页面元素，比如`简略标题`、`文章类型`等

![img](http://upload-images.jianshu.io/upload_images/301129-eb3d0453a04afea6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

表单验证

	//表单验证
	$("#form-article-add").validate({
		rules:{
			articletitle:{
				required:true,
			},
			articlecolumn:{
				required:true,
			},
			sources:{
				required:true,
			},
			publishdate:{
				required:true,
			},

		},
		onkeyup:false,
		focusCleanup:true,
		success:"valid",
		submitHandler:function(form){
			addArticle();
		}
	});

添加`addArticle`方法

	function addArticle() {
		var csrftoken = $.cookie('csrftoken');

		function csrfSafeMethod(method) {
		    // these HTTP methods do not require CSRF protection
		    return (/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
		}

		$.ajaxSetup({
		    beforeSend: function(xhr, settings) {
		        if (!csrfSafeMethod(settings.type) && !this.crossDomain) {
		            xhr.setRequestHeader("X-CSRFToken", csrftoken);
		        }
		    }
		});
	 	$.post("/articles/",
	    {
		    "title": $("#articletitle").val(),
		    "category": $("#articlecolumn").val(),
		    "source": $("#sources").val(),
		    "update_time": $("#publishdate").val(),
		},
		function(data,status) {
	        alert("数据: \n" + data + "\n状态: " + status);
	    });
	}

输入数据，日志显示数据新增成功

    [07/Aug/2017 07:26:19] "POST /articles/ HTTP/1.1" 201 185

### 总结
Restful实现了增删查改的接口，H-ui作为一套后台UI框架实现漂亮的模板的效果，结合起来开发效率方便很多。

统计当前python使用sdk环境列表

    pip freeze >requirements.txt

requirements.txt

	Django==1.11.4
	djangorestframework==3.6.3
	pytz==2017.2

代码参考
https://github.com/jackyshan/djangohuiapisimple