---
title: "RESTful API 设计规范"
source: "https://juejin.cn/post/7025222833798119454"
author:
  - "[[AGIPlayer]]"
published: 2021-10-31
created: 2026-05-12
description: "工作中如果要设计API，那么你的接口符合RESTful API规范吗？什么是RESTful？那我们通过本篇文章一起来学习下吧！"
tags:
  - "clippings"
---
> 你必须非常努力，才能看起来毫不费力！
> 
> 微信搜索公众号\[ 漫漫Coding路 \]，一起From Zero To Hero!

## 前言

需求开发中设计了几个API接口，组长说我的设计不规范（手动狗头），那周末必须学习一波，防止下次被嘲讽。

## 简介

在主流公司的程序开发中，为了提高程序开发迭代的速度，基本都是前后端分离架构，而前端既包括网页、App、小程序等等，因此必须要有一个统一的规范用于约束前后端的通信，RESTful API则是目前比较成熟的API设计理论。

要想理解RESTful，就需要先明白REST。REST是 [Roy T. Fielding](https://link.juejin.cn/?target=https%3A%2F%2Froy.gbiv.com%2F "https://roy.gbiv.com/") 在其2000年的博士论文中提出的，是 `RE` presentational `S` tate `T` ransfer 词组的缩写，可以翻译为“表现层状态转移”，其实这个词组前面省略了个主语--“Resource”，加上主语后就是“资源表现层状态转移”。每个词都能看懂，连在一起不知道什么意思了有木有？

![ Roy T. Fielding](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50b38e3479ea4ae09e11fd063576a6cd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

1. Resource（资源）
	所谓资源，就是互联网上的一个实体。URI（Uniform Resource Identifier）的全称就是统一资源标识符，和我们这里的资源一个意思。一个资源可以是一段文字、一张图片、一段音频、一个服务。
2. 表现层（Representation）
	"资源"是一种信息实体，它可以有多种外在表现形式。我们把"资源"具体呈现出来的形式，叫做它的"表现层"（Representation）。比如一篇文章，可以使用XML、JSON、HTML的形式呈现出来。
3. 状态转移（State Transfer）
	访问一个网站，就代表了客户端和服务器的一个互动过程。在这个过程中，势必涉及到数据和状态的变化。互联网通信协议HTTP协议，是一个无状态协议，这意味着所有的状态都保存在服务器端。因此，如果客户端想要操作服务器，必须通过某种手段，让服务器端发生"状态转化"（State Transfer）。而这种转化是建立在表现层之上的，所以就是"表现层状态转化"。

上面我们介绍了REST的基本概念，那么服务REST规范的设计，我们就可以称为RESTful。接下来我们就来看一下RESTful API的设计规范。

## 协议

协议是最基本的设计，表示前后端通信规范，现阶段应该使用 `HTTPs` 协议。

## 域名

`API` 的根入口点应尽可能保持足够简单，这里有两个常见的例子：

- api.example.com/\* （子域名下）
- example.com/api/\* （主域名下）

域名应该考虑拓展性，如果不确定API后续是否会拓展，应该将其放在子域名下，这样可以保持一定的灵活性。

## 路径

路径又称为端点，表示API的具体地址。在路径的设计中，需遵守下列约定：

- 命名必须全部 `小写`
- 资源（resource）的命名必须是 `名词` ，并且必须是 `复数形式`
- 如果要使用连字符，建议使用‘-’而不是‘\_’，‘\_’字符可能会在某些浏览器或屏幕中被部分遮挡或完全隐藏
- 易读

命名必须全部小写和易读都无需解释，可以理解为规定，那么为什么命名必须是名词且需要复数形式呢？这是因为在RESTful中，主语是资源，资源肯定是名词，不能是动词。其次，一个资源往往对应数据库中一张表，表就是实体的集合，因此需要是复数形式。

下面是一些反例：

- [api.example.com/getUser](https://link.juejin.cn/?target=https%3A%2F%2Fapi.example.com%2FgetUser "https://api.example.com/getUser")
- [api.example.com/addUser](https://link.juejin.cn/?target=https%3A%2F%2Fapi.example.com%2FaddUser "https://api.example.com/addUser")

下面是一些正例：

- [api.example.com/zoos](https://link.juejin.cn/?target=https%3A%2F%2Fapi.example.com%2Fzoos "https://api.example.com/zoos")
- [api.example.com/zoos/animal…](https://link.juejin.cn/?target=https%3A%2F%2Fapi.example.com%2Fzoos%2Fanimals "https://api.example.com/zoos/animals")

## HTTP动词

对于如何操作资源，有相应的HTTP动词对应，常见的动词有如下五个（括号里表示SQL对应的命令）：

- GET（SELECT）：从服务器取出资源（一项或多项）
- POST（CREATE）：在服务器新建一个资源
- PUT（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源）
- PATCH（UPDATE）：在服务器更新资源（客户端提供改变的属性）
- DELETE（DELETE）：从服务器删除资源

示例：

| HTTP动词 | 路径 | 表述 |
| --- | --- | --- |
| GET | /zoos | 获取所有动物园信息 |
| POST | /zoos | 新建一个动物园 |
| GET | /zoos/ID | 获取指定动物园的信息 |
| PUT | /zoos/ID | 更新指定动物园的信息（前端提供该动物园的全部信息） |
| PATCH | /zoos/ID | 更新某个指定动物园的信息（提供该动物园改动部分的信息） |
| DELETE | /zoos/ID | 删除某个动物园 |
| GET | /zoos/ID/animals | 获取某个动物园里面的所有动物信息 |

## 过滤

如果数据量很大，服务器不可能将全部数据都返回给前端，因此前端需要提供一些参数进行过滤，用于分页展示或者排序等，下面是一些常见的参数：

- ?limit=10：指定返回记录的数量
- ?offset=10：指定返回记录的开始位置。
- ?page=2&per\_page=100：指定第几页，以及每页的记录数。
- ?sortby=name&order=asc：指定返回结果按照哪个属性排序，以及排序顺序。
- ?animal\_type\_id=1：指定筛选条件

## HATEOAS

HATEOAS 是 *Hypermedia As The Engine Of Application State* 的缩写，从字面上理解是 *“超媒体即应用状态引擎”* 。其原则就是客户端与服务器的交互完全由API动态提供，客户端无需事先了解如何与服务器交互，即返回结果中提供链接，连向其他API方法，使得用户不查文档，也知道下一步应该做什么。

例如，若要处理订单与客户之间的关系，可以在订单的表示形式中包含链接，用于指定下单客户可以执行的操作（查看客户信息、查看订单信息、删除订单等操作）。rel表示这个API与当前网址的关系，href表示API的路径，title表示API的标题，type表示返回类型，action表示支持的操作类型。

```json
体验AI代码助手 代码解读复制代码{
  "orderID":3,
  "productID":2,
  "quantity":4,
  "orderValue":16.60,
  "links":[
     {
      "rel":"customer",
      "href":"https://adventure-works.com/customers/3",
      "action":"GET",
      "title":"get customer info",
      "types":["text/xml","application/json"]
    },
    {
      "rel":"self",
      "href":"https://adventure-works.com/orders/3",
      "action":"GET",
      "title":"get order info",
      "types":["text/xml","application/json"]
    }]
}
```

Github的API就是这种设计，访问 [api.github.com](https://link.juejin.cn/?target=https%3A%2F%2Fapi.github.com%2F "https://api.github.com/") 会得到一个所有可用API的网址列表。

```json
体验AI代码助手 代码解读复制代码{
  "current_user_url": "https://api.github.com/user",
  "emojis_url": "https://api.github.com/emojis",
  "events_url": "https://api.github.com/events",
  ......
}
```

## 版本控制

API一直保持静态的可能性很小，随着业务需求变化，可能会添加新的资源，底层的数据结构可能也会有更改。在更新API提供新功能的同时，需要考虑对已使用该API用户的影响，因此需要保持向前兼容，这就引出了版本控制。主要的版本控制方法有如下几种：

1. URI版本管理

每次修改 Web API 或更改资源的架构时，向每个资源的 URI 添加版本号。 以前存在的 URI 应像以前一样继续运行，并返回符合原始架构的资源。

```
api.example.com/v1/*
api.example.com/v2/*
```

该方法的版本控制机制非常简单，但是随着 API 多次迭代，服务器需要支持多个版本的路由，增大了维护的成本。 此方案也增加了 HATEOAS 实现的复杂性，因为所有链接都需要在其 URI 中包括版本号。

2. 查询字符串版本控制

不是提供多个 URI，而是通过在追加查询字符串的方式来指定版本，例如 `https://adventure-works.com/customers/3?version=2` 。 如果 version 参数被较旧的客户端应用程序省略，则应默认为有意义的值（例如 1）。

此方法具有语义优势（即，同一资源始终从同一 URI 进行检索），但它依赖于代码处理请求以分析查询字符串并发送回相应的 HTTP 响应。 该方法也与 URI 版本控制机制一样，增加了实现 HATEOAS 的复杂性。

3. 自定义请求标头进行版本控制

在请求的header中自定义版本控制选项。

```
GET https://adventure-works.com/customers/3 HTTP/1.1
Custom-Header: api-version=1
```
4. Accept标头进行版本控制

当客户端应用程序向 Web 服务器发送 HTTP GET 请求时，可以 Accept 标头规定它可以处理的内容的格式。 通常， *Accept* 标头的用途是客户端指定响应的正文应是 XML、JSON 或者其他可处理的的格式。 但是，我们也可以指定该标头为使客户端需要的资源版本。

```
体验AI代码助手 代码解读复制代码GET https://adventure-works.com/customers/3 HTTP/1.1
Accept: application/vnd.adventure-works.v1+json
```

上例将 *Accept* 标头指定为 *application/vnd.adventure-works.v1+json* 。 *vnd.adventure-works.v1* 元素向 Web 服务器指示它应返回资源的版本 v1，而 *json* 元素则指定响应正文的格式应为 JSON。

此方法可以说是最纯粹的版本控制机制并自然地适用于 HATEOAS，后者可以在资源链接中包含相关数据的 MIME 类型。

在现实世界中，API永远不会完全稳定。因此，如何管理这一变化非常重要。对于大多数API而言，商定好部分版本的控制策略，然后对API详细记录和逐步弃用是可接受的做法。

## 服务端响应

API响应，需要遵守HTTP设计规范，选择合适的状态码返回。你可能见过有的接口始终返回状态码200，然后通过返回体中的code字段进行区分请求是否成功，这种是不符合规范的，相当于状态码没有了任何作用，下面就是一个反例。

```
体验AI代码助手 代码解读复制代码HTTP/1.1 200 ok
Content-Type: application/json
Server: example.com

{
    "code": -1,
    "msg": "该活动不存在",
}
```

其次，在出现错误时，需要返回错误信息，常见的返回方式就是放在返回体中。

```
HTTP/1.1 401 Unauthorized
Server: nginx/1.11.9
Content-Type: application/json
Transfer-Encoding: chunked
Cache-Control: no-cache, private
Date: Sun, 24 Jun 2018 10:02:59 GMT
Connection: keep-alive

{"error_code":40100,"message":"Unauthorized"}
```

## 状态码

HTTP状态码由三个十进制数字组成，第一个十进制数字定义了状态码的类型，HTTP状态码共分为5种类型：

| 分类 | 描述 |
| --- | --- |
| 1xx | 信息，服务器收到请求，需要请求者继续执行操作 |
| 2xx | 成功，操作被成功接收并处理 |
| 3xx | 重定向，需要进一步的操作以完成请求 |
| 4xx | 客户端错误，请求包含语法错误或无法完成请求 |
| 5xx | 服务器错误，服务器在处理请求的过程中发生了错误 |

API不需要1xx类型的状态码，因此我们主要看下其他几个类型常见的状态码：

1. 2xx状态码

| 状态码 | 英文名称 | 描述 |
| --- | --- | --- |
| 200 | OK | 请求成功，一般用于GET和POST请求 |
| 201 | Created | 请求成功并创建了新的资源，用于POST、PUT、PATCH请求。例如新增用户、修改用户信息等，同时在返回体中，我们既可以返回创建后实体的所有信息数据，也可以不返回相关信息。 |
| 202 | Accepted | 已接受请求，但未处理完成，会在未来再处理，通常用于异步操作 |
| 204 | No Content | 该状态码表示响应实体不包含任何数据，使用DELETE进行删除操作时，需返回该状态码 |

2. 3xx状态码

API 用不到301状态码（永久重定向）和302状态码（暂时重定向，307也是这个含义），因为它们可以由应用级别返回，浏览器会直接跳转，API 级别可以不考虑这两种情况。

API 用到的3xx状态码，主要是303 See Other，表示参考另一个 URL。它与302和307的含义一样，也是"暂时重定向"，区别在于302和307用于GET请求，而303用于POST、PUT和DELETE请求。收到303以后，浏览器不会自动跳转，而会让用户自己决定下一步怎么办。

下面是一个例子。

```html
体验AI代码助手 代码解读复制代码HTTP/1.1 303 See Other
Location: /api/orders/12345
```
3. 4xx状态码

| 状态码 | 英文名称 | 描述 |
| --- | --- | --- |
| 400 | Bad Request | 客户端请求的语法错误，服务器无法理解 |
| 401 | Unauthorized | 表示用户没有权限（令牌、用户名、密码错误） |
| 403 | Forbidden | 没有权限访问该请求，服务器收到请求但拒绝提供服务 |
| 404 | Not Found | 服务器无法根据客户端的请求找到资源（如路径不存在） |
| 405 | Method Not Allowed | 客户端请求的方法服务端不支持，例如使用 POST 方法请求只支持 GET 方法的接口 |
| 406 | Not Acceptable | 用户GET请求的格式不可得（比如用户请求 JSON 格式，但是只有 XML 格式） |
| 408 | Request Time-out | 客户端请求超时 |
| 410 | Gone | 客户端GET请求的资源已经不存在。410 不同于 404，如果资源以前有现在被永久删除了可使用410 代码，网站设计人员可通过 301 代码指定资源的新位置 |
| 415 | Unsupported Media Type | 通常表示服务器不支持客户端请求首部 Content-Type 指定的数据格式。如在只接受 JSON 格式的 API 中放入 XML 类型的数据并向服务器发送 |
| 429 | Too Many Requests | 客户端的请求次数超过限额 |

4. 5xx状态码

5xx状态码表示服务端错误。一般来说，API 不会向用户透露服务器的详细信息，所以只要两个状态码就够了。

| 状态码 | 英文名称 | 描述 |
| --- | --- | --- |
| 500 | Internal Server Error | 客户端请求有效，服务器处理时发生了意外 |
| 503 | Service Unavailable | 服务器无法处理请求，一般用于网站维护状态 |

## 其他

- API的身份认证应该使用 [OAuth 2.0](https://link.juejin.cn/?target=https%3A%2F%2Fwww.ruanyifeng.com%2Fblog%2F2014%2F05%2Foauth_2_0.html "https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html") 框架
- 服务器返回的数据格式，应该尽量使用JSON，避免使用XML

## 参考

[www.ruanyifeng.com/blog/2011/0…](https://link.juejin.cn/?target=https%3A%2F%2Fwww.ruanyifeng.com%2Fblog%2F2011%2F09%2Frestful.html "https://www.ruanyifeng.com/blog/2011/09/restful.html")

[www.ruanyifeng.com/blog/2014/0…](https://link.juejin.cn/?target=https%3A%2F%2Fwww.ruanyifeng.com%2Fblog%2F2014%2F05%2Frestful_api.html "https://www.ruanyifeng.com/blog/2014/05/restful_api.html")

[zhuanlan.zhihu.com/p/68103094](https://link.juejin.cn/?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F68103094 "https://zhuanlan.zhihu.com/p/68103094")

[segmentfault.com/a/119000001…](https://link.juejin.cn/?target=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000015384373 "https://segmentfault.com/a/1190000015384373")

[docs.microsoft.com/zh-cn/azure…](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.microsoft.com%2Fzh-cn%2Fazure%2Farchitecture%2Fbest-practices%2Fapi-design "https://docs.microsoft.com/zh-cn/azure/architecture/best-practices/api-design")

## 总结

本篇文章我们一起来学习了RESTful API的基本概念和设计规范，基于该规范，在工作中可以根据团队来做些适配。

## 更多

个人博客: [lifelmy.github.io/](https://link.juejin.cn/?target=https%3A%2F%2Flifelmy.github.io%2F "https://lifelmy.github.io/")

微信公众号：漫漫Coding路