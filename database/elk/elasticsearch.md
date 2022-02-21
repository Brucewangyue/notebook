# Elasticsearch

## 索引 Index

* 对逻辑数据的逻辑存储

* 相当于关系型数据库中的表

* 索引可以存放在不同的机器上，不同机器上的同一索引称为分片（shard），每个分片可以有多个副本（replica）



## 文档 document

* 存储在es中的主要实体叫文档
* 相当于关系型数据库中的行
* 跟MongoDB的区别在于：es 中相同字段必须有相同类型
* 文档中的字段可重复出现，称为多值字段（multivalued）
* 字段类型：
  * 文本
  * 数值 integer
  * 日期
  * 关键字 keyword：该字段不会被分析，即使字符串包含多个单词，它们也被视为单个单元
  * 经纬度 geo_point
  * 复杂类型（子文档）
* 文档类型：
  * 一个索引对象可以存储很多不同用途的对象，例如在一个索引中同时存储博客数据和评论数据
  * 每个文档可以有不同的结构
  * 还是遵循同一字段名不能设置为不同类型



## 映射 mapping

* 所有文档写进索引之前都会先进行分析，如何将输入的文本分割为词条、哪些词条会被过滤、这种行为叫映射

* 实例：

  映射为数据集提供了字段特征

  ```shell
  curl -X PUT "0.0.0.0:9200/shakespeare?pretty" -H 'Content-Type: application/json' -d'
  {
    "mappings": {
      "properties": {
      "speaker": {"type": "keyword"},
      "play_name": {"type": "keyword"},
      "line_id": {"type": "integer"},
      "speech_number": {"type": "integer"}
      }
    }
  }
  '
  ```

  



## API





## 配置

[官方配置文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/important-settings.html)

* **添加跨域**

```json
http.cors.enabled:true
http.cors.allow-origin:"*"
```

