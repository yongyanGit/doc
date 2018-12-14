### Elasticsearch 基本查询语法

#### 基本概念

* Index ：一系列文档(Document)的集合，类似于mysql中数据库的概念。
* Type：在Index里面可以定义不同的type，type类似于mysql中表的概念，是一系列具有相同特征数据的集合，一般一个Index中只有一种type。
* Document：文档的概念类似于mysql中的一条记录，它的格式是json格式。
* Shards：在数据量很大的时候，进行水平扩展，提高搜索性能。

#### 查询表达式

要使用查询表达式，只需要将查询语句传递给```query```参数：

如下使用```match_all```查询，匹配所有文档，与空查询效果一致。

```json
GET /_search
{
    "query": {
        "match_all": {}
    }
}
```

比如通过match匹配姓名

```json
post /loan_log/loan_log_type/_search/

{
	"query":{
		"match":{"Name":"WANGYANZHE611"}
		}
}
```

查看index结构

```
GET /loan_log/loan_log_type/_mapping?pretty/
```



