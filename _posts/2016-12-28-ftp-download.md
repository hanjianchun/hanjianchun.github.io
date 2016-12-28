---
layout: post
title:  "linux如何实现ftp下载"
categories: ftp
tags:  ftp download
author: HjC
---

* content
{:toc}

Linux环境下实现ftp远程下载文件




*要实现下载，顾名思义，需要一个服务端提供连接下载服务；这里我们使用[vsftpd](http://vsftpd.beasts.org/ "VSFTPD")*

----------
两台linux,一台当服务端（192.168.201.130），一台当客户端（192.168.201.131）

# 服务端配置(192.168.201.130) #


> 首先检查服务端有没有安装vsfpd服务

    > rpm -qa | grep vsftpd 
    > vsftpd-2.2.2-21.el6.x86_64

> 如果没有安装vsftpd可以使用yum安装命令进行安装
	
	> yum install vsftpd
	
> 启动或者关闭服务的命令是

	> service vsftpd start
	> service vsftpd stop
	
# 客户端配置(192.168.201.131) #

>安装ftp
	
	> yum install ftp 

到这里环境就搭建完成了

# 编写shell脚本 #

>需求：在客户端每天早上9点将服务端的数据库备份文件下载下来，下载完成之后删除服务端3天前的备份，删除本地7天前的备份

*新建一个shell脚本命名为download.sh*

```shell
#/bin/sh
del_remote_date=`date -d -3day +%Y%m%d`	#获取3天前的时间
del_local_date=`date -d -7day +%Y%m%d`	#获取7天前的时间
get_remote_date=`date '+%Y%m%d'`	#获取当前的时间
remote_path="dbbak/"			#服务器数据库备份目录
local_path="/home/dbbak/"	#本地下载目录
ftpIp="192.168.201.130"		#远程服务器的ip
login="ftpuser test123"		#远程服务器的用户名和密码
 
ftp -i -v -n <<- EOF	#进入ftp模式并设置结束标志EOF
open $ftpIp		#打开连接
user $login							#用户登录
cd $remote_path						#切换远程的用户目录
lcd $local_path			#切换本地的下载目录
get *$get_remote_date.zip		#get命令下载今天的备份文件
mdelete *$del_remote_date.zip		#删除3天前的文件
bye
EOF				#ftp结束

cd $local_path		#切换到本地下载目录
rm -f *$del_local_date.zip		#删除7天前的本地备份文件

```

>修改sh脚本的可执行权限[r=4,w=2,x=1]

	> chmod 744 download.sh
	
>现在新建一个linux调度crontab
	
	> crontab -l       #查看本地的调度
	> crontab -e       #进入编辑页面，可以新增或者删除调度
	
	
	#新建调度每天早上9点执行，并将日志写入download.log中
	0 9 * * * /bin/sh  /home/dbbak/download.sh  >>/home/dbbak/download.log
	
	

# 常见问题或错误 #

>1.客户端在登录时报错：500 OOPS: cannot change directory	

	> setsebool -P ftpd_disable_trans on
	
>2.在客户端中使用root登录ftp服务时可能会被拒绝，在服务端内新建一个用户专用于ftp下载，然后可以给这个用户赋予权限保证系统安全！