---
layout:     post
title:      Elasticsearch系列总结3
subtitle:   Elasticsearch基础知识
date:       2021-11-14
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - elasticsearch

---


# Elasticsearch系列总结3

Elasticsearch基础知识
--

## 基本概念

Elasticsearch是用Java开发并且是当前最流行的开源的企业级搜索引擎。能够达到实时搜索，稳定，可靠，快速，安装使用方便。客户端支持Java、.NET（C#）、PHP、Python、Ruby等多种语言。

### 数据库概念对比

与数据库的概念对比如下图所示。

![picture1](/img/elasticsearch/database.png)

索引 index。一个索引就是一个拥有几分相似特征的文档的集合。比如说，可以有一个客户数据的索引，另一个产品目录的索引，还有一个订单数据的索引。一个索引由一个名字来标识（必须全部是小写字母的），并且当我们要对对应于这个索引中的文档进行索引、搜索、更新和删除的时候，都要使用到这个名字。

映射 mapping。 ElasticSearch中的映射（Mapping）用来定义一个文档。mapping是处理数据的方式和规则方面做一些限制，如某个字段的数据类型、默认值、分词器、是否被索引等等，这些都是映射里面可以设置的。

字段Field。相当于是数据表的字段|列。

字段类型 Type。每一个字段都应该有一个对应的类型，例如：Text、Keyword、Byte等。

文档 document。一个文档是一个可被索引的基础信息单元，类似一条记录。文档以JSON（Javascript Object Notation）格式来表示。

集群 cluster。一个集群就是由一个或多个节点组织在一起，它们共同持有整个的数据，并一起提供索引和搜索功能。

节点 node。一个节点是集群中的一个服务器，作为集群的一部分，它存储数据，参与集群的索引和搜索功能。一个节点可以通过配置集群名称的方式来加入一个指定的集群。默认情况下，每个节点都会被安排加入到一个叫做“elasticsearch”的集群中。这意味着，如果在网络中启动了若干个节点，并假定它们能够相互发现彼此，它们将会自动地形成并加入到一个叫做“elasticsearch”的集群中。在一个集群里，可以拥有任意多个节点。而且，如果当前网络中没有运行任何Elasticsearch节点，这时启动一个节点，会默认创建并加入一个叫做“elasticsearch”的集群。
### 全文检索

- 通过一个程序扫描文本中的每一个单词，针对单词建立索引，并保存该单词在文本中的位置、以及出现的次数。
- 用户查询时，通过之前建立好的索引来查询，将索引中单词对应的文本位置、出现的次数返回给用户，因为有了具体文本的位置，所以就可以将具体内容读取出来了。

### 倒排索引

与传统数据库不同，由对应索引包含的数据转变成对应数据可以在哪些索引中找到。

![picture1](/img/elasticsearch/index.png)


参考文献
--