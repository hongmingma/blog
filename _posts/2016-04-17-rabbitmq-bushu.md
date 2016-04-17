---
layout: post
title:  rabbitMQ 集群环境搭建
category: 消息队列
---


示例为在 192.168.6.200，192.168.6.201 搭建 rabbitmq高可用集群步骤：
安装用户需要具备root 权限

####1、安装erlang
	https://www.erlang-solutions.com/resources/download.html
 
	sudo wget http://packages.erlang-solutions.com/erlang-solutions-1.0-1.noarch.rpm
	sudo rpm -Uvh erlang-solutions-1.0-1.noarch.rpm
	sudo rpm --import http://packages.erlang-solutions.com/rpm/erlang_solutions.asc
	sudo yum install erlang
	sudo yum install esl-erlang

	备注：
	安装过程中可能因为一些包的冲突导致安装失败，卸载冲突的包，重新安装即可
	sodu rpm -e esl-erlang-18.3-1.x86_64

####2、安装rabbitmq server

	sudo wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.1/rabbitmq-server-3.6.1-1.noarch.rpm
	sudo rpm --import https://www.rabbitmq.com/rabbitmq-signing-key-public.asc
	sudo yum install rabbitmq-server-3.6.1-1.noarch.rpm
	备注：必须在erlang安装成功后rabbit安装才能成功

####3、hostname 配置
	因为rabbit集群节点通过hostname通信，所以需要配置hostname和IP对应关系
	vi  /etc/hosts
	192.168.6.201  TEST-6-201
	192.168.6.200  TEST-6-200
	备注：修改hostname之后需要重启rabbitmq(如果rabbitmq已经启动过 sudo rabbitmq-server restart)

####4、节点间.erlang.cookie信息同步
	erlang节点集群的信任关系是通过.cookie认证的,所以要保证所有节点.cookie中的内容要相同
	文件路径 var/lib/rabbitmq/.erlang.cookie

####5、集群管理 http://www.rabbitmq.com/clustering.html

#####以集群方式启动每个节点
	[jx_java_web@TEST-6-200 rabbitmq]$ sudo rabbitmq-server -detached
	[jx_java_web@TEST-6-201 rabbitmq]$ sudo rabbitmq-server -detached
	
#####集群状态示例，查看是否启动正常
	[jx_java_web@TEST-6-200 rabbitmq]$ sudo rabbitmqctl cluster_status
	Cluster status of node 'rabbit@TEST-6-200' ...
	[{nodes,[{disc,['rabbit@TEST-6-200']}]},{running_nodes,['rabbit@TEST-6-200']},{cluster_name,<<"rabbit@TEST-6-200">>},{partitions,[]},
 	{alarms,[{'rabbit@TEST-6-200',[]}]}]

#####创建集群
	[jx_java_web@TEST-6-200 rabbitmq]$ sudo rabbitmqctl stop_app 
	[jx_java_web@TEST-6-200 rabbitmq]$  sudo rabbitmqctl join_cluster rabbit@TEST-6-201 【默认节点类型是disc,数据是存储在磁盘的】
	[jx_java_web@TEST-6-200 rabbitmq]$ sudo rabbitmqctl start_app
	[jx_java_web@TEST-6-200 rabbitmq]$ sudo rabbitmqctl cluster_status
	
#####重启集群：（需要在每个节点关闭和重启）
	sudo rabbitmqctl stop
	sudo rabbitmq-server -detached
	
#####从集群中移除某个节点，需要在某节点执行reset
	rabbitmqctl stop_app
	rabbitmqctl reset
rabbitmqctl start_app

####6、配置集群管理后台

#####启用管理后台插件
	sudo rabbitmq-plugins enable rabbitmq_management
	
#####增加远程登录管理员用户
	sudo rabbitmqctl  add_user  admin 123456
	sudo rabbitmqctl  set_user_tags  admin  administrator
	管理页面：http://192.168.6.200:15672 

#####mq命令行管理软件使用说明：
	http://www.rabbitmq.com/man/rabbitmqctl.1.man.html










 