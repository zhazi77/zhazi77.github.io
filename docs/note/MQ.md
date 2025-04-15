---
draft: true
date:
  created: 2025-03-09
categories:
  - Learning
  - Memo
tags:
  - MQ
authors:
  - zhazi
---


# 青训营笔记：消息队列


记录学习消息队列过程中的思考与实践。

**组织逻辑：** 先简要介绍一下 MQ，然后介绍一下常用的 MQ，然后对几个重要的 MQ 进行详细介绍

# TODO: 阅读一下 Git 用户手册

??? info "参考文献"

    - 青训营课程
    - [Git 用户手册](https://git-scm.com/docs/user-manual)

<!-- more -->

## 消息队列的应用场景

### 场景一：系统崩溃

解决方案：解耦

### 场景二：服务能力有限

解决方案：削峰

### 场景三：链路耗时长尾

解决方案：异步

### 场景四：本地日志故障

解决方案：???


## 消息队列简介

!!! qoute "摘录"

    消息队列，指保存消息的一个容器，本质是一个队列。但这个队列需要支持**高吞吐、高并发、并且高可用**。

### 消息队列发展历程

### 业界消息队列对比

- Kafka
- RocketMQ
- Pulsar
- BMQ

## Kafka

### 使用场景

- 日志信息
- Metrics 数据
- 用户行为

### 如何使用 Kafka

- 创建集群
- 新增 topic
- 编写生产者逻辑
- 编写消费者逻辑

### 基本概念

- Topic:
- Cluster:
- Producer:
- Consumer:
- ConsumerGroup:
- Partition:
- Offset:
- Replica:

### 数据复制

### KafKa 架构
