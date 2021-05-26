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

我们在使用SockJS作为传输层时，遇到浏览器对跨域请求报错：

`Failed to load resource: Cannot use wildcard in Access-Control-Allow-Origin when credentials flag is true.`

搜索发现，浏览器要求当XHR请求指定了`withCredentials`等于`true`时，

而服务端返回的`Access-Control-Allow-Orgin`必须是精确匹配。否则不允许页面获取该资源。

这里的精确匹配是指协议、域名、端口全部都匹配（在服务端返回的`Access-Control-Allow-Orgin`中）。

## XMLHttpRequest.withCredentials

这个选项默认为`false`，即跨域访问时，浏览器不会在请求中携带Cookie等安全信息。

这是出于安全性的考虑，这样客户端可以避免Cookie等安全信息意外的泄漏给另一个域。

但是有些服务是依赖这些安全信息的，就需要设置这个选项为`true`，同时还要满足上面的精确匹配要求，外加`Access-Control-Allow-Credentials: true`。

## Access-Control-Allow-Credentials

客户端可以控制安全信息的发送了，那么服务端是否想要呢？于是就有了`Access-Control-Allow-Credentials`头域。

当客户端设置`withCredentials`等于`true`时，如果服务器不返回`Access-Control-Allow-Credentials: true`，请求就失败了。

为了预判服务器是否支持CORS，浏览器会进行preflight：

![](https://upload.wikimedia.org/wikipedia/commons/c/ca/Flowchart_showing_Simple_and_Preflight_XHR.svg)

## 检查浏览器支持跨域


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
