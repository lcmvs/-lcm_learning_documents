# Kafka

kafka是用于构建实时数据管道和流应用程序。具有横向扩展，容错，wicked fast（变态快）等优点，并已在成千上万家公司运行。

1. **发布和订阅**消息(流)，在这方面，它类似于一个消息队列或企业消息系统。

2. 以容错(故障转移)的方式**存储**消息(流)。

3. 在消息流发生时**处理**它们。

## 核心api

- 应用程序使用 [Producer API]() 发布消息到1个或多个topic（主题）中。
- 应用程序使用 [Consumer API]() 来订阅一个或多个topic，并处理产生的消息。
- 应用程序使用 [Streams API]() 充当一个流处理器，从1个或多个topic消费输入流，并生产一个输出流到1个或多个输出topic，有效地将输入流转换到输出流。
- [Connector API]() 可构建或运行可重用的生产者或消费者，将topic连接到现有的应用程序或数据系统。例如，连接到关系数据库的连接器可以捕获表的每个变更。

![kafka入门介绍](assets/KmCudlf7DXiAVXBMAAFScKNS-Og538.png)

  

## 基本术语

### Topic

Kafka将消息分门别类，每一类的消息称之为一个主题（Topic）。

### Producer

发布消息的对象称之为主题生产者（Kafka topic producer）

### Consumer

订阅消息并处理发布的消息的对象称之为主题消费者（consumers）

### Broker

已发布的消息保存在一组服务器中，称之为Kafka集群。集群中的每一个服务器都是一个代理（Broker）。 消费者可以订阅一个或多个主题（topic），并从Broker拉数据，从而消费这些已发布的消息。

