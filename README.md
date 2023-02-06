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
