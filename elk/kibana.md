# Kibana

## 索引模式 index pattern

索引模式告诉Kibana你想要探索哪些Elasticsearch索引。索引模式可以匹配单个索引的名称，也可以包含一个通配符(*)来匹配多个索引

例如： Logstash 通常以logstash- yyyy.mm.dd格式创建一系列索引。要浏览2018年5月以来的所有日志数据，可以指定索引模式logstash-2018.05*。