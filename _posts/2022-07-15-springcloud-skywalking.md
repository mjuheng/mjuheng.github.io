---
layout: post
title: SpringCloud集成skywalking
categories: [SpringCloud]
description: SpringCloud集成skywalking
keywords: SpringCloud, skywalking
---

SpringCloud集成skywalking

## 环境搭建
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
-e SW_ES_USER=es的用户名
-e SW_ES_PASSWORD=es的密码
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

### 进入docker容器
```markdown
docker exec -it 容器id /bin/bash
```

### 下载Java agent
https://skywalking.apache.org/downloads/

### 启动时添加JVM参数
```markdown
-javaagent:D:\git_project\cloud-study\agent\skywalking-agent.jar   // agent地址
-Dskywalking.agent.service_name=cloud-producer                     // 服务名 
-Dskywalking.collector.backend_service=192.168.100.97:11800        // skywaling 采集地址
```

## 日志采集
### 自定义SkyWalking链路追踪（个别方法不会上报到OAP服务中）
自定义追踪即是在每个(或者想要的）请求上加上请求参数和返回参数的获取
注意：
① 这里是加在业务层,即@Service层，不要加在@Controller，没有用的。如果一个业务方法想在ui界面的跟踪链路上显示出来，只需要在业务方法上加上@Trace注解即可。
② 加入@Tags或@Tag 我们还可以为追踪链路增加其他额外的信息，比如记录参数和返回信息。实现方式：在方法上增加@Tag或者@Tags。 @Tag 注解中 key = 方法名 ； value = returnedObj 返回值 arg[0] 参数
这里注意returnedObj 是固定写法
#### 添加maven仓库
```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>8.5.0</version>
</dependency>
```
#### 示例
```java
@Trace
@Tags({@Tag(key = "checkMessage", value = "returnedObj"), @Tag(key = "message", value = "arg[0]")})
public String checkMessage(String message) {
    return remoteConsumerService.checkMessage(message);
}
```

### 输出日志获取
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


## 链路告警
```markdown
在容器config/alarm-settings.yml文件中

SkyWalking 的发行版都会默认提供config/alarm-settings.yml文件，里面预先定义了一些常用的告警规则。如下：

过去 3 分钟内服务平均响应时间超过 1 秒。
过去 2 分钟服务成功率低于80%。
过去 3 分钟内服务响应时间超过 1s 的百分比
服务实例在过去 2 分钟内平均响应时间超过 1s，并且实例名称与正则表达式匹配。
过去 2 分钟内端点平均响应时间超过 1 秒。
过去 2 分钟内数据库访问平均响应时间超过 1 秒。
过去 2 分钟内端点关系平均响应时间超过 1 秒。
这些预定义的告警规则，打开config/alarm-settings.yml文件即可看到
告警规则配置项的说明：
1.Rule name：规则名称，也是在告警信息中显示的唯一名称。必须以_rule结尾，前缀可自定义
2. Metrics name：度量名称，取值为oal脚本中的度量名，目前只支持long、double和int类型。详见Official OAL script
3.Include names：该规则作用于哪些实体名称，比如服务名，终端名（可选，默认为全部）
4.Exclude names：该规则作不用于哪些实体名称，比如服务名，终端名（可选，默认为空）
5.Threshold：阈值
6. OP： 操作符，目前支持 >、<、=
7. Period：多久告警规则需要被核实一下。这是一个时间窗口，与后端部署环境时间相匹配
8.Count：在一个Period窗口中，如果values超过Threshold值（按op），达到Count值，需要发送警报
9.Silence period：在时间N中触发报警后，在TN -> TN + period这个阶段不告警。 默认情况下，它和Period一样，这意味着相同的告警（在同一个Metrics name拥有相同的Id）在同一个Period内只会触发一次
10.message：告警消息

Webhook可以简单理解为是一种Web层面的回调机制，通常由一些事件触发，与代码中的事件回调类似，只不过是Web层面的。由于是
Web层面的，所以当事件发生时，回调的不再是代码中的方法或函数，而是服务接口。例如，在告警这个场景，告警就是一个事件。当该
事件发生时，SikyWalking就会主动去调用一个配置好的接口，该接口就是所谓的Webhook。 SkyWalkng的告警消息会通过
HTTP 请求进行发送，请求方法为 POST，Content-Type 为 application/json，其JSON 数据实基于
List<org.apache.skywalking.oap.server.core.alarm.AlarmMessage进行序列化的

接受样例:
[
    {
    "scopeId": 2,
    "scope": "SERVICE_INSTANCE",
    "name": "59a8a26b81074f61a008b0915c636b77@192.168.99.53 of cloud-producer",
    "id0": "Y2xvdWQtcHJvZHVjZXI=.1_NTlhOGEyNmI4MTA3NGY2MWEwMDhiMDkxNWM2MzZiNzdAMTkyLjE2OC45OS41Mw==",
    "id1": "",
    "ruleName": "service_instance_resp_time_rule",
    "alarmMessage": "Response time of service instance 59a8a26b81074f61a008b0915c636b77@192.168.99.53 of cloud-producer is more than 1000ms in 2 minutes of last 10 minutes",
    "tags": [ ],
    "startTime": 1658372716763
    },
    {
    "scopeId": 1,
    "scope": "SERVICE",
    "name": "cloud-producer",
    "id0": "Y2xvdWQtcHJvZHVjZXI=.1",
    "id1": "",
    "ruleName": "service_resp_time_rule",
    "alarmMessage": "Response time of service cloud-producer is more than 1000ms in 3 minutes of last 10 minutes.",
    "tags": [ ],
    "startTime": 1658372716763
    },
    {
    "scopeId": 1,
    "scope": "SERVICE",
    "name": "cloud-producer",
    "id0": "Y2xvdWQtcHJvZHVjZXI=.1",
    "id1": "",
    "ruleName": "service_resp_time_percentile_rule",
    "alarmMessage": "Percentile response time of service cloud-producer alarm in 3 minutes of last 10 minutes, due to more than one condition of p50 > 1000, p75 > 1000, p90 > 1000, p95 > 1000, p99 > 1000",
    "tags": [ ],
    "startTime": 1658372716763
    }
]
```