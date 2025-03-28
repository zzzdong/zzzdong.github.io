+++
date = '2025-03-27T17:56:24+08:00'
draft = false
title = '来重试GRPC吧'
+++

## 前言

最近在摸一个etcd的Rust client，很自然地使用[tonic](https://github.com/hyperium/tonic)来做grpc的请求，但是发现tonic并[没有](https://github.com/hyperium/tonic/issues/733)提供一个重试机制，在token失效的时候，无法自动重试，于是就尝试摸索一把，看一看能不能搞定。

## 正文

先来看看tonic的文档，是可以用[Interceptor](https://docs.rs/tonic/latest/tonic/service/trait.Interceptor.html)来做拦截器的，里面更是建议用[tower](https://github.com/tower-rs/tower)机制来做中间件。这两个中间件的实现都有一个问题，就是不能重发数据，因为它们都是在grpc的transport层来做的，而且在transport层已经是只有一个不能Clone的`tonic::Request<tonic::body::Body>`，所以不能在错误时重发。

既然在grpc的底层transport太底层了不能重发，那么我们就看再上一层的。

好啦，我们观察grpc的请求的话，可以看到grpc的client和server交互存在4种模式：

1. unary
2. server stream
3. client stream
4. bidirectional stream

其中unary和server stream都是client发送一个message，然后等待服务器响应的。即发送`tonic::Request<prost::Message>`，`prost::Message`通常是一个可clone的struct。我们就可以通过Clone message，然后发送多次。

在查看tonic的代码时，我们可以找到[tonic::client::Grpc](https://docs.rs/tonic/latest/tonic/client/struct.Grpc.html)，其中它提供了对4种不同的模式发送消息。

在编程的世界中，任何问题都可以通过增加一个间接的中间层来解决。那么邪恶开始了，直接拷贝抽取tonic::client::Grpc来加一个trait，再实现它。

```rust
use prost::Message;
use tonic::{Request, Response, Status, Streaming};

pub trait GrpcService {
    async fn unary<M1: Message + Clone, M2: Message + Clone>(&self, request: Request<M1>, path: PathAndQuery) -> Result<Response<M2>, Status>;

    async fn server_stream<M1: Message + Clone, M2: Message>(&self, request: Request<M1>, path: PathAndQuery) -> Result<Response<Streaming<M2>>>, Status>;

    async fn client_stream<M1: Message, M2: Message>(&self, request: Request<Streaming<M1>>, path: PathAndQuery) -> Result<Response<M2>, Status>;

    async fn bidirectional_stream<M1: Message, M2: Message>(&self, request: Request<Streaming<M1>>, path: PathAndQuery) -> Result<Response<Streaming<M2>>>, Status>;
}
```

现在就可以实现GrpcService这个trait，其中底层调用tonic::client::Grpc。

```rust
pub struct TonicClient {
    inner: tonic::client::Grpc<tonic::transport::Channel>,
}

impl GrpcService for TonicClient {
    async fn unary<M1: Message + Clone, M2: Message + Clone>(&self, request: Request<M1>， path: PathAndQuery) -> Result<Response<M2>, Status> {
        self.inner.ready().await.map_err(|e| {
            crate::Error::new(
                crate::ErrKind::Grpc,
                format!("Service was not ready: {}", e),
            )
        })?;
        let codec = tonic::codec::ProstCodec::default();

        self.inner.unary(req, path, codec).await.map_err(Into::into)
    }
    ......
}

```

代价来啦，由于是重写了Grpc请求部分，所有tonic_build生成的代码就失效了，需要手动写了。例如：

```rust
#[derive(Debug, Clone)]
pub struct InnerKvClient<S> {
    service: S,
}
impl<S> InnerKvClient<S>
where
    S: GrpcService,
{
    pub fn new(service: S) -> Self {
        Self { service }
    }

    pub async fn range(
        &mut self,
        request: impl tonic::IntoRequest<pb::RangeRequest>,
    ) -> Result<tonic::Response<pb::RangeResponse>> {
        let path = http::uri::PathAndQuery::from_static("/etcdserverpb.KV/Range");
        self.service.unary(request.into_request(), path).await
    }
    ......
}
```

最后，回到开始需求：为etcd client添加一个自动刷新token的功能。在unary请求时，出现token失效的情况，就重新去申请token，并重新发起请求。

至于watch这样的streaming的请求，直接简单粗暴地在请求前不管三七二十一的先重新申请token再发送请求。

## 编外话


对具体实现的代码感兴趣的，可以[查看我的github](https://github.com/zzzdong/etcd-client-rust)。
