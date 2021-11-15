---
layout:     post
title:      Elasticsearch系列总结1
subtitle:   Docker环境安装Elasticsearch
date:       2021-11-14
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - elasticsearch
    - docker
    - installation

---


# Elasticsearch系列总结1

Docker环境安装Elasticsearch
--

Elasticsearch的安装总共分为三个部分，分别是Elasticsearch，以及kibana安装。

---

## 安装Elasticsearch

Elasticsearch的安装首先创建一个network用于内部通信，然后暴露出9200和9300供外部使用。

```bash
docker network create elasticsearch_net

docker run -d --name elasticsearch -p 9200:9200 -p 9300:9300 --network elasticsearch_net -v elasticsearch_volume:/root -e "privileged=true" -e "discovery.type=single-node" elasticsearch:7.14.2
```

## 安装kibana
```bash
docker run -d --name kibana --network elasticsearch_net -e ELASTICSEARCH_URL=http://192.168.2.11:9200 -p 5601:5601 kibana:7.14.2
```

## 安装成果

![picture1](/img/elasticsearch/installation.png)  

参考文献
--

[docker安装elasticsearch和kibana](https://segmentfault.com/a/1190000022831545)

