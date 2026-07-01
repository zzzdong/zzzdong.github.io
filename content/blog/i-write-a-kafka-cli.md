+++
date = '2026-07-01T22:14:19+08:00'
draft = false
title = '我用 Rust 写了个 Kafka 命令行工具和 Rust 客户端'
+++

## 前言

近来我用rust写了一些小工具，比如这个[kfk](https://github.com/zzzdong/kfk)工具，来帮助我管理kafka集群。

## 功能

它是Rust实现，可以musl编译成静态二进制文件，没有外部依赖。可以自由在大部分的Linux上运行。


```shell
kfk --help
A pure Rust Kafka CLI tool for cluster management, topic operations, and message produce/consume

Usage: kfk [OPTIONS] <COMMAND>

Commands:
  config      Cluster configuration management
  configs     List all configured clusters (shorthand)
  node        Cluster node operations
  nodes       List all brokers (shorthand)
  topic       Topic operations
  topics      List all topics (shorthand)
  produce     Produce messages to a topic (reads from stdin)
  consume     Consume messages from a topic
  group       Consumer group operations
  groups      List all consumer groups (shorthand)
  completion  Generate shell completion script
  help        Print this message or the help of the given subcommand(s)
```

提供常见的基本操作，在简易操作Kafka集群时，可以使用这个工具。比如: 测试集群是否正常，消费消息来调试等。

欢迎各位使用，可以从[github](https://github.com/zzzdong/kfk/releases/)下载，我打包了Linux上的可执行文件。

## 一点故事

一开始，并没有想到写这个工具，初衷是我想要一个Rust的可以方便编译的Kafka客户端，来连接Kafka集群做消息的生产者和消费者。

当前Rust生态中，推荐的是[rdkafka](https://crates.io/crates/rdkafka)，它是对librdkafka的Rust绑定。当我尝试使用rdkafka时，想做musl编译，但是遇到了一些问题，它依赖于openssl库————出名的难以交叉编译。

在看到了一些其他的Rust Kafka客户端，比如[kafka](https://crates.io/crates/kafka)，它是一个纯Rust实现的Kafka客户端，没有依赖于任何外部库。但是它居然不支持SASL验证。

既然这样，不如我自己实现一个吧。

在AI的帮助下，磨了不少的时间，断断续续，终于实现了[kafka_client](https://github.com/zzzdong/kafka_client)。

它是一个纯Rust实现的Kafka客户端，没有依赖于任何外部库。它支持SASL验证和TLS连接。虽然它的间接依赖库如rustls和zstd是需要编译C，但是在kfk编译验证发现不用额外的操作，只要有musl工具链就可以编译成功。

## 一些胡思乱想

在现在的AI盛行时代，很多时候，只需要一些想法，加以AI的辅助，就可以完成不错的结果。比如kfk这个命令行工具，差不多就是全部都是AI编写的，kafka_client也是大量的使用AI的辅助。

那么，程序员的价值在哪里？是不是古法编程这些技术就要被废弃了？另外程序员应该面对AI？

前不久，公司来了个实习生，在他口中，全部都直接给AI就可以了。对此我有些异议的。当前，全部依赖于AI，还是有些欠缺，可能是目前的AI技术还不够成熟，总感觉做得不够好。也可能是我自己想多了吧。

另外，能看到的是，在AI的冲击下，程序员的角色正在发生改变。而且，新入门越来越难了，因为AI已经比很多初级程序员更专业了。像我，感觉自己就是一个AI的人形agent，做着一些AI在电脑上暂时无法完成的任务，遇到问题，就是无脑问AI。

---------------

PS. Kafka自带的命令行真是难用的，我每次都要去查文档，而且它的引用config文件的参数还是不统一的。

PS. Go写的[kaf](https://github.com/birdayz/kaf)命令行工具挺不错的，kfk受了它的启发。
