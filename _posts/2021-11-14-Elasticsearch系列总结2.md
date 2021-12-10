---
layout:     post
title:      Elasticsearch系列总结2
subtitle:   Elasticsearch基础知识
date:       2021-11-14
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - elasticsearch

---


# Elasticsearch系列总结2

Elasticsearch 使用教程
--

## 基本操作

### 索引操作

``` 
PUT /es_db

GET /es_db

DELETE /es_db
```
### 文档操作

put和post的区别：

PUT需要确定id才能进行操作，而POST是可以针对整个资源集合进行操作的，如果不写id就由ES生成一个唯一id进行创建，如果填了id那就针对这个id的文档进行创建/更新。

PUT只会将json数据都进行替换, POST只会更新相同字段的值。

PUT与DELETE都是幂等性操作。

``` 
PUT /es_db/_doc/1
{
"name": "张三",
"sex": 1,
"age": 25,
"address": "广州天河公园",
"remark": "java developer"
}

PUT /es_db/_doc/1
{
"name": "白起老师",
"sex": 1,
"age": 25,
"address": "张家界森林公园",
"remark": "php developer assistant"				
}

GET /es_db/_doc/1

DELETE /es_db/_doc/1

GET /es_db/_doc/_search?_source=name,age

GET /es_db/_doc/_search?q=age[25 TO 26]&from=0&size=1

GET /es_db/_doc/_search?sort=age:desc

GET /es_db/_doc/_mget 
{
 "ids":["1","2"]  
 }
```
### 批量操作

批量create是直接创建，批量index如果存在则是替换，否则创建。

``` 
GET _mget
{
"docs": [
{
"_index": "es_db",
"_type": "_doc",
"_id": 1
},
{
"_index": "es_db",
"_type": "_doc",
"_id": 2
}
]
}

POST _bulk
{"create":{"_index":"article", "_type":"_doc", "_id":3}}
{"id":3,"title":"白起老师1","content":"白起老师666","tags":["java", "面向对象"],"create_time":1554015482530}
{"create":{"_index":"article", "_type":"_doc", "_id":4}}
{"id":4,"title":"白起老师2","content":"白起老师NB","tags":["java", "面向对象"],"create_time":1554015482530}

POST _bulk
{"index":{"_index":"article", "_type":"_doc", "_id":3}}
{"id":3,"title":"图灵徐庶老师(一)","content":"图灵学院徐庶老师666","tags":["java", "面向对象"],"create_time":1554015482530}
{"index":{"_index":"article", "_type":"_doc", "_id":4}}
{"id":4,"title":"图灵诸葛老师(二)","content":"图灵学院诸葛老师NB","tags":["java", "面向对象"],"create_time":1554015482530}

POST _bulk
{"delete":{"_index":"article", "_type":"_doc", "_id":3}}
{"delete":{"_index":"article", "_type":"_doc", "_id":4}}

POST _bulk
{"update":{"_index":"article", "_type":"_doc", "_id":3}}
{"doc":{"title":"ES大法必修内功"}}
{"update":{"_index":"article", "_type":"_doc", "_id":4}}
{"doc":{"create_time":1554018421008}}
```
## 高级操作

### Query

``` 
POST /es_db/_doc/_search
{
"query": {
"term": {
"name": "admin"
}
}
}

POST /es_db/_doc/_search
{
"from": 0,
"size": 2, 
"query": {
"match": {
"address": "广州"
}
}
}

POST /es_db/_doc/_search
{
"query":{
"multi_match":{
"query":"张三",
"fields":["address","name"]
}
}
}

POST /es_db/_doc/_search
{
"query":{
"query_string":{
"query":"广州 OR 长沙"
}
}
}

POST /es_db/_doc/_search
{
"query" : {
"range" : {
"age" : {
"gte":25,
"lte":28
}
}
}
}

POST /es_db/_doc/_search
{
"query" : {
"range" : {
"age" : {
"gte":25,
"lte":28
}
}
},
"from": 0,
"size": 2,
"_source": ["name", "age", "book"],
"sort": {"age":"desc"}
}
```
### Filter
``` 
POST /es_db/_doc/_search
{
"query" : {
"bool" : {
"filter" : {
"term":{
"age":25
}
}
}
}
}
```

参考文献
--

图灵学院java架构课程