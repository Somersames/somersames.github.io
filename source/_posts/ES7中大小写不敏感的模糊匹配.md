---
title: ES7中大小写不敏感的模糊匹配
date: 2020-03-20 22:14:24
tags: [ElasticSearch]
categories: ElasticSearch
---
在ES7.0中

如果要实现大小写的模糊查询，则首先必须要自定义 `analysis`，在自定义的 `analysis` 里面，如果是针对keyword类型的字段， analysis 要定义成 normalizer，而对于text类型的话，则需要为analyzer。如下演示的是`normalizer`类型的定义。


> 新建索引
```json
{
  "settings": {
    "analysis": {
      "normalizer": {
        "self_normalizer": {
          "type": "custom",
          "char_filter": [],
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  },
  "mappings":{
    "properties":{
      "field_1":{
        "type":"keyword",
        "normalizer": "self_normalizer"
      }
    }
  }
}
```

>此时向ES中新增几条数据：
```json
{
	"took": 6,
	"timed_out": false,
	"_shards": {
		"total": 1,
		"successful": 1,
		"skipped": 0,
		"failed": 0
	},
	"hits": {
		"total": {
			"value": 4,
			"relation": "eq"
		},
		"max_score": 1.0,
		"hits": [
			{
				"_index": "test_ascii",
				"_type": "_doc",
				"_id": "2",
				"_score": 1.0,
				"_source": {
					"field_1": "abc"
				}
			},
			{
				"_index": "test_ascii",
				"_type": "_doc",
				"_id": "1",
				"_score": 1.0,
				"_source": {
					"field_1": "ABC"
				}
			},
			{
				"_index": "test_ascii",
				"_type": "_doc",
				"_id": "3",
				"_score": 1.0,
				"_source": {
					"field_1": "aBC"
				}
			},
			{
				"_index": "test_ascii",
				"_type": "_doc",
				"_id": "4",
				"_score": 1.0,
				"_source": {
					"field_1": "Abc"
				}
			}
		]
	}
}
```

可以看到此时的对于`field_1`，在Es中的值大小写都有的，此时进行模糊查询：
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "bool": {
            "must": [
              {
                "wildcard": {
                  "field_1": {
                    "value": "*a*",
                    "boost": 1
                  }
                }
              }
            ],
            "adjust_pure_negative": true,
            "boost": 1
          }
        }
      ],
      "adjust_pure_negative": true,
      "boost": 1
    }
  }
}
```
> 返回值
```java
{
	"took": 9,
	"timed_out": false,
	"_shards": {
		"total": 1,
		"successful": 1,
		"skipped": 0,
		"failed": 0
	},
	"hits": {
		"total": {
			"value": 4,
			"relation": "eq"
		},
		"max_score": 1.0,
		"hits": [
			{
				"_index": "test_ascii",
				"_type": "_doc",
				"_id": "2",
				"_score": 1.0,
				"_source": {
					"field_1": "abc"
				}
			},
			{
				"_index": "test_ascii",
				"_type": "_doc",
				"_id": "1",
				"_score": 1.0,
				"_source": {
					"field_1": "ABC"
				}
			},
			{
				"_index": "test_ascii",
				"_type": "_doc",
				"_id": "3",
				"_score": 1.0,
				"_source": {
					"field_1": "aBC"
				}
			},
			{
				"_index": "test_ascii",
				"_type": "_doc",
				"_id": "4",
				"_score": 1.0,
				"_source": {
					"field_1": "Abc"
				}
			}
		]
	}
}
```