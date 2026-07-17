+++
date = '2026-07-17T18:00:00+08:00'
draft = false
title = 'kafka_client支持GSSAPI认证啦'
+++


> AI预警！Rust预警！

## 前言

在[上一篇文章中](https://zzzdong.github.io/blog/i-wrote-a-kafka-client-lib/)不是介绍了纯 Rust 实现的 [kafka_client](https://crates.io/crates/kafka_client) 库么，有个小伙伴还去看了，并提了个[issue](https://github.com/zzzdong/kafka_client/issues/1)，说是没有支持 Kerberos 认证。

既然有人要，那我就写一个吧。

## 实现

同样的，纯 Rust 是我们的追求，所以，在查找了一轮后，我又开始准备自己手撸了。

同样的起手式，先AI对话问一下认证的流程和原理。依然是看着头疼，不管了，直接让它实现吧。在它干活的时候，自己找找文章看看 Kerberos 是怎样的。

在经过各种磨难后，终于在 0.5.0 版本中，支持了 Kerberos 认证。

项目的实现，采用 Rust-Crypto 的各种密码学库来实现各种算法。


## 磨难

这种功能的添加，最重要，就是先把集成测试先搭建起来。然后就可以让 AI 干活了。

中间也是遇到了不少问题，比如：

- 连接 KDC 获取 service ticket 失败，原因是 service principal 里 hostname 不一致————连接用 127.0.0.1 但 KDC 里注册的是 localhost。
- 好不容易获取到service ticket，client构建认证数据格式不对，导致一直认证失败。

反正我看着AI在修，也是感到头秃，在不断修，偶尔进入死胡同。修复时除了 CodeBuddy 上的 DeepSeek 和 HY3，也用 QWen 和 Kimi 来交叉代码审计。最后下载了不少RFC供AI阅读对照实现，终于跑通了。


## 思绪乱飞

Kerberos 认证的支持是比较有意义的，在一些企业应用中，可能就是需要。

在我看来，Kerberos 认证，就是引入一个第三方的认证中心，来验证用户的身份。客户端先去 kdc 获取 ticket，然后去 broker 拿到后，它再去 kdc 验证 ticket 的有效性。这样通过后，就完成认证了。

还有就是对于这些比较复杂的认证机制，它们一般都是有规范和标准的，在实现时，第一步就是要先下载好 RFC 文档，让 AI 对照规范去实现。同时这些涉及密码学的算法，它们的规范可能有一些测试数据，可以使用它们构建测试用例来保证代码的健壮性。

## 胡说八道

目前的LLM，知识广度和深度都是比很多人要强，但是在特定领域的，还是要依赖外部的资料，比如这次的RFC。所以写一些严格算法、实现时，可以考虑下载相关规范、文档给它。

当前的实现，只是通过了一个简易的集成测试，在生产环境中，还需要进一步的测试和验证。

> 你都路过了，顺手[给个星星嘛~](https://github.com/zzzdong/kafka_client/)
