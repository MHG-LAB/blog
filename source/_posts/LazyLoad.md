---
title: 'LazyLoad_懒加载,将无形等待时间化为有形'
tags:
  - Hexo
  - NPM
  - LazyLoad
  - 奇淫巧技
  - 等待
  - 图片
  - 加载
categories: 好方法
copyright: true
abbrlink: f47a8d59
date: 2020-03-09 07:54:10
---

说实话，很早以前接触懒加载是在看别人提到过，但当时对此不温不火的，因为当时部署在Coding上，速度基本加载图片几毫秒的事。而且当时每篇文章也就三四张图，基本不影响加载时间。

可是现在部署在github上，国内一访问，终于明白LazyLoad的妙用了。

即使加载了CloudFlareCDN，选了个比较快的CDNIP加载Hosts中，也只有120kb/s的速度，致命的是，我上一篇博文图片将近31MB。

![图片](https://unpkg.zhimg.com/chenyfan-oss@1.0.0/pic/lazy/1.jpg "上一次手贱,图片添加的比较多.")

以120kb/s的速度,以前加载个不到1MB的博文(Live2D看板娘本身自带LazyLoad)也就几秒钟的事情,加载个30MB的网页...

宏观来讲,就是打开博文一片空白,既看不到文字也看不到图片.

那么这就有一个问题,网页一片空白,你是怪博主还是怪网络呢?

很简单,肯定是怪博主,因为如果是网络,那也不至于连文字都加载不出来吧?

因为网页加载时,会将页面里所有资源加载完毕时才会显示,在加载完整之前是不显示的.

最最致命的是,博客使用CloudFlareCDN,总会遭到XXX的恶意丢包:
(测试环境为IPv6)

```
ping blog.cyfan.ga -t

正在 Ping blog.cyfan.ga [2606:4700:3033::681c:10de] 具有 32 字节的数据:
来自 2606:4700:3033::681c:10de 的回复: 时间=438ms
请求超时。
来自 2606:4700:3033::681c:10de 的回复: 时间=462ms
来自 2606:4700:3033::681c:10de 的回复: 时间=257ms
来自 2606:4700:3033::681c:10de 的回复: 时间=270ms
来自 2606:4700:3033::681c:10de 的回复: 时间=286ms
来自 2606:4700:3033::681c:10de 的回复: 时间=279ms
来自 2606:4700:3033::681c:10de 的回复: 时间=288ms
来自 2606:4700:3033::681c:10de 的回复: 时间=288ms
来自 2606:4700:3033::681c:10de 的回复: 时间=281ms
请求超时。
来自 2606:4700:3033::681c:10de 的回复: 时间=285ms
来自 2606:4700:3033::681c:10de 的回复: 时间=281ms
来自 2606:4700:3033::681c:10de 的回复: 时间=289ms
来自 2606:4700:3033::681c:10de 的回复: 时间=287ms
来自 2606:4700:3033::681c:10de 的回复: 时间=288ms
请求超时。
来自 2606:4700:3033::681c:10de 的回复: 时间=291ms

2606:4700:3033::681c:10de 的 Ping 统计信息:
    数据包: 已发送 = 18，已接收 = 15，丢失 = 3 (16% 丢失)，
往返行程的估计时间(以毫秒为单位):
    最短 = 257ms，最长 = 462ms，平均 = 304ms
Control-C
^C
```

如果丢包丢的正好是图片,即使加载完毕,估计也是这么一个半死不活的图标:

![图片](https://unpkg.zhimg.com/chenyfan-oss@1.0.0/pic/lazy/2.jpg "...")

啧啧,有伤风雅有伤风雅.....((/- -)/

这个时候,懒加载就变得非常重要了.

# 说了那么多,什么是懒加载?

名符其义,懒加载懒加载,你懒得看我就懒得加载.

说的专业点,就是只加载在视野里的图片,不在视野里的图片先不加载,用一张相同的图片代替.

这样绝对有用,只加载用户可能看的图片,对于可能不看的图片暂时不加载,当用户滑动到这个地方再加载.

实际上懒加载运用的很广泛,举一个常见的例子,当你浏览知乎时,难道不是划到哪图片才加载到哪吗?只不过图片加载时间太快,很多人都没有留意罢了.

图片懒加载是提升网站性能和用户体验的一个非常很好方式，并且几乎所有的大型网站都使用到了，比如微博，仅把用户可见的部分显示图片，其余的都暂时不加载，做法就是：让所有图片元素src指向一个小的站位图片比如loading，并新增一个属性(如data-original)存放真实图片地址。每当页面加载（或者滚动条滚动），使用JS脚本将可视区域内的图片src替换回真实地址，并做请求重新加载。

# 所以,你依旧没读懂?

直接看例子吧.

当时,我的上一篇博文正常国内加载到显示文字用了将近10min,也就是空白页显示了10min,纵使在国外加载也用了4min.

也就是正常游客看起来根本加载不了,只有空白页.

现在,当使用懒加载后:

![图片](https://unpkg.zhimg.com/chenyfan-oss@1.0.0/pic/lazy/3.gif "请注意开发者工具的监视项")

(为了降低用户查看该动图的时间,我进行了压缩减帧:30fps,大小约有4.3MB,请耐心等待)

看看开发者工具Network一项,我们很自然发现,当我滑动到某张图片上方,某张图片才开始加载,使用一张动图作为未加载的动画.

这,就是懒加载的奥秘.

# 如何使用:

## html



我们正常引用一张图片,格式应该如下:


```html
<img src="/xxx.jpg" alt="图片" title="title">
```

懒加载在html的图片语言中使用了回源, `data-original` 表示真正的图片位置,而 `src` 则是懒加载的外观图片.


```html
<img src="https://unpkg.zhimg.com/chenyfan-oss@1.0.0/pic/lazy.gif" data-original="/xxx.jpg" alt="图片" title="title">
```

这样,当打开时网站回家再 `src` 的资源,只有当滑动到图片上方,再加载 `data-original` 原始图片.

## Hexo

当然,Hexo本来使用Markdown,而Markdown本来和html就是亲兄弟,如果不介意,用 `<img>`  标签也成.

但是对于已经有了的图片,这样做简直就是噩梦,总不可能一个一个换过去吧.

这时候,建议使用 `Troy` 大佬开发的 `hexo-lazyload-image`

### 安装`hexo-lazyload-image`:

npm安装:

```
npm install hexo-lazyload-image --save
```

> 关于npm安装慢,请参考以前写的一篇[博文>>](/2019/07/19/国内加快NPM下载速度/)

接着再hexo配置文件 `_config.yml `

```
lazyload:
  enable: true 
  onlypost: false
  loadingImg: # eg. ./images/loading.png
```

- `enable` 不解释
- `onlypost` 指是全部站点都用懒加载还是对于文章使用懒加载,默认为 `false`
- `loadingImg` 懒加载所需要的先前图片,目前看来是图片都行,gif也成

post一下,试试吧.



### 实现原理

Hexo-lazy-image 实现原理
因为文章都是使用markdown来编写的，所以不可能要求我们在markdown里将所有图片路径都指向站位图片，并附加另一个属性，所以，这个工作必须留给hexo的generate部分来做。

最终可分为两步：

在 `hexo after_post_render`事件或者`after_render:html`事件里将生产出来的文章html代码中所有img元素都加上 `data-original` 属性，并把 `src` 值付给他， 然后在将 `src` 值致为 `loading` 图片
注入 `simple-lazyload` 脚本在每个页面最后面，当页面加载过后负责判定当前需要重新加载的图片。

原大佬博客: [Troy's 博客](https://troyyang.com/2017/08/06/hexo-lazyload-image/)

## 意外发现

实际上懒加载还帮我掩饰了一个问题,对于一些失效的图片再也不会加载不出来了,(因为被懒加载图片替换掉了)

# 结尾:

不知道大家有没有这样的心理:

打开一篇文章,图片死死加载不出来,折腾好长一段时间,终于加载出来了,却发现这些图片毫无作用,这时候估计所有人心里都很恼火吧.

目前看来,懒加载基本上全是好处:

1. 缩短不需要看图片用户加载时间.
2. 提升网页打开速度
3. 暗示用户网络有问题而不是网站有问题 ~~(臭婊脸)~~
4. 美化未加载网页
5. 将用户等待时间从主观上缩短,暗示用户等待时间变短,让用户知道不是卡了,而是正在加载.

当然,懒加载所需的图片要尽量的小,我的懒加载图片是剽窃百度的,压缩了一下约300kb,加载所需时间为0.2s.

这是我现在用的懒加载图片：

![LazyLoad](https://unpkg.zhimg.com/chenyfan-oss@1.0.0/pic/lazy.gif)
