# GET 请求可以发送 body 吗

我们在发送 request 消息的时候，往往会使用 POST 请求的 body 携带数据，而不会在 GET 请求的 body 里携带数据。因为我们认为 GET 请求应该是幂等的，而且请求携带的信息应该通过 URI 传递。那么是否可以在 GET 请求里发送 body 呢。

实际上对于 HTTP 请求来说，请求体结构定义和实际使用的时候用的 method——也就是它的语义——是无关的。因此 GET 请求也可以像 POST 请求一样发送 body，GET 请求和 POST 请求实际上只有语义上的区别，HTTP 协议的实现方们，也就是客户端和服务端之间约定了 GET 和 POST 的不同作用。

但是关于是否该在 GET 里使用 body，[RFC7231](https://tools.ietf.org/pdf/rfc7231.pdf) 里关于 GET 部分是这样说的:

> A payload within a GET request message has no defined semantics; sending a payload body on a GET request might cause some existing implementations to reject the request.

意思就是说在 GET 请求里使用 body 是不符合语义的，同时在 GET 请求里发送 body 也可能会导致目前现有的 HTTP 实现方拒绝请求。

从官方的文档可以看出是不鼓励我们在 GET 请求里添加 body 的。
事实上，目前大部分的 HTTP 协议实现方也禁止通过 GET 传 body。

例如 `XMLHttpRequest` 会将 GET 请求和 HEAD 请求里的 body 忽略或者设为 null

而 `fetch` 会在执行发送请求时候报错。

目前我了解到的是只有通过 `curl` 可以在 GET 请求里写入 body。

```bash
curl -v -X GET http://localhost:8080/?id=100 -d "Get body"
```

## 还有没有其他办法通过 GET 传递对象呢

其实也是有的。实际上 body 字段传递的内容也只是文本而已，通过 Content-type 指定了它的类型，比如说 `application/json`，服务端收到后就会将其解析为 JSON，这是一种 HTTP 协议实现方的约定。那么如果我们有一个可以被 JSON 化的对象，我们就可以将其 stringify 然后放在 GET 请求的 URI 的 param 里去传递，人为去约定服务端收到后解析为 JSON 结构。
