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
![](/images/posts/docker/docker-rabbitmq-search.bmp)

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