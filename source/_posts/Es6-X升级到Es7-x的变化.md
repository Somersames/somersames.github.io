---
title: Es6.X升级到Es7.x的变化
date: 2019-07-07 00:35:24
tags: [ElasticSearch]
categories: [ElasticSearch]
---
ElasticSearch6.升级至ElasticSearch7.x的一些变化

由于最近需要将`Es6.x`升级至`Es7`，所以正好记录下在升级过程中遇到的一些问题，以便以后翻阅。


## 区别

### Es7.x系列中取消了Type

在`Es6`系列之前，创建一个索引是需要`index,type`这两个缺一不可的，例如如下请求：

```json
 PUT localhost:9200/es_6     
{
    "mappings":{
        "index_type":{
            "properties":{
                "message":{
                    "type":"text"
                }
            }
        }
    }
}


response:
{
    "acknowledged": true,
    "shards_acknowledged": true,
    "index": "es_6"
}
```

但是在 `ES7` 版本中，如果再使用这个 Json 串的话是会跑出一个异常的，如下：

```json
{
    "error": {
        "root_cause": [
            {
                "type": "mapper_parsing_exception",
                "reason": "Root mapping definition has unsupported parameters:  [index_type : {properties={message={type=text}}}]"
            }
        ],
        "type": "mapper_parsing_exception",
        "reason": "Failed to parse mapping [_doc]: Root mapping definition has unsupported parameters:  [index_type : {properties={message={type=text}}}]",
        "caused_by": {
            "type": "mapper_parsing_exception",
            "reason": "Root mapping definition has unsupported parameters:  [index_type : {properties={message={type=text}}}]"
        }
    },
    "status": 400
}
```



那么此时在`ES7`版本中，建立 mapping 是不需要 Type 的，所以其索引修改为下：

```json
PUT localhost:9201/es_7

{
    "mappings":{
        "properties":{
            "message":{
                "type":"text"
            }
        }
    }
}

{
    "acknowledged": true,
    "shards_acknowledged": true,
    "index": "es_7"
}
```



### ES7.x中新建数据

在 ES6 中由于有一个 Type 类型，因此在新建数据的时候都需要穿入一个Type，那么在 Es7 里面，由于 Type 被取消了，所以在 ES7 里面的新增就需要稍微修改下了。

```json
POST localhost:9201/es_7/_create/1

{
	"message":"a"
}

{
    "_index": "es_7",
    "_type": "_doc",
    "_id": "1",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```

其实还有另一种写法：

```json
POST localhost:9201/es_7/_doc/2?op_type=create
```

剩下的一些改动可能就是新的业务上线需要对某些数据进行频繁的改动，而ES的乐观锁机制导致经常失败，这个问题得需要单独处理下