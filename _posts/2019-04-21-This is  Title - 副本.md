---
layout:     post
title:      Kafka系列总结1
subtitle:   Docker环境安装Kafka
date:       2021-11-09
author:     Yikun Chen
header-img: img/art-Anaconda-TensorFlow.jpg
catalog: true
tags:
    - kafka
    - docker
    - installation

---


# Kafka系列总结1

Docker环境安装Kafka
--

Kafka的安装总共分为三个部分，分别是zookeeper，kafka以及kafka-manager的安装。

---

## 安装Zookeeper

zookeeper的安装需要暴露出2181端口供外界使用，

```bash
docker run -d --name zookeeper --publish 2181:2181 --volume /etc/localtime:/etc/localtime --restart=always wurstmeister/zookeeper
```

## 安装Kafka

kafka的安装需要指定本机的ip host，查看方法是在命令行输入ipconfig，查看其中的ipv4地址。

```bash
docker run -d --name kafka --publish 9092:9092 --link zookeeper --env KAFKA_ZOOKEEPER_CONNECT={your host ip}:2181 --env KAFKA_ADVERTISED_HOST_NAME={your host ip} --env KAFKA_ADVERTISED_PORT=9092  --volume /etc/localtime:/etc/localtime wurstmeister/kafka:2.11-0.11.0.3
```

## 安装Kafka-manager

Kafka-manager主要是一个管理界面，方便查看信息。

```bash
docker run -d --name kafka-manager --link zookeeper:zookeeper --link kafka:kafka -p 9001:9000 --restart=always --env ZK_HOSTS=zookeeper:2181 sheepkiller/kafka-manager
```


## 安装成果

![picture1](/img/kafka/installation.png)  

参考文献
--

[docker下安装kafka和kafka-manager](https://www.cnblogs.com/brady-wang/p/13757713.html)
[Docker 安装 kafka 单机版本](https://www.icode9.com/content-4-1040800.html)