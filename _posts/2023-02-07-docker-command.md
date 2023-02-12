---
layout: post
title: Docker常用命令
categories: [Docker]
description: Docker常用命令
keywords: Docker
---

Docker常用命令

### 创建并运行容器
-a stdin: 指定标准输入输出内容类型，可选 STDIN/STDOUT/STDERR 三项；<br />
-d: 后台运行容器，并返回容器ID；<br />
-i: 以交互模式运行容器，通常与 -t 同时使用；<br />
-P: 随机端口映射，容器内部端口随机映射到主机的端口<br />
-p: 指定端口映射，格式为：主机(宿主)端口:容器端口<br />
-t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；<br />
--name="nginx-lb": 为容器指定一个名称；<br />
--dns 8.8.8.8: 指定容器使用的DNS服务器，默认和宿主一致；<br />
--dns-search example.com: 指定容器DNS搜索域名，默认和宿主一致；<br />
-h "mars": 指定容器的hostname；<br />
-e username="ritchie": 设置环境变量；<br />
--env-file=[]: 从指定文件读入环境变量；<br />
--cpuset="0-2" or --cpuset="0,1,2": 绑定容器到指定CPU运行；<br />
-m :设置容器使用内存最大值；<br />
--net="bridge": 指定容器的网络连接类型，支持 bridge/host/none/container: 四种类型；<br />
--link=[]: 添加链接到另一个容器；<br />
--expose=[]: 开放一个端口或一组端口；<br />
--volume , -v: 绑定一个卷<br />
```shell
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

### 文件拷贝
```shell
# 将主机/path目录拷贝到容器/path目录下
docker cp /path 容器id:/path/

# 将主机/path目录拷贝到容器中，并重命名为path
docker cp /path 容器id:/path

# 将容器的/path目录拷贝到主机的/path目录中
docker cp  容器id:/path /path/
```

### 容器制作镜像
-a :提交的镜像作者；

-c :使用Dockerfile指令来创建镜像；

-m :提交时的说明文字；

-p :在commit时，将容器暂停。
```shell
docker commit -a "runoob.com" -m "my apache" a404c6c174a2  mymysql:v1
```

### 镜像导出
```shell
# 根据镜像ID将镜像导出成一个文件
docker save 镜像id > 文件名.tar
docker export 镜像id > 文件名.tar

# 将多个镜像打包成一个文件
docker save -o images.tar postgres:9.6 mongo:3.4
```

### 镜像导入
```shell
docker load < 文件名.tar
docker import - 镜像名 < 文件名.tar
```
## save和export两种方案的差别
特别注意：两种方法不可混用。<br />
如果使用 import 导入 save 产生的文件，虽然导入不提示错误，但是启动容器时会提示失败，会出现类似"docker: Error response from daemon: Container command not found or does not exist"的错误。

- 文件大小不同```export 导出的镜像文件体积小于 save 保存的镜像```
- 是否可以对镜像重命名 ``import支持，load不支持``
- 是否可以同时将多个镜像打包到一个文件中```export不支持，save支持```
- 是否包含镜像历史，export 导出（import 导入）是根据容器拿到的镜像，再导入时会丢失镜像所有的历史记录和元数据信息（即仅保存容器当时的快照状态），所以无法进行回滚操作。
而 save 保存（load 加载）的镜像，没有丢失镜像的历史，可以回滚到之前的层（layer）。
- 应用场景不同，docker export 的应用场景：主要用来制作基础镜像，比如我们从一个 ubuntu 镜像启动一个容器，然后安装一些软件和进行一些设置后，使用 docker export 保存为一个基础镜像。然后把这个镜像分发给其他人使用，比如作为基础的开发环境；docker save 的应用场景：如果我们的应用是使用 docker-compose.yml 编排的多个镜像组合，但我们要部署的客户服务器并不能连外网。这时就可以使用 docker save 将用到的镜像打个包，然后拷贝到客户服务器上使用 docker load 载入

### Dockerfile
Docker文件示例
```text
#base mirror
FROM java:8
#create by
MAINTAINER  xx
#volume
VOLUME /xx
#拷贝到容器
ADD xx.jar app.jar
EXPOSE 9310
ENTRYPOINT ["java","-Dlog4j2.formatMsgNoLookups=true","-jar","app.jar","--server.port=${PORT}","--spring.profiles.active=${active}","spring.cloud.nacos.discovery.ip=${HOSTIP}","&"]
RUN echo "Asia/Shanghai" > /etc/timezone
```
运行Dockerfile命令
```shell
docker build -t 镜像名:版本 .
```