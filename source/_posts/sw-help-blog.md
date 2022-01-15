---
title: 欲善其事，必利其器 - 论如何善用ServiceWorker
author: ChenYFan
tags:
  - JSdelivr
  - ServiceWorker
  - 黑科技
categories:
  - 随心扯
des: 从唯一一个拥有国内节点能够加速海外资源的静态文件分发站JSDelivr因为各种滥用和敏感被封杀这一事件讲起，谈谈关于ServiceWorker奇技淫巧
hide: true
key: 'serviceworker,sw'
abbrlink: c0af86bb
index_img: 'https://npm.elemecdn.com/chenyfan-os@0.0.0-r2'
banner_img: 'https://npm.elemecdn.com/chenyfan-os@0.0.0-r2'
date: 2022-01-08 21:57:45
---

ServiceWorker作为前端革命领袖，毫不夸张地被誉为前端黑科技，此文将阐述如何巧妙的使用它来实现一些看起来匪夷所思的事情。

<!--more-->

> 从2022/1/8开始，本文将持续更新。当前状态：更新中

# 起因 - 巨厦坍塌

2021/12/20日，赶在旧年的末尾，一则`JSdelivrSSL证书错误`缓缓上了v2ex论坛热点。

此前JSD由于各种原因,曾经不正常了一段时间,所以大家并未对此感冒.正当人们以为这只是JSdelivr每年一度的`年经`阵痛,发个issue,过一段时间就好了的时候.官方直接爆出大料:**JSDelivr had lost their ICP license**

由此可见,过去的几年里,当人们发现JSD对个人面向国内加速拥有者无与伦比的效果时,各种滥用方式层出不穷:图床曾一阵流行,国内搜索引擎JSdelivr十有八九都是作为图床的,连PicGo插件都出了Github+JSdelivr图床;猛一点的,直接做视频床,甚至为了突破单文件20M限制开发了一套ts切片m3u8一条龙服务;作妖的,托管了不少突破网络审查的脚本和规则集;寻死的,添加了大量的政治宗教敏感,有些甚至不配称为宗教,直接上来就是骗钱的.

jas并不是没有发布许可条款，但这并不能阻止白嫖大军的进程。在羊毛大军中，只要是你是免费的、公益的，你就要做好被薅爆的结果。但是薅羊毛的前提是羊还活着，倘若羊被薅死了，哪来的羊毛给诸君所薅？

总之，不管怎样，JSDelivr在决定将节点设置为`NearChina`，可以肯定的是，在最近很长一段时间内，我们都无法享受国内外双料同时加速的快感，换句话说，jsd在中国就被永久地打入了冷宫。

视线转向国内，jsd的替代品并不少。早在我写[图床的千层套路](https://blog.cyfan.top/p/eb490c73.html)我就试着假想jsd不可用时，我们该用什么。最终我给出的一份较为完美的答案-npm图床，优点无非就是镜像多速度快，许可条款较为宽松，缺点也很明显，需要安装node，用专门的客户端上传。

那事情就逐渐变得扑朔迷离起来了，我们应当如何选择合理的CDN加速器呢。

这时候，我想起了前端黑科技Serviceworker。是的，这种情况下使用SW最为巧妙不过，它可以在后台自动优选最佳的CDN，甚至可以用黑中黑`Promise.any`打出一套漂亮的并行拳。经过两天的完善，我终于写出了一套具有离线可达、绕备、优选CDN、跟踪统计合一的SW脚本。[此博客使用的SW](/sw.js)

接下来我将从头开始讲述ServiceWorker的妙用。

# Before Start

## What Is The ServiceWorker

网上对于SW的解释比较模糊，在这里，我将其定义为`用户浏览器里的服务器`，功能强大到令人发指。是的，接下来的两张图你应该能显著的看到这一差距：

![没有ServiceWorker中继，平淡无奇](https://npm.elemecdn.com/chenyfan-os@0.0.0-r5/1.jpg)

![ServiceWorker中继，刺激拉满](https://npm.elemecdn.com/chenyfan-os@0.0.0-r5/2.jpg)

在第一张图中，用户和服务器的关系直的就像电线杆，用户想要什么，服务器就还给他什么。

第二张图中，用户在被ServiceWorker控制的页面中，无论向哪个服务器发起请求，其过程都会被SW捕获，SW可以仿佛不存在一般单纯地请求服务器，返回原本应该返回的内容【透明代理】；也可以对当前服务器返回的内容进行随意的捏造、修改【请求修改结果】；甚至可以将请求指向完全另一台服务器，返回不是此服务器应该返回的内容【移花接木】；当然，SW也可以直接返回已经存储在本地的文件，甚至离线的时候也能返回【离线访问可达性】。

由于SW对于用户页面的操纵实在过于强大，因此，它被设计成`不可跨域请求`、`SW脚本必须在同一域名下`、`必须在HTTPS条件下运行`、`不可操纵DOM和BOM`，同样的，为了避免阻塞和延迟，SW也被特意设计成`完全异步的`。这些点将会在后面一一讲述。

> 当然，开发者至上，为了方便本地调试，本机地址`localhost`和`127.0.0.1`被浏览器所信任，允许以`非HTTPS`方式运行serviceworker。

## What Relationship Between ServiceWorker and PWA

很多人看到sw，第一反应是PWA，即渐进式Web应用。实际上，SW确实是PWA的核心与灵魂，但SW在PWA中起的主要作用是缓存文件，提供给离线访问。并没有完整地发挥出SW的巧妙用法。

SW可以完全脱离PWA存在，当然，PWA可离不开SW ：）

## And WorkBox ?

WorkBox是谷歌开发的一款基于SW的缓存控制器，其主要目的是方便维护PWA。核心依旧是SW，但还是没有SW原本的自定义程度高（

## Why Not WorkBox ?

首先，博客呢，是没有必要用PWA，有SW做中间件足矣。同时，WorkBox只能简单的缓存数据，并不能做到拦截篡改请求的功能，尤其不能精准把握每一个资源的缓存情况，自定义程度并不高。

~~自己编写SW，格局就打开了~~

# Start From Zero

## 安装 / Install

首先，SW的本质是JS脚本，要安装它必须要经过一个html。毕竟，只有拿到了html，JS才能运行于DOM上下文。

剥离层层加成，安装的代码只有一行

```JavaScript
navigator.serviceWorker.register('/sw.js')
```

其中，`/sw.js`即为ServiceWorker脚本所在，由于安全性，你不能加载跨域的SW。

例如，当前网页为`https://blog.cyfan.top`，以下加载位置是允许的

```url
/sw.js
https://blog.cyfan.top/sw.js
```

以下加载是不允许的:

```url
http://blog.cyfan.top/sw.js#非HTTPS
https://cyfan.top/sw.js#非同一域名，视为跨域
https://119.91.80.151:59996/sw.js#虽然为同一文件,但非同一域名，视为跨域
./sw.js#容易造成SW脚本获取路径不一致
```

在加载前，我们最好判断一下dom是否加载完了，不然安装sw可能会卡dom

加载完成后，register函数将返回一个`Promise`，由于前端大多不适用于`异步`，我们通常以`同步`的方式`.then()`和`.catch()`来获取是否加载成功。

为了方便判断脚本是否能够加载，我们还要判断navigator里有无sw这一属性`'serviceWorker' in navigator`。

由于SW安装后，页面需要刷新后才能交给SW所宰割，同时为了避免浏览器缓存的影响，我通常采用修改`search`的方式强刷新，而不是通过`reload`函数。同样的，为了避免刚安装完就刷新的尴尬感，建议用`setTimeout`延迟一秒刷新。

简易的完整安装代码如下:

```html
<script>
window.addEventListener('load', async () => {
    navigator.serviceWorker.register(`/sw.js?time=${new Date().getTime()}`)
        .then(async reg => {
            //安装成功，建议此处强刷新以立刻执行SW
            setTimeout(() => {
                window.location.search = `?time=${new Date().getTime()}`
            }, 1000)
        }).catch(err => {
            //安装失败，错误信息会由err传参
        })
});
</script>
```

一刷新，世界就变成了ServiceWorker的瓮中之鳖，接下来，该是SW脚本正式登场的时候了。

## SW安装初始化 / Installations

首先，先尴尬的开一个空缓存列表：

```js
const CACHE_NAME = 'ICDNCache';//可以为Cache版本号，但这样可能会导致缓存冗余累积
let cachelist = [];
```

`cachelist`里面填写的是预缓存网址，例如在离线时返回的错误页面。此处不宜添加过多网址，此处点名@一下Akilar。

此处我建议只缓存离线页面展示的内容:

```js
let cachelist = [
    '/offline.html',
    'https://npm.elemecdn.com/chenyfan-os@0.0.0-r6'
];
```

同时监听sw安装时开启此缓存空间：

```js
self.addEventListener('install', async function (installEvent) {
    self.skipWaiting();
    installEvent.waitUntil(
        caches.open(CACHE_NAME)
            .then(function (cache) {
                console.log('Opened cache');
                return cache.addAll(cachelist);
            })
    );
});
```

由于SW完全没有办法访问DOM，因此对于全局变量，不应当用`window`，而是`self`指代自己。

`addEventListener`这一监听器将监听`install`,也就是这一段代码只会在脚本首次安装和更新时运行.

`skipWaiting`的作用是促进新版本sw跳过waiting这一阶段，直接active。

> 关于SW的状态（waiting，installing，activing）将在文后详细解释。

`installEvent.waitUntil`的作用是直接结束安装过程的等待，待会在后台完成开启缓存空间这一操作。


`cache.addAll`将会直接获取`cachelist`里面所有的网址并直接缓存到CacheStorage。如果此处网址过多，将在页面加载时疯狂请求所有的url~~(例如1k个)~~

现在，SW初始化已经完成了。接下来，我将讲述SW如何捕获页面的请求。

## 捕获请求 / Fetch Event

### 添加监听器 / AddEventListener

```js
self.addEventListener('fetch', async event => {
    event.respondWith(handle(event.request))
});

const handle = async(req)=>{
    //do something
}
```

第一行很简单，绑定一个监听器，监听`fetch`事件，即网页向服务器获取请求，也就是相当于前端的`XMLHTTPRequest`

`event.respondWith`即设定返回内容，交给`handle`主函数处理，传参`event.request`。这是一个`Request`对象，里面包含了请求的详细信息。

接下来，我们开始实战吧。

> 以下所有内容均针对handle修改

### 透明代理 / Transparent Proxy

顾名思义，此实战脚本的作用是SW代理目前的所有流量但不进行修改，仿佛SW不存在一般。

```js
const handle = async(req)=>{
    return fetch(req)
}
```

`fetch`这个函数相当于前端的`ajax`或者`XMLHTTPRequest`，作用是发起一个请求，获得一个返回值。由于sw不可访问`window`，在sw中是无法使用`ajax`或`XMLHTTPRequest`。同时，`fetch`是一个异步函数，直接调用它会返回一个`Promise`。

`fetch`只能传递`Requset`对象,而`Requset`对象有两个参数`(url,[option])`,第一个参数是网址,第二个参数为`Request`的内容,例如`body`或`header`。

此脚本适用于卸载`ServiceWorker`的替换脚本。因为sw在无法拉取新版本时不会主动卸载，依旧保持运行，填入一个透明代理sw即可。

由于SW冷启动【即页面关闭后SW】处于暂停状态是从硬盘读取的，这会导致第一次请求有少许性能延迟[~10ms]。

### 篡改请求 / Edit Requset

对于一张图片，有时候服务端会变态到让你必须用`POST`协议才能获得，此时用SW篡改最为方便。

```js
const handle = async (req) => {
    if ((req.url.split('/'))[2].match('xxx.com')) {
        //xxx.com为图片所在域名
        return fetch(req.url, {
            method: "POST"
        })
    }
    return fetch(req)
}
```

> 注意，在ServiceWorker里面，header头是不能修改refferer和origin的，因此此方法无法绕开新浪图床反盗链

### 篡改响应 / Edit Response

这个例子会检测返回内容，若为html，将把所有的"TEST"都替换成"SHIT"

```js
const handle = async (req) => {
    const res = await fetch(req)
    const resp = res.clone()
    if (!!resp.headers.get('content-type')) {
        if (resp.headers.get('content-type').includes('text/html')) {
            return new Response((await resp.text()).replace(/TEST/g, 'SHIT'), {
                headers: resp.headers,
                status: resp.status
            })
        }
    }
    return resp
}
```

`const resp = res.clone()`由于`Response`的`body`一旦被读取，这个`body`就会被锁死，再也无法读取。`clone()`能够创造出响应的副本用于处理。

`resp.headers.get('content-type')`通过读取响应的头，判断是否包含`text/html`，如果是，将响应以`text()`异步流的方式读取，然后正则替换掉响应内容，并还原头和响应Code。

返回的内容必须是`Response`对象，所以`new Response`构建一个新对象，并直接返回。不匹配html头将直接原封不动地透明代理。

### 移花接木 / Graft Request To Another Server

`unpkg.zhimg.com`是`unpkg.com`的镜像网站。此脚本将会把所有的`unpkg.com`流量直接拦截到`unpkg.zhimg.com`，用于中国大陆内CDN加速。

由于npm镜像固定为GET请求方式并且没有其他鉴权需求，所以我们没有必要还原`Request`其他数据。

```js
const handle = async (req) => {
    const domain = req.url.split('/')[2];
    if (domain.match("unpkg.com")) {
        return fetch(req.url.replace("https://unpkg.com", "https://zhimg.unpkg.com"));
    }
    else {
        return fetch(req)
    }
}
```

`domain.match`捕获请求中是否有待替换域名，检查出来后直接`replace`掉域名，如果没有匹配到，直接透明代理走掉。

### 持久化缓存 / Cache Persistently

对于来自CDN的流量，大部分是持久不变的，因此，如果我们将文件获得后直接填入缓存，之后访问也直接从本地缓存中读取，那将大大提升访问速度。

```js
const handle = async (req) => {
    const cache_url_list = [
        /(http:\/\/|https:\/\/)cdn\.jsdelivr\.net/g,
        /(http:\/\/|https:\/\/)cdn\.bootcss\.com/g,
        /(http:\/\/|https:\/\/)zhimg\.unpkg\.com/g,
        /(http:\/\/|https:\/\/)unpkg\.com/g
    ]
    for (var i in cache_url_list) {
        if (req.url.match(cache_url_list[i])) {
            return caches.match(req).then(function (resp) {
                return resp || fetch(req).then(function (res) {
                    return caches.open(CACHE_NAME).then(function (cache) {
                        cache.put(req, res.clone());
                        return res;
                    });
                });
            })
        }
    }
    return fetch(req)
}
```

`cache_url_list`列出所有待匹配的域名(包括http/https头是为了避免误杀其他url)，然后`for`开始遍历待列表，如果url中匹配到了，开始执行返回缓存操作。

cache是一个近似于Key/Value(键名/键值)，只要有对应的`Request`(`KEY`)，就能匹配到响应的`Response`(`VALUE`)。

`caches.match(req)`将会试图在CacheStorage中匹配请求的url获取值，然后丢给管道同步函数`then`，传参`resp`为Cache匹配到的值。

此时管道内将尝试返回resp，如果resp为`null`或`undefined`[即获取不到对应的缓存]，将执行fetch操作，fetch成功后将`open`打开CacheStorage，并`put`放入缓存。此时如果`fetch`失败将直接报错，不写入缓存。

在下一次获取同一个URL的时候，缓存匹配到的将不再是空白值，此时`fetch`不执行，直接返回缓存，大大提升了速度。

由于npm的cdn对于latest缓存并不是持久有效的，所以我们最好还是判断一下url版本中是否以@latest为结尾。

```js
const is_latest = (url) => {
    return url.replace('https://', '').split('/')[1].split('@')[1] === 'latest'
}
//...
for (var i in cache_url_list) {
    if (is_latest(req.url)) { return fetch(req) }
    if (req.url.match(cache_url_list[i])) {
        return caches.match(req).then(function (resp) {
            //...
        })
    }
}
```


### 离线化缓存 / Cache For Offline

对于博客来说，并不是所有内容都是一成不变的。传统PWA采用SW更新同时刷新缓存，这样不够灵活，同时刷新缓存的版本号管理也存在着很大的漏洞，长时间访问极易造成庞大的缓存冗余。因此，对于博客的缓存，我们要保证用户每次获取都是最新的版本，但也要保证用户在离线时能看到最后一个版本的内容。

因此，针对博客来说，策略应该是先获取最新内容，然后更新本地缓存，最后返回最新内容；离线的时候，尝试访问最新内容会回退到缓存，如果缓存也没有，就回退到错误页面。

即：

```
Online:
发起Request => 发起fetch => 更新Cache => 返回Response
Offline:
发起Request => 获取Cache => 返回Response
```


```js
const handle = async (req) => {
    return fetch(req.url).then(function (res) {
        if (!res) { throw 'error' } //1
        return caches.open(CACHE_NAME).then(function (cache) {
            cache.delete(req);
            cache.put(req, res.clone());
            return res;
        });
    }).catch(function (err) {
        return caches.match(req).then(function (resp) {
            return resp || caches.match(new Request('/offline.html')) //2
        })
    })
}
```

`if (!res) { throw 'error' }` 如果没有返回值，直接抛出错误，会被下面的Catch捕获，返回缓存或错误页面

`return resp || caches.match(new Request('/offline.html'))` 返回缓存获得的内容。如果没有，就返回从缓存中拿到的错误网页。此处offline.html应该在最开始的时候就缓存好



### 并行请求 / Request Parallelly

> **正在施工**