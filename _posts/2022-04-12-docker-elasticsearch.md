---
layout: post
title: Docker安装ElasticSearch
categories: [Docker]
description: Docker安装ElasticSearch
keywords: Docker, ElasticSearch
---

Docker安装ElasticSearch

### 查看仓库里的ElasticSearch
```java
docker search ElasticSearch
```

### 拉取ElasticSearch镜像
```java
docker pull ElasticSearch
```

### 启动ElasticSearch
```java
docker run -d --name es -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" 镜像id
```
#### 参数说明
-d：后台启动
--name：容器名称
-p：端口映射
-e：设置环境变量
discovery.type=single-node：单机运行
如果启动不了，可以加大内存设置：-e ES_JAVA_OPTS="-Xms512m -Xmx512m"
