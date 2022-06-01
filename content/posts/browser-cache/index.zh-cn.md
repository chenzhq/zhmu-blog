---
weight: 1
title: "浏览器缓存"
date: 2022-04-08T03:40:32+08:00
lastmod: 2022-04-08T03:40:32+08:00
draft: false
author: "chenzheqi"
authorLink: "https:/chenzhq.com"
description: "梳理浏览器缓存"
resources:
- name: "featured-image"
  src: "featured-image.jpg"

tags: ["前端", "浏览器", "缓存"]
categories: ["技术"]

lightgallery: true

math:
  enable: true
toc:
  auto: true
---
<!-- markdownlint-disable no-duplicate-heading no-inline-html -->

## 什么是缓存

在计算机世界中，只要有数据读取的地方，往往就会有缓存。最初的缓存，指的是CPU中一块独立的数据存储区域，用来临时存放从内存或磁盘中读取到的数据，这样CPU就可以直接从这块区域里读取数据，避免了内存寻址，甚至磁盘扫描等高成本动作。

后来，缓存进一步扩充，不止存在于CPU和内存之间，在内存和磁盘，磁盘和网络之间也可以通过设计合理的缓存策略，减少数据流通的成本。因此可以说，**凡是位于速度相差较大的两种硬件之间，用于协调两者数据传输速度差异的结构，均可称之为缓存[^cache-wiki]。**

[^cache-wiki]: <https://zh.wikipedia.org/wiki/%E7%BC%93%E5%AD%98>

浏览器主要在磁盘和网络之间，介入多层缓存，通过合理的策略，尽可能地复用本地数据，减少网络请求和负载，进而带来页面性能的提升。

## 浏览器有多少缓存

最常用的浏览器缓存是HTTP缓存，即客户端和服务端基于HTTP协议，实现的一套缓存策略。除此之外，还有三类缓存：内存缓存(Memory Cache)，Sevice Worker缓存和Push缓存。这四种缓存分别在一次请求的不同切入点上起作用：

{{< figure src="./cache-path.png" title="缓存路径">}}

一个请求由页面发起，沿着左侧的箭头，逐级向下，判断需要的资源在当前层是否存在，存在则直接返回，不继续流转到下一层，不存在则继续向下匹配。

如果所有的缓存都没有匹配上，浏览器将向服务器发起一个新的HTTP请求，协商获取最新的资源，响应会沿着右侧的箭头，逐级向上，回传给页面，同时按照一定的规则将返回的数据复制一份到缓存中。

### Memory Cache

内存缓存是浏览器自身实现的一套逻辑，没有相应的规范描述这层缓存应该如何工作，因此不同浏览器可能有不一样的缓存方式，对开发者和使用者来说，是一个黑盒。以我目前了解的资料[^a-tale-of-four-caches]来看，Chrome浏览器在这一层缓存中，主要用于这么几种场景：

[^a-tale-of-four-caches]: <https://calendar.perfplanet.com/2016/a-tale-of-four-caches/>

  1. `<link rel=preload>`引入的资源，会缓存在这里，当后续浏览器真正使用这个资源时，可以直接从内存缓存中读取；
  2. DOM元素或CSS规则中引用到的外部资源，会缓存在这里，比如`<img>`中`src`指向的图片，第一次解析到会发送请求获取url对应的内容，然后缓存在内存中，后面如果再次引入一个相同的图片，浏览器就无需发送请求，直接从内存中读取

浏览器在匹配内存缓存中的资源时，除了资源的URL要相同外，资源类型(Content-Type)，CROS，还有其他的特性也要保持一致。另外，内存缓存基本不受 HTTP 缓存中的请求响应头的影响，比如`Cache-Control`中的`max-age`, `no-cache`。唯一的例外就是`no-store`，如果`Cache-Control`的值是`no-store`，那么在一些场景下，这个响应对应的资源不会缓存在内存中。

{{< admonition type=tip title="内存缓存" open=true >}}
这里的 Memory Cache 指的是浏览器内部实现的一种缓存能力，并不是 HTTP 缓存的一部分，因此和浏览器开发者工具 Network 里看到的 from memory 缓存并不一样。
{{< /admonition >}}

### Service Worker Cache

Servie Worker独立运行在浏览器进程外，本质上充当 Web 应用程序、浏览器与网络（可用时）之间的代理服务器，开发者可以利用其对请求拦截的能力，主动采取合适的缓存策略。

{{< figure src="https://s3.bmp.ovh/imgs/2022/04/08/e5348c16752e5b6c.png" title="Service Worker 基本工作原理">}}

相比于其它缓存，Servie Worker 缓存最大的优势就是，可以让我们**自由地控制缓存哪些文件、如何匹配缓存、如何读取缓存，并且缓存是持续性的[^service-worker]**。由于 Service Worker 涉及到请求拦截，因此必须工作在 **HTTPS** 环境下。

[^service-worker]: <https://web.dev/service-worker-caching-and-http-caching/>

{{< admonition type=tip title="TIP" open=true >}}
Service Worker 在 localhost 或者 127.0.0.1 这种本地环境下的时候，可以不需要 HTTPS 环境。
{{< /admonition >}}

### HTTP Cache

HTTP 缓存是利用 HTTP 协议中定义的一系列消息头，实现的一套缓存策略。常用的响应头和请求头包括

* `Last-Modified`, 对应请求头 `If-Modified-Since`
* `Etag`, 对应请求头 `If-None-Match`
* `Cache-Control`, 对应请求头同样是 `Cache-Control`

这一层是 Web 开发最关键，也最常使用的缓存，本文后面会详细介绍这些请求和响应头是如何协同工作的。

### Push Cache

Push Cache 是 HTTP/2 中实现的缓存。基本原理是，当浏览器向服务器发送一个请求时，服务器可以在返回这个资源的同时，额外带上其它可能需要的资源，浏览器接收到这些响应后缓存起来，未来需要时可以直接从缓存中读取。它只在一次Session中存在，Session结束或一段时间后（Chrome中是5分钟）就会失效，而且并不严格遵守 HTTP 的缓存指令(可以缓存`no-store`或`no-cache`的资源)。

HTTP Push 的设计本意是通过减少请求次数来提高网页性能。然而这项技术从诞生开始就一直没有得到很好的浏览器支持，根据 Chrome 团队的调研，这种提前 push 文件给客户端的方式，并没有带来明显的性能提高。因此 Chrome 在2020年已经移除了对 Service Push 的支持[^chrome-remove-push]。而其它浏览器也都没有特别积极地去实现这项特性，因此 HTTP2 Service Push 已经宣告废弃[^http-push-dead]。

{{< admonition type=info title="信息" open=true >}}
Nginx 仍然支持 HTTP/2 Push，可以通过`http2_push`指令推送资源给客户端
{{< /admonition >}}

[^chrome-remove-push]: <https://groups.google.com/a/chromium.org/g/blink-dev/c/K3rYLvmQUBY/m/vOWBKZGoAQAJ?pli=1>
[^http-push-dead]: <https://evertpot.com/http-2-push-is-dead/>

## HTTP缓存是如何工作的

以上4类缓存中，Memory Cache 是浏览器自身实现的一套逻辑，我们无法介入；Service Worker 一般应用于PWA中，普通 Web 应用较少使用；而 Push Cache 已经名存实亡。只有 HTTP 缓存从 HTTP 1.0 时代发展到现在，经久不衰，也的确解决了 Web 应用中的缓存难题。下面将重点介绍 HTTP 缓存是如何工作的。

再次强调，请求的资源只有在 Memory Cache 和 Service Worker Cache 中都不存在时，才会走到 HTTP 缓存这一步，因此下面的内容都是在此前提下展开的。

### 缓存的简单设计

HTTP 协议使用请求应答模型实现通信，一次请求必然对应一个响应。资源(例如图像，脚本等)存在于服务器中，客户端通过一次 HTTP 请求获取资源，服务端收到请求后，将最新的资源返回给客户端。这就是一次简单完整的请求应答过程。

{{< figure src="./http-around.jpeg" title="HTTP缓存">}}

如果现在要在这个流程中添加缓存的能力，其实也就是”服务端要告诉客户端，我这次给你的数据，你可以在自行存储**一段时间**，过了这段时间后，再找我要最新的数据“。

那么这个语义中最核心的问题就是，如何定义一段时间，使客户端和服务端都能统一理解，并不产生偏差。最直观的方式，是服务端在返回数据的同时，带上一个失效时间，客户端在这个失效时间之前不再发送请求，直接从本地缓存中读取。

当然这个方案有一个明显的缺点：过期时间是服务器时间，客户端判断是否失效用的是本地时间，如果两端的时间有偏差，缓存表现就会和预期不符。【TODO，画个图】

解决这个问题的方式也很简单，使用相对的时间长度替换绝对的时间点即可。也就是，服务端在返回数据的同时，带上这次响应可缓存的时长，比如300秒，客户端就基于它本地的时间缓存300秒即可。

当客户端缓存失效后，理应再次请求服务端，获取最新的资源，服务器收到请求后，也应当返回最新的资源。但是，假如这个资源并没有变化，再次传输这堆重复的数据，无论是对两端机器的负载，还是对网络带宽的占用，都是一种浪费。

因此我们可以再设计一种兜底方案，如果服务端确定被请求的资源并没有变化，可以告诉客户端这个特殊情况，不返回数据，客户端收到这个空 body 的响应后，直接从本地缓存中获取资源即可。

解决前后的交互图如下。

{{< mermaid >}}
sequenceDiagram
    participant 客户端
    participant 服务端
    note right of 服务端: 使用绝对时间
    客户端->>服务端: 请给我图片1
    服务端->>客户端: 给你图片1 :framed_picture: <br>在2022年12月31号23:59:59秒以前不用再找我
    note right of 服务端: 使用相对时间
    客户端->>服务端: 请给我一张图片2
    服务端->>客户端: 给你图片2 :framed_picture: <br>这次响应创建于2022年5月1号07:32:44<br>请在本地保留3600秒
{{< /mermaid >}}

在 HTTP 协议中，针对上述第二种方案，抽象出了一个新鲜度(Freshness)的概念。

#### 新鲜度

一个HTTP响应从创建到失效的这段时间，称为这个响应的一次**保鲜生命周期**(Freshness Lifetime)，简称保鲜期。准确来说，是这次响应所携带数据的保鲜期。记住，**保鲜期是一段时长。** 例如300秒。保鲜期由服务端定义，并传给客户端。

> A fresh response is one whose age has not yet exceeded its freshness lifetime.  Conversely, a stale response is one where it has.[^rfc7234-fresh]

[^rfc7234-fresh]: <https://datatracker.ietf.org/doc/html/rfc7234#section-4.2>

客户端接收到响应数据后，同时会存储每个响应资源对应的保鲜期。当再次需要请求该资源时，按照约定的算法，计算这个响应资源的年龄，判断是否小于保鲜期，如果是，则直接使用缓存的内容。不是，则表示该资源**已过期**(Stale)，需要从缓存中移除。

判断缓存失效后，需要重新发送一个新请求，这次请求响应会重新计算一次保鲜期。

这里又引入了概念：响应年龄，或者称之为寿命。

#### 响应年龄/寿命

每个HTTP响应都有年龄(Age)。根据RFC[^calc-age-rfc]的规定，计算响应年龄有两种完全独立的方式：

[^calc-age-rfc]: <https://datatracker.ietf.org/doc/html/rfc7234#section-4.2.3>

如果本地和原始服务器(Origin Server)的时间是一致的，响应年龄可以这样计算，称为表观年龄(apparent_age):

$$ 表观年龄 = 响应接收时间 - 响应创建时间 $$

{{< figure width="80%" src="./apparent_age.jpg" title="表观年龄计算示意图">}}

如果这次请求响应经过的所有缓存都支持 HTTP 1.1，年龄的计算就不依赖服务端时间，而是和请求创建时间相关，这样计算出来的年龄称为修正年龄(corrected_age_value):

$$ 修正年龄 = 响应接收时间 - 请求发起时间 + 可能的响应头age $$

{{< figure width="70%" src="./corrected_age.jpg" title="修正年龄计算示意图">}}

{{< admonition type=info title="响应头中的Age" open=true >}}
当一个HTTP请求经过了中间代理(比如CDN)时，在响应头中会有一个`Age`字段，表示这个资源在代理中被缓存了多久(秒)。
{{< /admonition >}}

根据上面两种方式计算出的年龄，再结合网络环境，可以进一步确定响应的初始年龄：

如果这次请求响应经过的所有代理和服务器，都支持HTTP/1.1协议，则

$$ 响应初始年龄 = 修正年龄 $$

否则，

$$ 响应初始年龄 = max(表观年龄, 修正年龄) $$

其实经过上面的分析，基本可以确定，修正年龄一定大于表观年龄，因此初始响应年龄就是修正年龄的值，同时也代表着，**响应年龄的计算起始点是请求发起的时间**。

初始年龄计算完成后，随着时间的流逝，响应的年龄也在不断的增加，即响应当前年龄：

$$ 当前年龄 = 响应初始年龄 + \overbrace{(现在的时间 - 响应接收时间)}^{\text{响应存储时间}} $$

通过这个公式计算而来的当前年龄，就是在判断缓存资源是否过期时，用来和新鲜度(Freshness)进行匹配的数值。

{{< figure src="./Freshness-Lifetime.png" title="保鲜期">}}

{{< admonition type=info title="启发式过期时间" open=true >}}
如果响应头中没有`Expires`或`Cache-Control`等明确的过期时间，浏览器会使用启发算法计算一个过期时间：(Date - Last-modifile) * 10%。
{{< /admonition >}}

以上就是一个简陋却完整的 HTTP 缓存核心方案设计，以及相关RPC文档中的部分概念介绍。现在回到实际的 HTTP 协议中，我们实际使用的缓存分为两类：不发送请求的**强缓存**，和发送请求的**协商缓存**。

### 强缓存

强缓存生效时，浏览器不需要向服务器发起 HTTP 请求，而是直接使用本地的缓存文件。可以通过设置`Expires`或`Cache-Control`来实现强缓存策略。

#### Expires

响应头中的`Expires`是一个 UTC 绝对时间，用来指定这个响应的过期时间。在 HTTP/1.0 的协议中，这个时间被客户端拿来直接对比，判断资源是否过期。因此会有时间差引起的问题。也就是上面介绍的根据一个绝对时间点来判断资源是否过期。

在 HTTP/1.1 中，`Expires`会和`Date`一起用来计算保鲜期，也就是使用相对的时长来判断是否过期，判断资源是否过期的逻辑，和 HTTP/1.0 中已经不一样了，也就解决了时间差带来的问题。

但这个响应头更大的问题在于设计过于简单，无法满足特殊的缓存需求，比如对共享缓存的控制等。因此现在，除非确信网络环境中有不支持 HTTP/1.1 的设备，否则不建议使用`Expires`，而是用`Cache-Control`代替。

#### Cache-Control

`Cache-Control` 是 HTTP/1.1 引入用于替代`Expires`的消息头字段，通过组合多种指令来实现更加灵活的缓存机制。缓存指令是单向的，这意味着在请求中设置的指令，不一定被包含在响应中。[^Cache-Control-MDN]

[^Cache-Control-MDN]: <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control>

`Cache-Control`包括以下这些指令：

|      消息头      | 请求的含义                       | 响应的含义                                                               |
| :--------------: | :------------------------------- | :----------------------------------------------------------------------- |
|   **max-age**    | 不接受Age大于max-age的资源       | 资源的过期/缓存时间                                                      |
|    max-stale     | 可接受的最大过期时间             |                                                                          |
|    min-fresh     | 可接受的最小新鲜度               |                                                                          |
|   **no-cache**   | 与服务器协商后，决定是否使用缓存 | 与服务器协商前，不允许使用本次响应的缓存                                 |
|   **no-store**   | 不要缓存任何请求和响应的数据     | 不要缓存任何请求和响应的数据                                             |
|   no-transform   | 不允许中介代理修改资源数据       | 不允许中介代理修改资源数据                                               |
|  only-if-cached  | 只接受已缓存的响应               |                                                                          |
| must-revalidate  |                                  | 一旦资源过期，在成功向原始服务器验证之前，缓存不能用该资源响应后续请求。 |
|    **public**    |                                  | 响应可被代理和客户端缓存                                                 |
|   **private**    |                                  | 响应只能被客户端缓存                                                     |
| proxy-revalidate |                                  | 共享缓存的must-revalidate                                                |
|     s-maxage     |                                  | 共享缓存的max-age                                                        |
|  **immutable**   |                                  | 响应正文不会随时间而改变，不要发送协商缓存头                             |

这些指令大多数可以一起使用，比如常用的`cache-control: public, max-age=300`表示响应可以被所有代理缓存，缓存时长300秒。

{{< mermaid >}}
sequenceDiagram
    participant 客户端
    participant 服务端
    rect rgb(140, 173, 205)
    客户端->>服务端: GET /img1.png
    服务端->>客户端: :framed_picture: <br>Expires: Wed, 31 Dec 2022 23:59:59 GMT
    end
    rect rgb(200, 150, 255)
    客户端->>服务端: GET /img2.png
    服务端->>客户端: :framed_picture: <br>Date: Wed, 31 Dec 2022 23:59:59 GMT<br>`Cache-Control`: max-age=3600
    end
{{< /mermaid >}}

#### 内存OR磁盘

强缓存用于存放资源的位置有两处，磁盘(from disk)和内存(from memory)。缓存在内存上的数据会存有一份副本存储在磁盘，页面销毁后，内存缓存就会失效，比如关闭了浏览器tab，但磁盘缓存还是可以使用。

{{< figure src="https://s3.bmp.ovh/imgs/2022/04/08/6c8ca6af6225d2aa.png" title="From Memory Cache">}}

和一般预想的一样，缓存从内存种读取比从磁盘读取要快，在 Chrome Network 中可以看到，内存缓存的 Time 是 0ms，所以**我猜测**，内存缓存相当于在浏览器在这个页面的 render 进程中，用一个本地变量将这些数据缓存了一份。

{{< figure src="https://s3.bmp.ovh/imgs/2022/04/08/73d5bc270a00e3bf.png" title="Chrome Network" >}}

至于哪些资源缓存在内存中，哪些在磁盘里，不是固定的，可能这一次加载缓存在磁盘，下次刷新就移动到内存了（有人说会根据当前设备的内存使用情况进行判断）。

强缓存可以有效地避免浏览器向服务器发起过多的请求，从而提高页面性能，同时也减少服务器负载。如果强缓存失效，浏览器不得不重新发起请求，与服务器共同商讨，决定后续的缓存策略，这就是**协商缓存**。

### 协商缓存

协商缓存是浏览器通过和服务器的一次 HTTP 通信，决定缓存是否过期。如果判定没有过期，则返回 `304(Not Modifiled)`，没有消息体，表示可以继续使用缓存；如果过期了，则返回200，同时带上最新的数据，浏览器用这个新响应结果更新缓存。

那么现在问题就变成了，如何判定缓存是否过期。其实也很简单，无非就是浏览器将上一次的响应发给服务器，服务器收到数据后和当前资源进行匹配，匹配上了表示缓存没过期，反之表示过期。

但实际上，浏览器肯定不会将完整的资源返给服务端，而是用一个**标记**来标识这个资源，服务端直接匹配这个标记即可。

这个标记可以是文件的**更新时间**，也可以是文件的**内容特征值**。

### Last-Modified / If-Modified-Since

服务器返回一个资源时，通过响应头`Last-Modified`将这个资源的(最近)更新时间一起发送给浏览器，浏览器接收到这个信息后，会与该资源的URI绑定在一起。当浏览器需要再次请求相同资源时，通过`If-Modified-Since`将这个时间回传给服务器，服务器通过比对这个时间和资源当前的最近更新时间来判断缓存是否过期：

1. 如果满足条件(资源的上次更新时间晚于`If-Modified-Since`)，表示自从上次请求后，资源有更新，返回 HTTP Code `200`，和更新后的资源，同时会在响应头中带上新的`Last-Modified`值。
2. 如果不满足要求(资源的上次更新时间早于或等于`If-Modified-Since`)，表示资源未更新，HTTP Code `304`。

#### Last-Modified的问题

1. 如果在服务器上打开了原始文件，却没有真正修改内容，会造成 `Last-Modified` 变更，进而影响缓存协商命中；
2. `Last-Modified` 只能以秒计算，如果资源变更特别频繁，且对实时更新要求很高，`Last-Modified`无法满足要求；

如果遇到了这两类问题，可以使用`Etag`进行协商缓存。

### Etag / If-None-Match

RFC7232[^rfc7232-etag]中是这么描述`Etag`的。

[^rfc7232-etag]: https://datatracker.ietf.org/doc/html/rfc7232#section-2.3

> An entity-tag is an opaque validator for differentiating between multiple representations of the same resource, regardless of whether those multiple representations are due to resource state changes over time, content negotiation resulting in multiple representations being valid at the same time, or both.

按照文档的描述，`Etag`是一个资源的不透明验证器，这个验证器保证了只要资源没有变化，则(请求这个资源时返回的)`Etag`也不会变化。因此相比于`Last-Modified`而言，`Etag`可以以一种更精细的粒度控制资源的更新状态。

`Etag`和`If-None-Match`的匹配规则，和`Last-Modified`类似，就不赘述了。

#### Etag在Nginx上的问题

由于RFC明确表示`Etag`是一种不透明的验证器，那么不同的服务器可以自行决定`Etag`的生成方式。而 Nginx 内生成`Etag`的算法[^nginx-etag-src]依赖文件的**最后更新时间**和**文件大小**，这会带来2个问题：

[^nginx-etag-src]: <http://lxr.nginx.org/source/xref/nginx/src/http/ngx_http_core_module.c?r=7984%3Aae992b5a27b2#1698>

1. 部署在集群上的相同文件，`Etag`可能不一样
2. `Etag`和`Gzip`会有冲突

第一个问题是由于`Etag`和最后更新时间有关联，那么对于集群环境的部署来说，文件在多个机器上的更新的时间有可能不一致，这就会带来相同文件的`Etag`值不一致。

第二个问题则是因为，`Gzip`会压缩文件大小，因而对`Etag`的准确性有影响，所以在 v1.7.3 版本以前，如果开启了`Gzip`压缩，Nginx 会去掉`Etag`响应头；在v1.7.3版本之后，调整为将强`Etag` 转换为弱 `Etag`，而 Nginx 在协商时遇到弱`Etag`会直接返回原始文件，也就会导致协商缓存失效。

{{< admonition type=info title="建议" open=true >}}
如果没有上述`Last-Modified`的问题，可以去掉`Etag`，直接使用`Last-Modified`作为协商缓存的依据。
{{< /admonition >}}

到此，HTTP 缓存的原理和实现细节，就介绍完了。我们在实际开发中，为了尽可能多地利用好缓存的特性，需要针对不同的场景，配置不同的缓存策略。

## 缓存配置建议

好的缓存配置需要满足两个主要场景：

1. 尽可能在本地使用强缓存，减少对相同文件重复网络请求；
2. 资源更新后，客户端可以及时更新。

### 强缓存版本化的URL

版本化的URL指的是资源 url 中带有文件指纹或版本号。这类url有足够强的唯一标识能力，修改了内容，或发布了新版本，url 也会随之变更，所以客户端应该尽可能长地缓存这类资源。

``` http
Cache-Control: max-age=31536000
```

31536000(一年)是`max-age`能支持的最长时间，可以保证除非用户手动清空缓存，浏览器将不会向服务器发送重复的请求。

### 协商缓存非版本化的URL

对于网站来说，url 中没有版本化的信息，表示这个资源可能会变动。那么我们的原则是，保证资源永远是最新的，尽可能使用协商缓存，减少网络负载。

``` http
Cache-Control: no-cache
```

`no-cache`要求浏览器在每次使用URL的缓存版本前都必须与服务器重新验证。如果重新验证的过程发现资源未变更，则只返回304，不返回资源文件，可以极大地减少网络负载。

至于是依据`Etag`还是`Last-Modifiled`来做协商缓存，视情况而定，如果我们请求的是一个HTML文件，服务器是 Nginx 集群，我建议去掉`Etag`，直接使用`Last-Modifled`，毕竟网站的首页一般不会有毫秒级的变动，而且理论上，计算Etag肯定比比对时间要更耗时一些。

{{< admonition type=info title="http-equiv" open=true >}}
你可能尝试过在 HTML 的 head 中添加过类似这样的 Meta 信息：`<meta http-equiv="cache-control" content="max-age=0" />`，期望它能控制缓存，事实上，它不在规范中，也并不起作用。
{{< /admonition >}}

## 总结

浏览器缓存是我们日常开发中经常接触到的技术，有时也会碰到一些头疼的问题。网上有很多关于缓存的文章，这些文章大体的内容和逻辑没有什么问题，也能帮助我们解决一些临时的问题，但对于部分细节，则有些人云亦云，没有深究。比如计算响应寿命是从响应生成开始算，还是从请求发起开始算；内存缓存到底是什么缓存等。这类小误差，从实际应用场景来看，可以说是毫无影响，但对于一名开发者而言，只有躬身查阅规范文档，做几个demo实验，才能让我们真正了解到正确的逻辑和真正的知识，无论是对学习习惯，还是对能力的提升，都是大有裨益。希望阅读这篇文章的你，可以和我一起折木成林，一起成长！
