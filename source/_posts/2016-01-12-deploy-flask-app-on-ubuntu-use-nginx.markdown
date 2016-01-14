---
layout: post
title: "在 Ubuntu 上使用 Nginx 部署 Flask 应用"
date: 2016-01-12 14:21:29 +0800
comments: true
categories: summary
---

[**参考**](http://www.oschina.net/translate/serving-flask-with-nginx-on-ubuntu)

###安装
	sudo apt-get update && sudo apt-get upgrade
	sudo apt-get install python-setuptools
	sudo easy_install pip
	sudo pip install virtualenv
	sudo apt-get install build-essential python python-dev
	sudo add-apt-repository ppa:nginx/stable
	
	sudo apt-get install nginx
	sudo /etc/init.d/nginx start
	
	sudo pip install uwsgi
	
###目录
	sudo mkdir /var/www
	sudo mkdir /var/www/demoapp
	cd /var/www/demoapp
	virtualenv venv
	. venv/bin/activate
	pip install flask
	
###文件index.py
	from flask import Flask
	app = Flask(__name__)
	 
	@app.route("/")
	def hello():
	    return "Hello World!"
	 
	//这两行要注释掉，不然搭配nginx运行，python会和nginx打架争抢入口，造成Internal Server Error
	//if __name__ == "__main__":
	//    app.run(host='0.0.0.0', port=8080)

###配置nginx
	sudo rm /etc/nginx/sites-enabled/default
	
	vi /var/www/demoapp/demoapp_nginx.conf
	
	server {
        listen      5000;
        server_name localhost;
        charset     utf-8;
        client_max_body_size 75M;

        # 开启gzip
        gzip on;
        # 启用gzip压缩的最小文件，小于设置值的文件将不会压缩
        gzip_min_length 1k;
        # gzip 压缩级别，1-10，数字越大压缩的越好，也越占用CPU时间，后面会有详细说明
        gzip_comp_level 2;
        # 进行压缩的文件类型。javascript有多种形式。其中的值可以在 mime.types 文件中找到。
        gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
        # 是否在http header中添加Vary: Accept-Encoding，建议开启
        gzip_vary on;

        location / { try_files $uri @demoapp; }
        location @demoapp {
                include uwsgi_params;
                uwsgi_pass unix:/var/www/demoapp/demoapp_uwsgi.sock;
        }
	}

	sudo ln -s /var/www/demoapp/demoapp_nginx.conf /etc/nginx/conf.d/
	
###配置uWSGI
	sudo mkdir -p /var/log/uwsgi
	
	vi /var/www/demoapp/demoapp_uwsgi.ini
	
	[uwsgi]
	#application's base folder
	base = /var/www/demoapp
	
	#python module to import
	app = index
	module = %(app)
	
	home = %(base)/venv
	pythonpath = %(base)
	
	#socket file's location
	socket = /var/www/demoapp/%n.sock
	
	#permissions for the socket file
	chmod-socket    = 777
	
	#the variable that holds a flask application inside the module imported at line #6
	callable = app
	
	#location of log files
	logto = /var/log/uwsgi/%n.log
	
###uWSGI Emperor
	vi /etc/init/uwsgi.conf
	
	description "uWSGI"
	start on runlevel [2346]
	stop on runlevel [!2346]
	respawn
	
	env UWSGI=/usr/local/bin/uwsgi
	env LOGTO=/var/log/uwsgi/emperor.log
	
	exec $UWSGI --master --emperor /etc/uwsgi/vassals --die-on-term --uid www-data --gid www-data --logto $LOGTO

.  

	sudo mkdir /etc/uwsgi && sudo mkdir /etc/uwsgi/vassals
	sudo ln -s /var/www/demoapp/demoapp_uwsgi.ini /etc/uwsgi/vassals

###目录权限
	sudo chown -R www-data:www-data /var/www/demoapp/
	sudo chown -R www-data:www-data /var/log/uwsgi/

###启动
	start uwsgi
	/etc/init.d/nginx restart


###托管多个应用
####demoapp1
目录

	mkdir /var/www/demoapp1
	
index.py

	vi index.py 
	
	from flask import Flask
	app = Flask(__name__)
	 
	@app.route("/")
	def hello():
	    return "Hello World!"
	    
demoapp1_nginx.conf 
	
	vi demoapp1_nginx.conf
	
	server {
        listen      5001;//第二个应用端口
        server_name localhost;
        charset     utf-8;
        client_max_body_size 75M;

        # 开启gzip
        gzip on;
        # 启用gzip压缩的最小文件，小于设置值的文件将不会压缩
        gzip_min_length 1k;
        # gzip 压缩级别，1-10，数字越大压缩的越好，也越占用CPU时间，后面会有详细说明
        gzip_comp_level 2;
        # 进行压缩的文件类型。javascript有多种形式。其中的值可以在 mime.types 文件中找到。
        gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
        # 是否在http header中添加Vary: Accept-Encoding，建议开启
        gzip_vary on;

        location / { try_files $uri @demoapp1; }
        location @demoapp1 {
                include uwsgi_params;
                uwsgi_pass unix:/var/www/demoapp1/demoapp1_uwsgi.sock;
        }
	}


demoapp1_uwsgi.ini
	
	vi demoapp1_uwsgi.ini
	
	[uwsgi]
	#application's base folder
	base = /var/www/demoapp1
	
	#python module to import
	app = index
	module = %(app)
	
	home = %(base)/venv
	pythonpath = %(base)
	
	#socket file's location
	socket = /var/www/demoapp1/%n.sock
	
	#permissions for the socket file
	chmod-socket    = 777
	
	#the variable that holds a flask application inside the module imported at line #6
	callable = app
	
	#location of log files
	logto = /var/log/uwsgi/%n.log

外链

	ln -s /var/www/demoapp1/demoapp1_nginx.conf 	/etc/nginx/conf.d/
	ln -s /var/www/demoapp1/demoapp1_uwsgi.ini /etc/uwsgi/vassals/
	
virtualenv

	virtualenv venv
	. venv/bin/activate
	pip install flask
	
目录权限

	chown -R www-data:www-data /var/www/demoapp1/	
启动
	
	restart uwsgi
	/etc/init.d/nginx restart