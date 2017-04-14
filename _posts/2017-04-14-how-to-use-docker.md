---
layout: post
title:  "docker安装部署javaweb"
categories: docker
tags:  docker
author: hanjianchun
---

* content
{:toc}

docker容器安装部署javaweb项目



## 安装启动查看步骤
	
> 安装docker

	yum pull docker

> 启动、关闭和重启docker服务

	service docker start
	service docker stop
	service docker restart

> 设置为开机自启动

	systemctl enable docker

> 下载centos镜像

	docker pull centos

> 查看所有的镜像

	docker images

	REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
	docker.io/centos    latest              a8493f5f50ff        7 days ago          192.5 MB

> 新建并启动centos容器
	
	docker run -i -t -v /mnt/software/:/mnt/software/ centos:latest /bin/bash  
	1. 其中，相关参数包括：  
	2. -i：表示以"交互模式"运行容器  
	3. -t：表示容器启动后会进入其命令行  
	4. -v：表示需要将本地哪个目录挂载到容器中，格式：-v <宿主机目录>:<容器目录>  

> 查看docker容器的进程

	docker ps	#查看正在运行的容器
	docker ps -a	#查看所有的容器
	docker ps -l	#查看最近一次的容器

	CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                            PORTS                    NAMES
	63ef725483b8        tomcat_api_v1:01    "/bin/sh -c '/root/ap"   58 minutes ago      Up 58 minutes                     0.0.0.0:8888->8080/tcp   tomcat_api_01

> 已存在容器的执行命令
	
	docker stop {CONTAINER ID}		#关闭容器
	docker stop $(docker ps -q)		#停止全部运行的容器
	docker start {CONTAINER ID}		#开启容器
	docker attach {CONTAINER ID}		#1.进入开启的容器
	docker exec -it {CONTAINER ID} /bin/bash	#2.进入开启的容器
	在进入容器之后就可以安装jdk和tomcat并配置环境变量
> 删除容器和镜像

	docker rm {CONTAINER ID}	#删除关闭的容器
	docker rm $(docker ps -aq)	#删除全部的容器
	docker stop $(docker ps -q) & docker rm $(docker ps -aq)	#关闭并删除全部的容器
	docker rmi	${IMAGE ID}		#删除没有容器依赖的镜像

## 使用[docker commit]制作镜像

> commit制作镜像会把一些没用的全部打包过来，比如我安装了nginx,redis，tomcat;现在我只需要tomcat容器，然后commit制作了一个镜像，会把nginx和redis全部打包进来，docker history {IMAGE ID} 使用这个命令可以看到其实多了很大的空间，但是如果安装基础服务，比如JDK，那是可以用这个办法的：

	先启动centos容器，进入安装好jdk并配置好环境变量，退出；
	docker commit {CONTAINER ID}  {your_name}	#这就制作好了一个镜像，这个镜像就是当前{CONTAINER ID}的状态

## 使用dockerfile制作javaweb镜像
	
> 完美解决了docker commit的缺点，用什么就装什么，一点不浪费；

	首先在/home/ok/tomcat目录下新建文件，名为：Dockerfile
```docker
# Pull base image
FROM centos:latest

MAINTAINER ultrasoft "ultrasoft"

ADD jdk-8u111-linux-x64.tar.gz /usr/java

ENV JAVA_HOME /usr/java/jdk1.8.0_111
ENV PATH $JAVA_HOME/bin:$PATH

#TOMCAT_ADD
ADD apache-tomcat-7.0.77.tar.gz /root

ADD catalina.sh /root/apache-tomcat-7.0.77/bin
ADD server.xml /root/apache-tomcat-7.0.77/conf
RUN rm -rf /root/apache-tomcat-7.0.77/webapps/ROOT/

ADD ROOT.tar.gz /root/apache-tomcat-7.0.77/webapps/

EXPOSE 8080

ENTRYPOINT /root/apache-tomcat-7.0.77/bin/startup.sh && tail -f /root/apache-tomcat-7.0.77/logs/catalina.out

```
