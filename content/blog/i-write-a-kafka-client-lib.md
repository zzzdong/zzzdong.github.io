+++
date = '2026-07-07T21:18:00+08:00'
draft = false
title = '我用vibe开发了纯Rust实现的Kafka客户端库，性能还不错'
+++


> AI预警！Rust预警！胡说八道警告！

## 前言

我用vibe code的方式开发了一个纯Rust实现的[Kafka客户端](https://github.com/zzzdong/kafka_client)，性能还不错。在[测试](https://github.com/zzzdong/kafka-benchmark)中，发现比Rust包装的[rdkafka](https://crates.io/crates/rdkafka/0.39.0)还快了一些————它是C库[librdkafka](https://github.com/confluentinc/librdkafka)的封装。

在[上篇文章](https://zzzdong.github.io/blog/i-write-a-kafka-cli/)中介绍的kfk就是基于它来实现的。

## 有意思的 Kafka 协议

Kafka通信协议是挺有意思的，它基于自定义的二进制格式，在TCP的流上面，做了请求的多路复用。

- 使用length prefix来编码请求的长度，然后是请求的二进制数据。
- 使用correlation_id来关联请求和响应。
- 使用握手协商版本来确定每类请求的版本。
- 而编解码，又有一套flexible的编码格式， called "flexible encoding"。
- 消息体的定义，又是提供基于json的定义，方便开发。

不过，客户端和服务器的交互，需要客户端感知和处理的东西太多了，这点我不是很喜欢。

## 实现

在Rust的实现中，就是利用了json的消息定义，使用codegen的方式来生成消息体的struct。再利用Rust生成宏和trait来做编解码。

根据分层的架构，

- Transport层，负责底层连接，TCP和TLS。
- Wire层，负责消息的编码解码。使用Tokio的Frame来实现。
- Application层，在协议交互之上，负责Consumer和Producer的逻辑。

另外，提供了SASL协议验证，还有TLS通信。

## 性能比较

> 就问你快不快，至于是否正确，你就不要管了。

通过fork[rdkafka的benchmark](https://github.com/fede1024/kafka-benchmark)，加上kafka_client的benchmark，来对比性能。

结果可以查看[这里](https://zzzdong.github.io/kafka-benchmark/0.4.0_20260708/index.html)

AI给出的结论：

> ### 1. kafka_client 全面领先
> - 生产者平均提升 **336.8%**
> - 消费者 Assign 模式平均提升 **84.3%**
> - 消费者 Group 模式平均提升 **33.6%**
> 
> ### 2. 消息越小，优势越明显
> - 小消息场景（10B）kafka_client 优化效果最显著
> - 大消息场景瓶颈在网络 IO，客户端实现差异不明显
> 
> ### 3. Assign 模式性能碾压 Group
> - Assign 比 Group 快几十到几百倍
> - 如无消费组语义需求，优先使用 Assign 模式
> 
> ### 4. 数据完整性验证
> - 所有测试均通过消息计数、序列连续性、无重复验证



## 胡说八道

Vibe code盛行的今日，给二流的程序员提供了机会，也可以展示自己的能力。也让我这些可以输出代码，污染环境。

同时，Rust的特性，在AI agent中，好像更有利了一些，因为它的严格，以及性能。
