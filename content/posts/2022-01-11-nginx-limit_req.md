---
title: Nginx-limit_req
date: 2022-01-11T08:43:19+00:00
categories:
  - Nginx
  - 编程
tags:
  - Nginx
---

作为一款应用最为广泛的webserver，Nginx内置了限制请求速率的功能，即可以限制用户在给定时间段发出的请求能够到达应用服务器的数量。

该功能常用于安全目的，比如**减缓**首页登录密码的暴力破解，并且通过将速率限制为典型的真实用户值来防止DDOS攻击。通俗来讲，主要用于保护上游应用程序服务器不被太多的用户请求同时淹没。

## 工作原理

Nginx限速通过漏桶算法实现，该算法广泛用于分组交换通信网络中，用于处理带宽受限时的突发性。类比是一个桶，水从顶部倒入，从底部漏出；如果倒水的速度超过漏水的速度，水桶就会溢出。在请求处理方面，水代表来自客户端的请求，桶代表一个队列，其中请求根据先进先出 (FIFO) 调度算法等待处理。漏水代表请求退出缓冲区供服务器处理，溢出代表请求被丢弃。

![漏桶算法示意图](https://objects.wangaiguo.com/wp-content/uploads/2022/01/leaky-bucket-featured-image-300x180-3.jpg)

## 基本配置

配置速率限制主要有两个指令：

* limit_req_zone
* limit_req

以下是一个简单的配置示例：

```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
 
server {
    location /login/ {
        limit_req zone=mylimit;
        
        proxy_pass http://my_upstream;
    }
}
```

语法规则如下：

```nginx
Syntax:  limit_req_zone  key  zone=name:size rate=rate [sync];
Default:  —
Context:  http
```

limit_req_zone 在http模块中定义，参数含义如下：

* **key** ： 一般以上述示例的$binary_remote_addr作为限制特征，表示的是限制同一客户端ip地址
* **zone**: 在这里使用的是 $binary_remote_addr 而不是 $remote_addr。 $binary_remote_addr是$remote_addr的二进制形式，能够有效的减少存储空间。当区的大小为 1M 的时候，大约可以记录 16000个ip信息。
* **rate**: 设置最大请求速率。在示例中，速率不能超过每秒 10 个请求。NGINX 实际上以毫秒为单位跟踪请求，因此此限制对应于每 100 毫秒1 个请求。这意味着如果请求在前一个允许的请求之后不到 100 毫秒到达，则请求将被拒绝。

## 漏桶配置

如果在100毫秒内收到第二个请求，按照上述配置，直接返回`503`错误，这显然是不合适的，通过引入**burst**配置可以解决上述问题，在http模块配置好`limit_req_zone`后，可以按照示例在server上下文中直接引用`zone`， burst 表示漏桶的大小，具体配置如下：

```nginx
Syntax:   limit_req zone=name  [burst=number]  [nodelay | delay=number];
Default:   —
Context:   http,  server,  location
```

接下来，更新server配置：

```nginx
location /login/ {
    limit_req zone=mylimit burst=20;
 
    proxy_pass http://my_upstream;
}
```

这里将burst大小改为20，这意味着如果 21 个请求同时从给定 IP 地址到达，NGINX 会立即将第一个请求转发到上游服务器，并将剩余的 20 个放入队列中。然后它每 100 毫秒转发一个排队的请求，并且当排队的请求数超过 20 时直接返回`503`错误。

## delay配置

burst配置会使得流量很顺畅，但是会使你的服务器看起来很慢，在该示例，队列中的第20个请求，要等待两秒才会被转发，这显然是不合适的，要解决这个问题，可以通过添加参数nodelay进行配置。

使用该`nodelay`参数，NGINX 仍然根据`burst`参数分配队列中的槽并施加配置的速率限制，但不是通过间隔排队请求的转发。相反，当请求“过早”到达时，只要队列中有可用的插槽，NGINX 就会立即转发它。它将该槽标记为“已占用”，并且在经过适当的时间（在我们的示例中为 100 毫秒之后）之前，不会将其释放以供另一个请求使用。

和之前一样，假设 20 个槽的队列是空的，并且有 21 个请求同时从给定的 IP 地址到达。NGINX 立即转发所有 21 个请求并将队列中的 20 个插槽标记为已占用，然后每 100 毫秒释放 1 个插槽。（如果有 25 个请求，NGINX 将立即转发其中的 21 个，将 20 个插槽标记为已占用，并拒绝 4 请求返回`503`错误。）

现在假设在第一组请求被转发后的 101 毫秒，另外 20 个请求同时到达。队列中只有 1 个插槽已被释放，因此 NGINX 转发 1 个请求，拒绝其他 19 个请求。如果在 20 个新请求到达之前已经过去了 501 毫秒，则 5 个插槽是空闲的，因此 NGINX 立即转发 5 个请求并拒绝 15 个。

效果相当于每秒 10 个请求的速率限制。如果您想在不限制请求之间允许的间距的情况下施加速率限制，可以配置nodelay。

## 参考

[NGINX Rate Limiting](https://www.nginx.com/blog/rate-limiting-nginx/)
