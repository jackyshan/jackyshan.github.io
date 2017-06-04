---
layout: post
title: "ubuntu上ipsec搭建全程psk记录"
date: 2016-01-12 14:07:19 +0800
comments: true
categories: summary
---

### 目的
**实现在ubuntu服务器上搭建ipsec vpn**

### 安装
	apt-get update
	
	apt-get install libpam0g-dev libssl-dev make gcc
	
	wget http://download.strongswan.org/strongswan.tar.gz tar xzf strongswan.tar.gz cd strongswan-*
	
	./configure --enable-eap-identity --enable-eap-md5 \ --enable-eap-mschapv2 --enable-eap-tls --enable-eap-ttls --enable-eap-peap \ --enable-eap-tnc --enable-eap-dynamic --enable-eap-radius --enable-xauth-eap \ --enable-xauth-pam --enable-dhcp --enable-openssl --enable-addrblock --enable-unity \ --enable-certexpire --enable-radattr --enable-tools --enable-openssl --disable-gmp 
	
	make; make install

完成后使用命令ipsec version检查是否出现版本号等信息
若出现ipsec: command not found 则代表没有成功编译安装

### 配置
#### 编辑/usr/local/etc/ipsec.conf
	vi /usr/local/etc/ipsec.conf
	
	conn android_xauth_psk keyexchange=ikev1 left=%defaultroute leftauth=psk leftsubnet=0.0.0.0/0 right=%any rightauth=psk rightauth2=xauth rightsourceip=10.31.2.0/24 auto=add
	
#### 编辑/usr/local/etc/strongswan.conf
	vi /usr/local/etc/strongswan.conf	
	# strongswan.conf - strongSwan configuration file
	 #
	 # Refer to the strongswan.conf(5) manpage for details
	 #
	 # Configuration changes should be made in the included files
	 
	 charon {
	         load_modular = yes
	         duplicheck.enable = no
	         compress = yes
	         plugins {
	                 include strongswan.d/charon/*.conf
	         }
	 
	         dns1 = 8.8.8.8
	         dns2 = 8.8.4.4
	         nbns1 = 8.8.8.8
	         nbns2 = 8.8.4.4
	 
	         filelog {
	                /var/log/strongswan.charon.log {
	                    time_format = %b %e %T
	                    default = 2
	                    append = no
	                    flush_line = yes
	                }
	        }
	 }
	 
	 include strongswan.d/*.conf

#### 编辑/usr/local/etc/ipsec.secrets
	vi /usr/local/etc/ipsec.secrets
	
	: PSK "mykey"
	test %any : EAP "test123456"

#### 编辑/etc/sysctl.conf
将net.ipv4.ip_forward=1
一行前面的#号去掉，保存后执行sysctl -p。 

	sysctl -p
	
#### 配置iptables规则
	iptables -t nat -A POSTROUTING -j MASQUERADE iptables -I FORWARD -p tcp --syn -i ppp+ -j TCPMSS --set-mss 1356 
	
#### 保存iptables配置并配置开机自动载入
	iptables-save > /etc/iptables.rules cat > /etc/network/if-up.d/
	
	iptables<<EOF #!/bin/sh iptables-restore < /etc/iptables.rules EOF
	
	chmod +x /etc/network/ifup.d/iptables

#### 配置开机自动开启ipsec
	vi /etc/init/ipsec.conf
	
	#ipsec auto start
	
	start on runlevel [2333]
	stop on runlevel [!2333]
	
	respawn
	
	 
	exec ipsec start

