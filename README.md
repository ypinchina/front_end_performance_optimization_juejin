# 掘金小册——前端性能优化与实践

本项目将是掘金小册—— 前端性能优化与实践 的学习，本文件将记录学习笔记
## 知识体系： 从一道面试题说起

### 在展开性能优化的话题之前，我想先抛出一个老生常谈的面试问题：

* 从输入 URL 到页面加载完成，发生了什么？

    
我们将这个过程切分为如下的过程片段：

1. DNS 解析
2. TCP 连接
3. HTTP 请求抛出
4. 服务端处理请求，HTTP 响应返回
5. 浏览器拿到响应数据，解析响应内容，把解析的结果展示给用户
大家谨记，我们任何一个用户端的产品，都需要把这 5 个过程滴水不漏地考虑到自己的性能优化方案内、
反复权衡，从而打磨出用户满意的速度。

## 网络篇
1. webpack性能调优与Gzip

首先 ，根据引言的问题 ，我们先做网络部分的优化， 其中DNS和TCP连接前端无法改变太多，因此
我们主要从HTTP优化入手。
HTTP的优化主要分为两个部分
* 减少请求次数
* 减少单次请求所花费的时间

这两个优化无疑是指向我们日常开发的常见操作—— **资源的合并与压缩**
我们通常都是使用构建工具来实现这些需求，而业界巨无霸的构建工具无疑是webpack

### webpack的性能瓶颈

对于资源的压缩与合并，是老生常谈的问题，建议查看文档。
这里主要讲的是优化webpack,webpack性能调优的方向主要有两个：

1. webpack构建时间太长
2. webpack打包太大

### 构建过程提速策略

1. 不要让loader做过多的事情

babel-loader显然是强大的，但也是慢的
常见的优化方式是使用include和exclude进行文件范围限定。
刨除node_modules文件夹和bower_components，以下是官方文档的例子

```javascript
module: {
  rules: [
    {
      test: /\.js$/,
      exclude: /(node_modules|bower_components)/,
      use: {
        loader: 'babel-loader',
        options: {
          presets: ['@babel/preset-env']
        }
      }
    }
  ]
}
```
### 缓存之前构建过的js
使用文件夹限定的优化提升是有限的，所以我们还可以开启缓存将转译的结果
保存到文件系统，则至少可以将 babel-loader 的工作效率提升两倍。只需要
增加参数即可

将Babel编译过的文件缓存起来，下次只需要编译更改过的代码文件即可，这样可以大幅度加快打包时间。

`loader: 'babel-loader?cacheDirectory=true'`

讲完了loader，还需要放眼于plugins。
### 不要放过第三方库
每次构建的时候，一些第三方库都会重新编译，但是他们短时间内不会有变化，庞大的第三方库
的编译会极大消耗时间。

最好的方案是使用dllPlugin,这个插件会把第三方库单独打包到一个文件中，
这个文件就是一个单纯的依赖库。这个依赖库不会跟着你的业务代码一起被重新打包，
只有当依赖自身发生版本变化时才会重新打包。

使用dllPlugin处理文件分两步走
1. 基于 dll 专属的配置文件，打包 dll 库
2. 基于 webpack.config.js 文件，打包业务代码

然后，根据编写dll处理文件（此处理文件代码看文档），使用dllPlugin,
会生成两个文件
* vendor.js
* vendor-manifest.js

其中vendor.js是第三方库文件，而vendor-manifest.js则是则用于描述
每个第三方库对应的具体路径

然后只需要在webpack.config.js文件中配置dllPlugin的manifest关联到vendor.js即可
完成本次dllPlugin的优化（此处参考文档）

### 使用Happypack——将 loader 由单进程转为多进程

受限于 Node 是单线程运行的webpack 是单线程的，就算此刻存在多个任务，你也只能排队一个接一个地等待处理，
所以 Webpack 在打包的过程中也是单线程的，特别是在执行 Loader 的时候，
长时间编译的任务很多，这样就会导致等待的情况。HappyPack和ThreadLoader作用是一样的，
都是同时执行多个进程，从而加快构建速度。而Thread-Loader是webpack4官方提出的，而HappyPack作者已经
很久没维护了。

采用HappyPack开启多进程Loader

虽然webpack是单线程，但好在用户是多核电脑，Happypack 会充分释放 CPU 在多核并发方面的优势，
帮我们把任务分解给多个子进程去并发执行，大大提升打包效率。

这两种工具配置请参考文档，原理是把loader 的配置转移到 HappyPack 中去就好，
我们可以手动告诉 HappyPack 我们需要多少个并发的进程

需要注意的是，开启多进程的意义在于需要构建的文件的体积足够大，如果不够大，
不然效果不够明显，明显有高射炮打小苍蝇的感觉。因为进程开启需要时间，
进程之间的通信也需要时间，如果这点时间都不能忽略不计，那么意义不大。（太小会反而增加构建用时）

### 删除冗余代码
从webpack2开始，webpack开始支持es6自带的es module语法， 因此开始引入支持Tree-shaking
Tree-shaking原理就是基于 import/export来优化剪掉多余的代码。webpack5中mode为production
时默认开启Tree-shaking

* Tree-shaking显然是针对模块化大型代码裁剪比较给力。而精细程度的代码优化裁剪，
使用UglifyJsPlugin，可以优化删除console.log,和注释

### 按需加载
* 异步路由
原理是路由分包按需加载，Vue中使用import或者require来实现,如：
`const Editor = () => import('@/views/Editor') `

## Gzip压缩原理

具体的做法非常简单，只需要你在你的 request headers 中加上这么一句：

accept-encoding:gzip


Gzip 压缩背后的原理，是在一个文本文件中找出一些重复出现的字符串、临时替换它们，
从而使整个文件变小。根据这个原理，文件中代码的重复率越高，那么压缩的效率就越高，
使用 Gzip 的收益也就越大。反之亦然。
## 图片———— 性能与质量平衡点的博弈

前言
《高性能网站建设指南》的作者 Steve Souders 曾在 2013 年的一篇 博客 中提到：
曾经他以为性能优化的大头在于CSS与JS，后来他意识到自己错了，真正的大头在于网络图片

## 图片的性能优化需要在图片压缩性能和图片质量之间找到一个合理的平衡点

### 前置知识
二进制位数与色彩的关系：在计算机中，像素用二进制数来表示。不同的**图片格式**中像素与二进制位数之间的对应关系是不同的。一个像素对应的二进制位数越多，
它可以表示的颜色种类就越多，成像效果也就越细腻，文件体积相应也会越大。

1. jpg/jpeg
特点： **有损压缩，体积小，加载快，不支持透明**
jpg是色彩丰富的图片，即使有损压缩，也能保证图片的质量。jpg即使被压缩了50%,也能保证有60%以上的图片质量。jpg以24位二进制存储单个图，能呈现1600万中色彩
一般用于背景图和轮播图以及banner大图
缺陷： 不支持透明， 不善于处理LOGO和矢量图等线条感丰富的图片

2. png-8, png-24
特点： **无损压缩，质量高，体积大，支持透明**

-8可呈现256种色彩，而-24约1600万种色彩，png什么都好，唯一的BUG就是太大了。需要权衡何时用-8，何时用-24

应用场景： 对于多色彩鲜艳的大图片，一般是使用.jpg来呈现，因为若是使用.png，那实在是太大了，而png一般用于在较小、色彩简单或透明的logo小图。

3. svg
关键字 **文本文件、体积小、不失真，兼容性好**

* 文本文件 —— 可嵌入html文件里，作为DOM的一部分。 

SVG的缺点（局限性）： 1. 渲染成本高，对于性能来说很不利； 2. 有其他图片格式所没有的学习成本（可编程）

4. Base64与雪碧图（图像精灵）
两者都是用于解决小图标的解决方案: 
雪碧图是将多个小图标拼接在一张图片中，以减少http请求次数。
Base64是作为雪碧图的**补充**而存在的
Base64不是图片格式，而是一种编码方式。将图片编译成字符串存放在html文件或者css文件中减少这张图片的请求次数。但是并非所有图片都适用于这种编码方式。因为使用该编码会使图片的大小
膨胀为原文件的4/3（这是由 Base64 的编码原理决定的）。所以过大的图片会膨胀会使css文件或者HTML文件文件体积显著增大，即使减少了http请求次数，也无法弥补体积膨胀带来的性能开销，得不偿失。

一张图片使用Base64的场景
- 小图，图片尺寸很小
- 图片更新频率非常低（不需我们重复编码和修改文件内容，维护成本较低）

* Base64编码转换工具
- 最推荐的还是webpack的url-loader（全自动编译 + 用户设置图片尺寸转换阈值）
- 在线网页工具

5. WebP 年轻的全能型选手
于2010年被提出，是 Google 专为 Web 开发的一种旨在**加快图片加载速度的图片格式**，它支持有损压缩和无损压缩。
* 优点： WebP 像 JPEG 一样对细节丰富的图片信手拈来，像 PNG 一样支持透明，像 GIF 一样可以显示动态图片——它集多种图片文件格式的优点于一身。
对于有损与无损的webp图像都降低了体积。
图片优化是**性能与质量的博弈**, webp毫无疑问是赢家。 

* 缺点： 太新了，仅chrome支持，其他浏览器兼容性都不好； 
webp会占用更多服务器的负担 —— 和编码的jpg文件相比，同质量的webp文件会占用更多的计算资源。

## 存储篇 1：浏览器缓存机制介绍与缓存策略剖析

一般我们提到的缓存都是只认为是http缓存，实际上浏览器缓存机制有四个方面，它们按照获取资源请求时的优先级依次排列如下：
1. Memory Cache
2. Service Worker Cache
3. Http Cache
4. Push Cache

在chrome 的f12面板的network可以看到前三个 from memory cache , from service worker cache。 而 push cache很特殊，是HTTP2的新特性。

### HTTP缓存
这是我们日常开发中最为熟悉的缓存，它分为**强缓存**与**协商缓存**。优先级较高的是强缓存，在命中强缓存失败的情况下才会走协商缓存。
* 强缓存
强缓存只要是根据HTTP请求头中的expires和cache-control两个字段来控制的。强缓存中，当请求再次发出时，浏览器会根据这俩来判断目标资源
是否“命中强缓存”，如果命中则直接从缓存中获取资源，**不会再与服务端发生通信。** 命中强缓存则返回的HTTP状态码为200

#### 强缓存的实现： 从expires到cache-control

* expires 

是最开始使用的，它是一个记录具体时间的字符串，请求响应时，在响应头中将过期时间写入expires字段。
expires: Wed, 11 Sep 2019 16:12:18 GMT
当用户再次试图向服务器请求资源时，浏览器就会将本地客户端时间和expires字段时间作对比，如果未过期则直接从强缓存中取。
缺点很明显： 它最大的问题在于对“本地时间”的依赖。如果服务端和客户端的时间设置可能不同，或者我直接手动去把客户端的时间改掉，那么 expires 将无法达到我们的预期。

所以在HTTP1.1时新增cache-control用于**完全替代**expires 

cache-control: max-age=31536000
它用过设置一个时间长度而不是具体时间从而规避了expires带来的问题，当两个字段同时存在时，cache-control优先度更高

cache-control的特殊取值: no-cache和no-store
前者绕开浏览器，直接向服务端去确认资源是否过期（即走协商缓存路线）
后者更无情，顾名思义是不适用任何缓存策略，在 no-cache 的基础上，它连服务端的缓存确认也绕开了，只允许你直接向服务端发送请求、并下载完整的响应。

cache-control: max-age=3600, s-maxage=31536000

**s-maxage 优先级高于 max-age，两者同时出现时，优先考虑 s-maxage。如果 s-maxage 未过期，则向代理服务器请求其缓存内容。**

这个 s-maxage 不像 max-age 一样为大家所熟知。的确，在项目不是特别大的场景下，max-age 足够用了。但在依赖各种代理的大型架构中，我们不得不考虑**代理服务器**的缓存问题。s-maxage 就是用于表示 cache 服务器上（比如 cache CDN）的缓存的有效时间的，并只对 public 缓存有效。s-maxage仅在代理服务器中生效，客户端中我们只考虑max-age。
* public 与 private
public 与 private 是针对资源是否能够被代理服务缓存而存在的一组对立概念。

如果我们为资源设置了 public，那么它既可以被浏览器缓存，也可以被代理服务器缓存；如果我们设置了 private，则该资源只能被浏览器缓存。private 为默认值。

#### 协商缓存： 浏览器与服务器合作之下的缓存策略
协商缓存依赖于服务端与客户端之间的通讯；
协商缓存机制下，浏览器需要向服务器询问缓存的相关信息，进而判断是重新发起请求，下载完整响应，还是从本地获取缓存的资源。
如果服务端提示缓存文件未改动（Not Modify),资源则会被重定向到浏览器缓存，**这种情况下网络请求的状态码是304**。

#### 协商缓存的实现： 从last-modified到ETag
last-modified保存一个时间戳，如果用户启用了协商缓存，则会在首次返回是在响应头中收到：
Last-Modified: Fri, 27 Oct 2017 06:35:57 GMT

随后我们发的关于该资源的每次请求，都会带上一个If-modified-since的字段，值正是上一次last-modified的值：
If-Modified-Since: Fri, 27 Oct 2017 06:35:57 GMT
服务器接收到这个时间戳后，会比对该时间戳和资源在服务器上的最后修改时间是否一致，从而判断资源是否发生了变化。如果发生了变化，就会返回一个完整的响应内容，并在 Response Headers 中添加新的 Last-Modified 值；否则，返回如上图的 304 响应，Response Headers 不会再添加 Last-Modified 字段。

last-modified的弊端： 

* 用户编辑过文件，但是没修改文件内容，文件更新了新的保存时间，服务器还是认为文件有变化，因此其实本不用重新请求，却被服务器重新发起请求

* 修改文件保存过快，比如花了100ms完成了改动，由于if-modified-since只能检查到以秒为最小计量单位的时间差，所以它是感知不到这个改动的 —— 即需要重新请求时，没有重新请求。

这两个场景其实指向了同一个 bug——服务器并没有正确感知文件的变化。为了解决这样的问题，Etag 作为 Last-Modified 的补充出现了。

Etag 是由服务器为每个资源生成的唯一的标识字符串，这个标识字符串是基于文件内容编码的，只要文件内容不同，它们对应的 Etag 就是不同的，反之亦然。因此 Etag 能够精准地感知文件的变化。

Etag 和 Last-Modified 类似，当首次请求时，我们会在响应头里获取到一个最初的标识符字符串，举个🌰，它可以是这样的：

ETag: W/"2a3b-1602480f459"

那么下一次请求时，请求头里就会带上一个值相同的、名为 if-None-Match 的字符串供服务端比对了：

If-None-Match: W/"2a3b-1602480f459"

Etag 的生成过程需要服务器额外付出开销，会影响服务端的性能，这是它的弊端。因此启用 Etag 需要我们审时度势。正如我们刚刚所提到的——Etag 并不能替代 Last-Modified，它只能作为 Last-Modified 的补充和强化存在。 **Etag 在感知文件变化上比 Last-Modified 更加准确，优先级也更高。当 Etag 和 Last-Modified 同时存在时，以 Etag 为准。**

### MemoryCache 
这是浏览器最先尝试去命中的一种缓存，是内存中的缓存，速度最快，生命周期最低按，关闭tab后，进程结束，内存里数据不复存在。Base64几乎永远都可以被塞进内存，以及体积小的CSS文件和JS文件。大文件就甩进磁盘。

### Service Worker Cache

Service Worker 是一种独立于主线程之外的 Javascript 线程。它脱离于浏览器窗体，因此无法直接访问 DOM。这样独立的个性使得 Service Worker 的“个人行为”无法干扰页面的性能，这个“幕后工作者”可以帮我们实现离线缓存、消息推送和网络代理等功能。我们借助 Service worker 实现的离线缓存就称为 Service Worker Cache。

Service Worker 的生命周期包括 install、active
、working 三个阶段。一旦 Service Worker 被 install，它将始终存在，只会在 active 与 working 之间切换，除非我们主动终止它。这是它可以用来实现离线存储的重要先决条件。
PS：大家注意 Server Worker 对协议是有要求的，必须以 https 协议为前提。

### Push Cache

Push Cache 是指 HTTP2 在 server push 阶段存在的缓存。

* Push Cache 是缓存的最后一道防线。浏览器只有在 Memory Cache、HTTP Cache 和 Service Worker Cache 均未命中的情况下才会去询问 Push Cache。
* Push Cache 是一种存在于会话阶段的缓存，当 session 终止时，缓存也随之释放。
* 不同的页面只要共享了同一个 HTTP2 连接，那么它们就可以共享同一个 Push Cache。

## 存储篇2： 本地存储 —— 从Cookie 到 Web Storage、IndexDB

### Cookie
在web开发的早起，人们需要亟待解决的一个问题就是状态管理的问题： HTTP协议是一个无状态协议，浏览器向服务器发起一个请求，得到响应之后，故事就结束了，服务器不会记录关于
客户端的任何信息，如何才能在下次请求的时候，才能让服务器知道“我是我”呢？

Cookie在这样的背景下，应运而生。
cookie说白了就是一个小小的文本文件，每次请求的时候都需要携带，在客户端和服务器之间传来传去，它可以携带用户信息，当服务器检查 Cookie 的时候，便可以获取到客户端的状态，它是以
**键值对的形式存在**。

cookie的三个弱点
1. 太小，仅能存储4KB数据，超过就被裁切
2. cookie是紧跟域名的，同一个域名下的所有请求，都会携带Cookie。请求一张图片或一个CSS文件也要携带cookie，极大浪费带宽，存储的事不应该由cookie来承担，它本质是用于通讯的。

### Web Storage
Web Storage是HTML5提出用于解决客户端存储信息的技术
* 作用域：Local Storage、Session Storage 和 Cookie 都遵循同源策略。但 Session Storage 特别的一点在于，即便是相同域名下的两个页面，只要它们**不在同一个浏览器窗口中打开**，那么它们的 Session Storage 内容便无法共享。
* 优点
1. 存储量大 ，容量可达到5-10M之间
2. 仅存放在浏览器端，不会发生与服务器之间的通信
* 缺点
1. 只能存字符串，对象需要转JSON字符串
2. 存量有限，为解决存量问题需要搬出indexDB

### indexDB
indexDB是运行在浏览器的一个**非关系型数据库**，存量不会小于250MB。它不仅可以存储字符串，还可以存储二进制数据。
https://www.kancloud.cn/sllyli/performance/1242198

## 彩蛋篇：CDN 的缓存与回源机制解析

CDN （connection delivery network，即内容分发网络）指的是一组分布在各个地区的服务器。这些服务器存储着数据的副本，因此服务器可以根据哪些服务器与用户距离最近，来满足数据的请求。CDN提供快速服务，较少受高流量影响。

### CDN的核心功能
1. 缓存
2. 回源
缓存就是copy一份资源到另一个服务器的过程，而回源是指当服务器找不到缓存资源时，访问根服务器去请求

前端一般把**静态资源**存放在CDN。根服务器一般是业务服务器，它的核心任务在于生成动态页面或返回非纯静态页面，这两种过程都需要计算。
所谓“静态资源”，就是像 JS、CSS、图片等**不需要业务服务器进行计算即得的资源**。而“动态资源”，顾名思义是需要**后端实时动态生成的资源**，较为常见的就是 JSP、ASP 或者依赖服务端渲染得到的 HTML 页面。

## 渲染篇1： 服务端渲染的探索与实践

一般来说，在传统客户端渲染的模式下，服务端是把渲染需要的静态文件发送给客户端，客户端收到文件后需要在浏览器跑一遍JS，根据JS的运行结果生成相应的DOM。这种特性使得客户端的渲染的源代码总是特别简洁，如下
```<!doctype html>
<html>
  <head>
    <title>我是客户端渲染的页面</title>
  </head>
  <body>
    <div id='root'></div>
    <script src='index.js'></script>
  </body>
</html>
```
**页面上呈现的内容，你在 html 源文件里里找不到** —— 这正是它的特点。
而服务端渲染是在服务端跑了js代码，把生成组件或页面渲染成HTML字符串再返回给客户端，不需要在客户端再跑一次JS代码。**页面上呈现的内容，我们在 html 源文件里也能找到**
服务端渲染的好处首先是为了SEO,为了提高关键字在搜索引擎的排名；其次才是提高性能。服务端渲染解决了首屏加载过慢的问题，服务端承担了客户端渲染的任务，给用户直接呈现的页面，不需要在JS再运行渲染一遍。**服务端渲染本质上是本该浏览器做的事情，分担给服务器去做。**

## 渲染篇2： 知己知彼 —— 解锁浏览器背后的运行机制
浏览器渲染呈现整个网页这个过程，宛如一个黑盒。内核各部件协同工作。我们需要关心以下几个大模块：
* HTML解释器： 将HTML文档经过词法分析输出DOM树。
* CSS解释器： 解析CSS文档，生成样式规则。
* 图层布局计算模块： 布局计算每个对象的精确位置和大小。
* 视图绘制模块：进行具体节点的图像绘制，将像素渲染到屏幕上。
* Javascript引擎： 编译执行Javascript代码。

渲染过程：
浏览器根据HTML的节点生成DOM tree，然后把它与CSS解析生成(与DOM tree生成过程并行)的CSSOM tree相结合生成布局渲染树，然后浏览器以布局渲染树为蓝本，计算布局并绘制图像，最后生成所看到到页面。

**CSS解析是从右往左而不是从左往右**

我们至少可以总结出如下性能提升的方案：

* 避免使用通配符，只对需要用到的元素进行选择。

* 关注可以通过继承实现的属性，避免重复匹配重复定义。

* 少用标签选择器。如果可以，用类选择器替代。

* 减少嵌套。后代选择器的开销是最高的，因此我们应该尽量将选择器的深度降到最低（最高不要超过三层），尽可能使用类来关联每一个标签元素。

### CSS和JS文件对于浏览器渲染的阻塞
## CSS的阻塞
默认情况下，CSS 是阻塞的资源。浏览器在构建 CSSOM 的过程中，**不会渲染任何已处理的内容**。即便 DOM 已经解析完毕了，只要 CSSOM 不 OK，那么渲染这个事情就不 OK（这主要是为了避免没有 CSS 的 HTML 页面丑陋地“裸奔”在用户眼前）。
## JS的阻塞
浏览器渲染引擎和JS执行引擎是相互独立开来的，JS引擎优先级更高，遇到JS代码段会优先执行，因此会阻碍浏览器渲染，浏览器的控制权就被JS引擎夺取。  
JS的功能是**修改**页面,浏览器不清楚JS是否会对页面的结构进行修改造成影响，浏览器之所以让 JS 阻塞其它的活动，是因为它不知道 JS 会做什么改变，担心如果不阻止后续的操作，会造成混乱。但是我们是写 JS 的人，我们知道 JS 会做什么改变。假如我们可以确认一个 JS 文件的执行时机并不一定非要是此时此刻，我们就可以通过对它使用 defer 和 async 来避免不必要的阻塞，这里我们就引出了外部 JS 的三种加载方式。  

* 正常模式  
  ```<script src="index.js"></script>```  
* async模式  
  ```<script async src="index.js"></script>```  
  不会阻塞浏览器渲染，js的加载是异步的，加载完js文件立即执行js  
* defer模式  
  ```<script defer src="index.js"></script>```  
  defer 模式下，JS 的加载是异步的，执行是被推迟的。等整个文档解析完成、DOMContentLoaded 事件即将被触发时，被标记了 defer 的 JS 文件才会开始依次执行。  

从应用角度，如果js文件不依赖于DOM节点和其他js文件的执行，则使用async；如果js文件依赖于DOM节点的渲染和其他js文件的执行，则使用defer。  
通过审时度势地向 script 标签添加 async/defer，我们就可以告诉浏览器在等待脚本可用期间不阻止其它的工作，这样可以显著提升性能  
## 渲染篇3： 对症下药——DOM 优化原理与基本实践

### DOM为什么这么慢
js很快，Js的世界一切都是简单的。但是JS与DOM的交互不是属于JS的独舞，JS和浏览器引擎是两个不同的独立的引擎，相互的交互属于跨界交流，JS与DOM就像两个不同的岛屿，
交流需要过桥费。过桥费就需要开销，这笔开销不可忽略。所以每次JS对DOM的访问或者操作都会多一次过路费,性能开销就增大。所以，减少对DOM的操作在雅虎军规里并不是空穴来风。  

#### 对 DOM 的修改引发样式的更迭

过桥很慢，到了桥对岸，我们的更改操作带来的结果也很慢。  

很多时候，我们对 DOM 的操作都不会局限于访问，而是为了修改它。当我们对 DOM 的修改会引发它外观（样式）上的改变时，就会触发回流或重绘。  

这个过程本质上还是因为我们对 DOM 的修改触发了渲染树（Render Tree）的变化所导致的：  

* 回流：当我们对 DOM 的修改引发了 DOM 几何尺寸的变化（比如修改元素的宽、高或隐藏元素等）时，浏览器需要重新计算元素的几何属性（其他元素的几何属性和位置也会因此受到影响），然后再将计算的结果绘制出来。这个过程就是回流（也叫重排）。  

* 重绘：当我们对 DOM 的修改导致了样式的变化、却并未影响其几何属性（比如修改了颜色或背景色）时，浏览器不需重新计算元素的几何属性、直接为该元素绘制新的样式（跳过了上图所示的回流环节）。这个过程叫做重绘。
**回流必定重绘，重绘未必会回流**，两个都会消耗浏览器渲染性能，而回流消耗的性能比重绘更大，因此尽量减少两者的发生是提高性能的关键。
  
#### 药到病除：给你的 DOM “提提速”
* 减少DOM的访问与操作
使用变量缓存器DOM的节点信息，不要for循环多次访问，也不要对DOM进行多次操作，应该用DOMFragment缓存起对DOM的多次循环操作，最后在用创建的DOMFragment赋值给需要操作的DOM元素。例如
```
for(var count=0;count<10000;count++){ 
  document.getElementById('container').innerHTML+='<span>我是一个小测试</span>'
} 
```
优化为
```
let container = document.getElementById('container')
// 缓存需要操作的DOM节点信息，避免多次访问

// 创建一个DOM Fragment对象作为容器
let content = document.createDocumentFragment()
for(let count=0;count<10000;count++){
  // span此时可以通过DOM API去创建
  let oSpan = document.createElement("span")
  oSpan.innerHTML = '我是一个小测试'
  // 像操作真实DOM一样操作DOM Fragment对象
  content.appendChild(oSpan)
}
// 内容处理好了,最后再触发真实DOM的更改
container.appendChild(content)
```
## 渲染篇 4：千方百计——Event Loop 与异步更新策略

#### micro-task 和 macro-task
为了简化，标题称为微任务和宏任务，下文同。
事件循环中的异步队列主要有两种： 宏任务队列和微任务队列
宏任务包括： setTimeout、setInterval、 setImmediate、script标签（整体代码）、 I/O 操作、UI 渲染等。
微任务包括: process.nextTick, promise, mutationObserve

#### Event Loop过程解析
一个完整的事件循环过程包括以下几个阶段。
* 初始阶段： 调用栈为空，微任务队列为空，宏任务队列有且仅有一个script脚本（整体代码）  
* 全局上下文整体代码(script标签)被推入调用栈，同步执行，执行过程中调用的一些接口又会产生新的一些宏任务和微任务，并把它们推入各自的任务队列。同步代码执行完，script脚背则被移出宏任务队列，  
  这个**本质是宏任务执行和出队列的过程**  
* 然后执行微任务和微任务的出队操作，要注意的是，宏任务出队时是一个一个执行的，而微任务出队时是一队队执行的。因此处理微任务队列这一步时，是把微任务逐个执行并出队，直至把微任务队列清空。  
* **执行渲染操作，更新页面**  
* 检查是否存在web worker任务，如果有则对其进行处理。  
（上述过程循环往复，直到两个队列都清空）这就是完整的一个event loop过程  
#### 生产上的实践
vue中使用$nextTick来更新状态
而nextTick方法内部则是使用promise进行异步任务包装，也就是微任务。  

###渲染篇 5：最后一击——回流（Reflow）与重绘（Repaint）

#### 产生回流（重排）的操作  
* 最“贵”的操作：改变 DOM 元素的几何属性  
这个改变几乎可以说是“牵一发动全身”——当一个DOM元素的几何属性发生变化时，所有和它相关的节点（比如父子节点、兄弟节点等）的几何属性都需要进行重新计算，它会带来巨大的计算量。  

常见的几何属性有 width、height、padding、margin、left、top、border 等等。    

* “价格适中”的操作：改变 DOM 树的结构  
这里主要指的是节点的增减、移动等操作。浏览器引擎布局的过程，顺序上可以类比于树的前序遍历——它是一个从上到下、从左到右的过程。通常在这个过程中，当前元素不会再影响其前面已经遍历过的元素。  

* 最容易被忽略的操作：获取一些特定属性的值  
当你要用到像这样的属性：offsetTop、offsetLeft、 offsetWidth、offsetHeight、scrollTop、scrollLeft、scrollWidth、scrollHeight、clientTop、clientLeft、clientWidth、clientHeight 时，你就要注意了！  

“像这样”的属性，到底是像什么样？——这些值有一个共性，就是需要通过即时计算得到。因此浏览器为了获取这些值，也会进行回流。  

除此之外，当我们调用了 getComputedStyle 方法，或者 IE 里的 currentStyle 时，也会触发回流。原理是一样的，都为求一个“即时性”和“准确性”。    

### 如何规避触发回流和重绘  

了解触发回流和重绘的“导火索”，学会更聪明地使用他们  

#### 将“导火索”缓存起来，避免频繁访问  

上文提到使用访问特殊的几何属性时，会频繁触发回流和重绘 以下的代码：  
```
<script>
  // 获取el元素
  const el = document.getElementById('el')
  // 这里循环判定比较简单，实际中或许会拓展出比较复杂的判定需求
  for(let i=0;i<10;i++) {
      el.style.top  = el.offsetTop  + 10 + "px";
      el.style.left = el.offsetLeft + 10 + "px";
  }
  </script>
```
可以修改成   
```
// 缓存offsetLeft与offsetTop的值
const el = document.getElementById('el') 
let offLeft = el.offsetLeft, offTop = el.offsetTop

// 在JS层面进行计算
for(let i=0;i<10;i++) {
  offLeft += 10
  offTop  += 10
}

// 一次性将计算结果应用到DOM上
el.style.left = offLeft + "px"
el.style.top = offTop  + "px"

```

#### 避免逐条改变样式，使用类名去合并样式  

下面类似的代码
```
const container = document.getElementById('container')
container.style.width = '100px'
container.style.height = '200px'
container.style.border = '10px solid red'
container.style.color = 'red'
```  
可以改成有一个class加持的样子  

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Document</title>
  <style>
    .basic_style {
      width: 100px;
      height: 200px;
      border: 10px solid red;
      color: red;
    }
  </style>
</head>
<body>
  <div id="container"></div>
  <script>
  const container = document.getElementById('container')
  container.classList.add('basic_style')
  </script>
</body>
</html>

```  
前者每次单独操作，都去触发一次渲染树更改，从而导致相应的回流与重绘过程。  
合并之后，等于我们将所有的更改一次性发出，用一个 style 请求解决掉了。  

#### 将DOM离线化  
上文所述发生的回流与重绘，前提都是DOM存在于文档流中才会产生开销， 当把DOM元素从页面结构中拿掉时，后续对元素进行的样式的变化将不会产生回流和重绘——这个将元素“拿掉”的操作，就叫做 DOM 离线化。  

在修改元素的其他样式前，先把DOM元素样式属性设置为display:none;  
但是需要注意性价比，因为把DOM元素从文档流拿掉和再显现出来的操作都会触发回流和重绘，如果只需要很少地改变其他样式的话，离线化的操作就没有多少性价比。  

### Flush 队列：浏览器并没有那么简单

一个问题，以下代码，在Chrome浏览器中一共会触发多少次回流和重绘？  
```
let container = document.getElementById('container')
container.style.width = '100px'
container.style.height = '200px'
container.style.border = '10px solid red'
container.style.color = 'red'
```
一般来说，根据上面的知识，一共触发三次回流，一次重绘。但是打开chromeF12面板，看到其实只触发了一次回流和一次重绘。  
因为浏览器很聪明，他会把这些消耗性能的操作缓存到flush队列中，把我们触发的回流与重绘任务都塞进去，待到队列里的任务多起来、或者达到了一定的时间间隔，或者“不得已”的时候，再将这些任务一口气出队。  
注意这个不得已，上文有提到一些特殊的，计算实时几何信息的就会触发浏览器这个“不得已”。

补充： 并不是所有的浏览器都跟chrome一样聪明，所以该优化性能时都需要优化性能，让所有浏览器看起来保持一致。  

## 性能监测篇：Performance、LightHouse 与性能 API  

平时我们比较推崇的性能监测方案主要有两种：**可视化方案**、**可编程方案**。

### 可视化监测：从 Performance 面板说起

 Chrome浏览器F12打开即可看到Performance面板，
当我们选中图中所标示的实心圆按钮，Performance 会开始帮我们记录我们后续的交互操作；当我们选中圆箭头按钮，Performance 会将页面重新加载，计算加载过程中的性能表现。  
tips：使用 Performance 工具时，为了规避其它 Chrome 插件对页面的性能影响，我们最好在无痕模式下打开页面  

### 可视化监测： 更加聪明的 LightHouse  
Performance 无疑可以为我们提供很多有价值的信息，但它的展示作用大于分析作用。它要求使用者对工具本身及其所展示的信息有充分的理解，能够将晦涩的数据“翻译”成具体的性能问题。  

程序员们许了个愿：如果工具能帮助我们把页面的问题也分析出来就好了！上帝听到了这个愿望，于是给了我们 LightHouse：  

`Lighthouse 是一个开源的自动化工具，用于改进网络应用的质量。 你可以将其作为一个 Chrome 扩展程序运行，或从命令行运行。   
为Lighthouse 提供一个需要审查的网址，它将针对此页面运行一连串的测试，然后生成一个有关页面性能的报告。`  
* 首先在 Chrome 的应用商店里下载一个 LightHouse。   

* 除了直接下载，我们还可以通过命令行使用 LightHouse：  

```
npm install -g lighthouse
lighthouse https://juejin.im/books
```  
同样可以得到掘金小册的性能报告。  

* 此外，从 Chrome 60 开始，DevTools 中直接加入了基于 LightHouse 的 Audits 面板：  

LightHouse 因此变得更加触手可及了，这一操作也足以证明 Chrome 团队对 LightHouse 的推崇。  


### 可编程的性能上报方案： W3C 性能 API

W3C 规范为我们提供了 Performance 相关的接口。它允许我们获取到用户访问一个页面的每个阶段的精确时间，从而对性能进行分析。我们可以将其理解为 Performance 面板的进一步细化与可编程化。

主要是performance对象,控制台输入window.performance能看到其下相关属性。

下面给出一些计算时间的例子
```
const timing = window.performance.timing
// DNS查询耗时
timing.domainLookupEnd - timing.domainLookupStart
  
// TCP连接耗时
timing.connectEnd - timing.connectStart
 
// 内容加载耗时
timing.responseEnd - timing.requestStart

```

以下是关键性指标：
```
// firstbyte：首包时间	
timing.responseStart – timing.domainLookupStart	

// fpt：First Paint Time, 首次渲染时间 / 白屏时间
timing.responseEnd – timing.fetchStart

// tti：Time to Interact，首次可交互时间	
timing.domInteractive – timing.fetchStart

// ready：HTML 加载完成时间，即 DOM 就位的时间
timing.domContentLoaded – timing.fetchStart

// load：页面完全加载时间
timing.loadEventStart – timing.fetchStart
```

此外，通过访问 performance 的 memory 属性，我们还可以获取到内存占用相关的数据

本课完结与2023年10月13日
