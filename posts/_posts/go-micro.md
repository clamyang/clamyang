---
title: 初探 go-micro 微服务框架
date: 2023-02-16
comments: true
---

最近想了解下微服务，我们项目中目前还是单体服务，服务间的调用也都是通过 http 进行通信的，但这并不妨碍我们对微服务的好奇心，之前有听过几个微服务框架的名字，B站的 Kratos，字节的 Kitex，好未来的 go-zero 等等，但这几个并不是本篇的主题。具体哪个微服务框架并不重要，主要就是借着这个机会了解一下微服务相关的内容。

<!--more-->

我以国外的微服务框架 go-micro 作为学习资料，项目中包含了一个完整的微服务 demo，可以把代码 pull 下来在本地运行起来，地址在这 [micro-demo](https://github.com/go-micro/demo) 。



把这个微服务跑起来需要把环境准备好，安装 Docker，Kind，Skaffold。以 Docker 作为容器运行时，搭建本地 k8s 集群，最后通过 skaffold 应用 yaml 文件，创建出 Deployment 和 Service。这些步骤完成后，我们可以愉快的访问 demo 的主页了。



作为一个纯小白来说，理解这个项目还是比较麻烦的，上来我就遇到了两个问题：

1. http 调用的时候，我们有客户端和服务端，通过域名解析可以拿到服务端的地址，然后进行访问。但是这个 go-micro 的某一个项目中，没有看到任何其他服务的地址，只包含其他服务的名称。不禁发问，这是怎么调用到另一个服务的？怎么拿到被调用服务的地址的？
2. 调用者是怎么知道服务端提供哪些服务的呢？

对于使用过微服务框架的老司机来说，就没必须要继续往下看了，适合小白玩家，毕竟我也才了解。

我的服务是跑在 k8s 集群中，因为 service 的名字和服务的名字是一样的，我就猜测调用方直接对服务名称发起请求，利用 service 的特性，最后代理到 pod 上。

那如果是在本地运行的服务，多个服务之间是如何进行通信的呢？

本地服务启动的时候会有如下的日志输出

![](https://s2.loli.net/2023/02/16/AxIz9ieRpwfUlHV.png)

可以注意到，这个 currencyservice 服务注册到了 mdns 中，那这又是怎么一回事情呢？该怎么继续探索这个框架呢？毕竟在服务启动的时候并没有这部分的内容。



利用编辑器直接搜索这行输出，那肯定是摘出来一部分，全局搜索 Registry [mdns] 没搜到的话，把 mdns 换成 %s 试试，一般来讲括号里的内容都是不固定的。

![](https://s2.loli.net/2023/02/16/kZq9cQ5RnlWTGVP.png)

这样就找到了代码位置，顺着这个找到函数开头的部分，可以看到作者抽象出来的服务发现接口。

```go
// The registry provides an interface for service discovery
// and an abstraction over varying implementations
// {consul, etcd, zookeeper, ...}
type Registry interface {
   Init(...Option) error
   Options() Options
   Register(*Service, ...RegisterOption) error
   Deregister(*Service, ...DeregisterOption) error
   GetService(string, ...GetOption) ([]*Service, error)
   ListServices(...ListOption) ([]*Service, error)
   Watch(...WatchOption) (Watcher, error)
   String() string
}
```

go-micro 框架中默认有以下几种服务发现的方式：

![](https://s2.loli.net/2023/02/16/y12ClzqDSZh689L.png)

第一个是 cache 用来加速 dns 的解析，第二个是 k8s 集群中的服务发现，第三个就是本地启动服务的时候用到的 dns 解析服务。最后那个顾名思义，是在内存中的 dns 解析。

所以到这里，大概就能清楚是怎么一回事了。本地环境中，服务在启动的时候，注册本身的服务信息到 DNS 服务中，服务名、IP 地址、端口。后续其他服务要调用本服务，就会进行广播查服务名称对应的服务信息，然后进行访问。

那么针对第二个问题呢，我看了 go-micro 中的代码后有了答案，假设 frontend 服务要调用 currencyservice 服务，那么在 frontend 服务的 proto 文件中就要声明 currencyservice 的内容，个人理解可以把 currencyservice.proto 中的内容 copy 过来。

```go
syntax = "proto3";

package hipstershop;
option go_package = "./proto;demo";

// -----------------Currency service-----------------

service CurrencyService {
    rpc GetSupportedCurrencies(Empty) returns (GetSupportedCurrenciesResponse) {}
    rpc Convert(CurrencyConversionRequest) returns (Money) {}
}

// Represents an amount of money with its currency type.
message Money {
    // The 3-letter currency code defined in ISO 4217.
    string currency_code = 1;

    // The whole units of the amount.
    // For example if `currencyCode` is `"USD"`, then 1 unit is one US dollar.
    int64 units = 2;

    // Number of nano (10^-9) units of the amount.
    // The value must be between -999,999,999 and +999,999,999 inclusive.
    // If `units` is positive, `nanos` must be positive or zero.
    // If `units` is zero, `nanos` can be positive, zero, or negative.
    // If `units` is negative, `nanos` must be negative or zero.
    // For example $-1.75 is represented as `units`=-1 and `nanos`=-750,000,000.
    int32 nanos = 3;
}

message GetSupportedCurrenciesResponse {
    // The 3-letter currency code defined in ISO 4217.
    repeated string currency_codes = 1;
}

message CurrencyConversionRequest {
    Money from = 1;

    // The 3-letter currency code defined in ISO 4217.
    string to_code = 2;
}
```

以上是 frontend 服务的部分 proto 内容，里边列出了 currencyservice 需要用到的请求体和响应体，以及提供的相关功能。

也就意味着，如果后续 currencyservice 提供了新的“函数”，不仅要在本地更新 proto 文件，也要在所有调用方进行文件更新，但是感觉这种方式难免会存在遗漏的情况。