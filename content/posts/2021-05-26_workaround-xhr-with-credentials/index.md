---
title: "XMLHttpRequest CORS 考古"
date: 2021-05-26
categories:
    - 笔记
tags:
    - SockJS
    - XHR
draft: false
---

今天在用SockJS时，遇到浏览器对跨域请求报错：

`Failed to load resource: Cannot use wildcard in Access-Control-Allow-Origin when credentials flag is true.`

搜索发现这是一个浏览器阻止不安全的跨域访问的问题，我们来考一下古：

1993年，互联网正在高速发展，`<img>`这个标签被加入进来。

从那开始，网页变得漂亮多了。随着越来越多的标签也都加入进来，比如`<script>`。

那时候，网站之间可以随便访问，很容易发生跨站攻击，于是就有了CORS。

考古完成，下面开始干活。我们看看今天的这个问题:

## XMLHttpRequest.withCredentials

一番搜索发现，选项默认为`false`，浏览器跨域访问时，如果要携带Cookie等安全信息，就要设置这个选项为`true`。

这是出于安全性的考虑，这样客户端可以避免Cookie等安全信息意外的泄漏给另一个域。

如果服务依赖Cookie等安全信息，就要满足：

- `XMLHttpRequest.withCredentials`为`true`
- `Access-Control-Allow-Origin`精确匹配（这里的精确匹配是指协议、域名、端口全部都匹配）
- `Access-Control-Allow-Credentials`为`true`

`Access-Control-Allow-Origin`相信大家都很熟悉了，我们还是看看`Access-Control-Allow-Credentials`是干什么的：

## Access-Control-Allow-Credentials

客户端可以控制安全信息的发送了，那么服务端是否想要呢？于是就有了`Access-Control-Allow-Credentials`头域。

当客户端设置`withCredentials`等于`true`时，如果服务器不返回`Access-Control-Allow-Credentials: true`，请求就失败了。

为了预判服务器是否支持CORS，浏览器会进行preflight：

![](https://upload.wikimedia.org/wikipedia/commons/c/ca/Flowchart_showing_Simple_and_Preflight_XHR.svg)

那么，今天的问题怎么处理呢？

## Case by case

由于我是使用SockJS作为传输层，安全认证所需的信息协议层，所以我不需要客户端发送Cookie等安全信息。

但是SockJS在跨域时，只要发现浏览器支持`withCredentials`就会设置为`true`。

``` javascript
var cors = false;
try {
  cors = 'withCredentials' in new XHR();
} catch (ignored) {
}
AbstractXHRObject.supportsCORS = cors;
```

``` javascript
if ((!opts || !opts.noCredentials) && AbstractXHRObject.supportsCORS) {
    this.xhr.withCredentials = true;
}
```

所以，我试着workaround了一下：

## Workaround

加载SockJS前，shim XMLHttpRequest.send()，把withCredentials改为`false`:

``` javascript
(function(xhr) {
    window.shimmedXHR = XMLHttpRequest.prototype;
    send = xhr.send;
    xhr.send = function(data) {
        this.withCredentials = false;
        return send.apply(this, arguments);
    };
})(XMLHttpRequest.prototype);
```

加载SockJS后，再改回来：

``` javascript
(function(xhr) {
    xhr = window.shimmedXHR || xhr;
})(XMLHttpRequest.prototype);
```

测试一下，可以跨域进行SockJS连接了。

