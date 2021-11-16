---
title: Kafka常用命令
date: 2021-11-16 16:10:51
tags:
- Kafka
categories:
- Kafka
---

* 创建Topic

  `./kafka-topics.sh --zookeeper bigdata1:2181 --create --topic test--partitions 3 --replication-factor 3`

* 发送数据

  `./kafka-console-producer.sh --broker-list bigdata1:6667 --topic test` 

* 消费数据

  `./kafka-console-consumer.sh --bootstrap-server bigdata1:6667 --topic test`

* 查看Topic列表

  `./kafka-topics.sh --zookeeper bigdata1:2181 --list`

* 删除Topic

  `./kafka-topics.sh --delete --zookeeper bigdata1:2181 --topic test`

* 查看Topic描述

  `./kafka-topics.sh --zookeeper bigdata1:2181 --topic test --describe`

* 修改Topic

  `./kafka-topics.sh --alter --zookeeper bigdata1:2181 --topic test--partitions 3`

