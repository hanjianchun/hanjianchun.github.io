---
layout: post
title:  "掌上办公-走访调查接口文档"
categories: api
tags:  api
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

	首先在/home/ok/tomcat目录下新建文件，名为：Dockerfile,然后拷贝jdk，tomcat到当前目录，项目打包成ROOT.tar.gz;

```java
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

	指令详解：
	FROM	指明从哪个镜像制作
	MAINTAINER	名称	  邮箱
	ADD 	如果 <源路径> 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 <目标路径> 去。
	COPY	将文件从<源路径>复制到<目标路径>
	RUN		在制作镜像的过程中执行linux指令
	EXPOSE	指定对外提供的端口
	ENV		设置环境变量
	ENTRYPOINT		指定镜像启动的入口点

> Dockerfile编写完成后进行镜像生成

	docker build -t javaweb:0.1 .		#这句执行后会生成一个名为javawebTAG为0.1的镜像

> 启动容器

	docker run -d -p 8080:8080 --name tomcat_01 javaweb:0.1
	启动完成后会自动启动tomcat并部署项目
	如果想再启动一个容器，把端口和名称换一下就行
	docker run -d -p 8888:8080 --name tomcat_02 javaweb:0.1

> 现在就可以通过 http://ip:8080访问web网站了

## 遇到的问题
	
> docker部署了tomcat后调用命令 bin/startup.sh发现启动特别慢

	修改tomcat/bin目录下的文件catalina.sh
	加入下面一段代码。解决启动时耗时太久的问题
	if [[ "$JAVA_OPTS" != *-Djava.security.egd=* ]]; then 
		JAVA_OPTS="$JAVA_OPTS -Djava.security.egd=file:/dev/./urandom" 
	fi

	在dockerfile的时候把这个文件替换掉也可以的

