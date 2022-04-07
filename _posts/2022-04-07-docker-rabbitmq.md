---
layout: post
title: Docker安装RabbitMQ
categories: [Docker]
description: Docker安装RabbitMQ
keywords: Docker, RabbitMQ
---

Docker安装RabbitMQ

### 查看仓库里的RabbitMQ
```java
docker search rabbitmq
```
![](/images/posts/docker/docker-rabbitmq-search.jpg)

### 安装RabbitMQ
```java
docker pull rabbitmq
```

### 启动RabbitMQ
```java
docker run -d --hostname my-rabbit --name rabbit -p 15672:15672 -p 5672:5672 rabbitmq
```

### 安装可视化插件
```java
docker ps 
docker exec -it 镜像ID /bin/bash
rabbitmq-plugins enable rabbitmq_management
```

### 访问可视化界面，查看启动状态
```
// 默认的用户密码都是guest
http://ip:15672
```

### 创建用户分配权限
```
// 创建用户
cd /sbin
rabbitmqctl add_user 用户名 密码

// 分配访问权限
rabbitmqctl  set_permissions -p / 用户名 '.*' '.*' '.*'
```
不分配权限，连接时可能会抛出异常
```java
com.rabbitmq.client.ShutdownSignalException: connection error; protocol method: #method<connection.close>(reply-code=530, reply-text=NOT_ALLOWED - access to vhost '/' refused for user 'root', class-id=10, method-id=40)
	at com.rabbitmq.utility.ValueOrException.getValue(ValueOrException.java:66) ~[amqp-client-5.13.1.jar:5.13.1]
```