---
layout: post
title: SpringCloud集成skywalking
categories: [SpringCloud]
description: SpringCloud集成skywalking
keywords: SpringCloud, skywalking
---

SpringCloud集成skywalking

### skyalking文档地址
https://skywalking.apache.org/docs/main/v8.8.1/en/setup/backend/backend-setup/

### 使用Docker安装基础容器
```markdown
// 安装ElasticSearch7（切换skywalking默认存储方式为es）
docker run --name elasticsearch -p 9200:9200  -p 9300:9300 -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms84m -Xmx512m" -d elasticsearch:7.9.0

// 安装skywalking oap服务
docker run \
--name skywalking-oap \
--restart always \
-p 11800:11800 \
-p 12800:12800 -d \
--privileged=true \
-e TZ=Asia/Shanghai \
-e SW_STORAGE=elasticsearch7 \
-e SW_STORAGE_ES_CLUSTER_NODES=192.168.100.97:9200 \
-v /etc/localtime:/etc/localtime:ro \
apache/skywalking-oap-server:8.6.0-es7

// 安装skywalking ui服务
docker run \
--name skywalking-ui \
--restart always \
-p 8081:8080 -d \
--privileged=true \
--link skywalking-oap:skywalking-oap \
-e TZ=Asia/Shanghai \
-e SW_OAP_ADDRESS=192.168.100.97:12800 \
-v /etc/localtime:/etc/localtime:ro \
apache/skywalking-ui:8.6.0
```

### 下载Java agent
https://skywalking.apache.org/downloads/

### 启动时添加JVM参数
```markdown
-javaagent:D:\git_project\cloud-study\agent\skywalking-agent.jar   // agent地址
-Dskywalking.agent.service_name=cloud-producer                     // 服务名 
-Dskywalking.collector.backend_service=192.168.100.97:11800        // skywaling 采集地址
```

### 日志收集
springboot默认使用logback日志框架
##### 添加maven仓库
需要使用8.5.0以上版本，以下版本再添加logback配置时，启动会提示解析错误
```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>8.5.0</version>
</dependency>
```
#### 添加logback.xml文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  ~ Licensed to the Apache Software Foundation (ASF) under one or more
  ~ contributor license agreements.  See the NOTICE file distributed with
  ~ this work for additional information regarding copyright ownership.
  ~ The ASF licenses this file to You under the Apache License, Version 2.0
  ~ (the "License"); you may not use this file except in compliance with
  ~ the License.  You may obtain a copy of the License at
  ~
  ~      http://www.apache.org/licenses/LICENSE-2.0
  ~
  ~ Unless required by applicable law or agreed to in writing, software
  ~ distributed under the License is distributed on an "AS IS" BASIS,
  ~ WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  ~ See the License for the specific language governing permissions and
  ~ limitations under the License.
  -->
<configuration scan="true" scanPeriod=" 5 seconds">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
                <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%tid] [%thread] %-5level %logger{36} -%msg%n</Pattern>
            </layout>
        </encoder>
    </appender>

    <appender name="grpc-log" class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.log.GRPCLogClientAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
                <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] [%thread] %-5level %logger{36} -%msg%n</Pattern>
            </layout>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="grpc-log"/>
    </root>
</configuration>
```
#### 如果skywalking为安装在本地，需要添加jvm参数，或者修改agent.conf文件
```markdown
-Dskywalking.plugin.toolkit.log.grpc.reporter.server_host=192.168.100.97
-Dskywalking.plugin.toolkit.log.grpc.reporter.server_port=11800
-Dskywalking.plugin.toolkit.log.grpc.reporter.max_message_size=10485760
-Dskywalking.plugin.toolkit.log.grpc.reporter.upstream_timeout=30
```