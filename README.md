# front_end_performance_optimization_juejin
### 掘金小册——前端性能优化与实践

本项目将是掘金小册—— 前端性能优化与实践 的学习，本文件将记录学习笔记
## 知识体系： 从一道面试题说起

# 在展开性能优化的话题之前，我想先抛出一个老生常谈的面试问题：

* 从输入 URL 到页面加载完成，发生了什么？

    
我们将这个过程切分为如下的过程片段：

1. DNS 解析
2. TCP 连接
3. HTTP 请求抛出
4. 服务端处理请求，HTTP 响应返回
5. 浏览器拿到响应数据，解析响应内容，把解析的结果展示给用户
大家谨记，我们任何一个用户端的产品，都需要把这 5 个过程滴水不漏地考虑到自己的性能优化方案内、反复权衡，从而打磨出用户满意的速度。

## 网络篇
1. webpack性能调优与Gzip

首先 ，根据引言的问题 ，我们先做网络部分的优化， 其中DNS和TCP连接前端无法改变太多，因此
我们主要从HTTP优化入手。
HTTP的优化主要分为两个部分
* 减少请求次数
* 减少单次请求所花费的时间

这两个优化无疑是指向我们日常开发的常见操作—— **资源的合并与压缩**
我们通常都是使用构建工具来实现这些需求，而业界巨无霸的构建工具无疑是webpack

# webpack的性能瓶颈

对于资源的压缩与合并，是老生常谈的问题，建议查看文档。
这里主要讲的是优化webpack,webpack性能调优的方向主要有两个：

1. webpack构建时间太长
2. webpack打包太大

# 构建过程提速策略

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
# 缓存之前构建过的js
使用文件夹限定的优化提升是有限的，所以我们还可以开启缓存将转译的结果
保存到文件系统，则至少可以将 babel-loader 的工作效率提升两倍。只需要
增加参数即可

将Babel编译过的文件缓存起来，下次只需要编译更改过的代码文件即可，这样可以大幅度加快打包时间。

`loader: 'babel-loader?cacheDirectory=true'`

讲完了loader，还需要放眼于plugins。
# 不要放过第三方库
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

# 使用Happypack——将 loader 由单进程转为多进程

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





