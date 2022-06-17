---
layout: post
title: SpringBoot集成ElasticSearch
categories: [Spring]
description: SpringBoot集成ElasticSearch
keywords: Docker, ElasticSearch
---

SpringBoot集成ElasticSearch

# 引入依赖
```xml
<!--elasticsearch-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

# 添加配置文件
```yaml
spring:
  elasticsearch:
    uris: 192.168.100.97:9200
```

# 添加配置类，引入RestHighLevelClient
```java
import org.elasticsearch.client.RestHighLevelClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.client.ClientConfiguration;
import org.springframework.data.elasticsearch.client.RestClients;

import java.util.List;

@Configuration
@ConditionalOnProperty(name = "spring.elasticsearch.uris")
public class EsConfig {

    @Value("${spring.elasticsearch.uris}")
    private List<String> esLogServer;

    @Bean
    public RestHighLevelClient elasticsearchClient() {

        final ClientConfiguration clientConfiguration = ClientConfiguration.builder()
                .connectedTo(esLogServer.toArray(new String[0]))
                .build();

        return RestClients.create(clientConfiguration).rest();

    }
}
```

# 样例
Spring Boot提供高级接口，操作ElasticSearch就像写数据库的CRUD一样简单！
## 创建实体类
```java
import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

import java.io.Serializable;
import java.util.Date;

@Data
@Document(indexName = "student_info")
public class StudentInfo implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    private String id;

    private String name;

    private String xh;

    @Field(type = FieldType.Date)
    private Date enterTime;

    public StudentInfo() {

    }

    public StudentInfo(String id, String name, String xh, Date enterTime) {
        this.id = id;
        this.name = name;
        this.xh = xh;
        this.enterTime = enterTime;
    }
}
```
### 注解属性解析
@Document：在类级别应用以指示该类是映射到数据库的候选对象 
    indexName：存储此实体的索引的名称。这可以包含一个 SpEL 模板表达式，如"log-# 
    shards：索引的分片数 
    replicas：索引的副本数 
    createIndex: 标记是否在存储库引导时创建索引。默认值为true

@Id：应用于字段级别以标记用于标识目的的字段

@Transient：默认情况下，所有字段在存储或检索时都映射到文档，此注释不包括该字段。

@Field：应用于字段级别并定义字段的属性，大部分属性映射到各自的Elasticsearch Mapping定义 
    name：将在 Elasticsearch 文档中表示的字段名称，如果未设置，则使用 Java 字段名称 
    type：字段类型 
    format以及Date类型pattern的定义 
    store: 标记原始字段值是否应该存储在 Elasticsearch 中，默认值为false。

## 创建Mapper
集成自ElasticsearchRepository接口，提供了基础的增删改方法
```java
import com.studyself.elasticsearch.pojo.StudentInfo;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;

public interface StudentInfoMapper extends ElasticsearchRepository<StudentInfo, String> {
    
}
```

## 创建Service实现增删改查逻辑
```java
@Service
public class StudentInfoBiz {

    @Resource
    private StudentInfoMapper studentInfoMapper;
    @Resource
    private RestHighLevelClient restHighLevelClient;

    public void saveAll(List<StudentInfo> studentInfoList) {
        studentInfoMapper.saveAll(studentInfoList);
    }

    public void deleteById(String id) {
        studentInfoMapper.deleteById(id);
    }

    public void updateById(StudentInfo studentInfo) {
        studentInfoMapper.save(studentInfo);
    }

    public Map<String, Object> findList(StudentInfo studentInfo, Integer pageIndex, Integer pageSize) {

        Map<String, Object> dataMap = new HashMap<>();

        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder().trackTotalHits(true);
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        if (studentInfo.getName() != null) {
            boolQueryBuilder.filter(QueryBuilders.wildcardQuery("name", String.format("*%s*", studentInfo.getName())));
        }
        if (pageIndex != null && pageSize != null) {
            searchSourceBuilder.from((pageIndex - 1) * pageSize).size(pageSize);
        }
        searchSourceBuilder.query(boolQueryBuilder);


        SearchRequest searchRequest = new SearchRequest("student_info");
        searchRequest.source(searchSourceBuilder);
        try {
            SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);

            List<StudentInfo> result = new ArrayList<>();
            for (SearchHit hit : response.getHits()) {
                result.add(JSON.parseObject(hit.getSourceAsString(), StudentInfo.class));
            }


            dataMap.put("total", response.getHits().getTotalHits().value);
            dataMap.put("result", result);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return dataMap;
    }


    public StudentInfo findById(String id) {
        return studentInfoMapper.findById(id).orElse(null);
    }

    public void deleteAll() {
        studentInfoMapper.deleteAll();
    }

}
```
### 查询参数解析
QueryBuilders.matchQuery() 会根据分词器进行分词，分词之后去查询 
QueryBuilders.termQuery() 不会进行分词，且完全等于才会匹配 
QueryBuilders.termsQuery() 一个字段匹配多个值，where name = ‘A’ or name = ‘B’ 
QueryBuilders.multiMatchQuery() 会分词 一个值对应多个字段 where username = ‘zs’ or password = ‘zs’ 
QueryBuilders.matchPhraseQuery() 不会分词，当成一个整体去匹配，相当于 %like% 
如果想使用一个字段匹配多个值，并且这多个值是and关系，如下 要求查询的数据中必须包含北京‘和’天津QueryBuilders.matchQuery(“address”,“北京 天津”).operator(Operator.AND)
如果想使用一个字段匹配多个值，并且这多个值是or关系，如下 要求查询的数据中必须包含北京‘或’天津 
QueryBuilders.matchQuery(“address”,“北京 天津”).operator(Operator.OR)


# REST API操作ElasticSearch
### 创建索引
http://ip:port/索引   put
{
    "settings": {
        "number_of_shards": 3,
        "number_of_replicas": 1
    },
    "mappings": {
        "properties": {
            "name": {
                "type": "text"
            },
            "country": {
                "type": "keyword"
            },
            "age": {
                "type": "integer"
            },
            "date": {
                "type": "date",
                "format": "yyyy-MM-dd HH:mm:ss"
            }
        }
    }
}
### 新增数据
http://ip:port/索引/类型/id  put(指定id)
http://ip:port/索引/类型     post(自动生成随机id)
{
"name": "张三",
"country": "China"
}
### 查询全部索引
http://ip:port/_cat/indices?v
### 查询索引数据
http://ip:port/索引名/_search?pretty