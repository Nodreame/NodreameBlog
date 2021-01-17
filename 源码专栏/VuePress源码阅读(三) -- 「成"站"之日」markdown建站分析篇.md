# VuePress源码阅读(三) -- 「成"站"之日」markdown建站分析篇

![20200915090022c2dd2276a857106a69e80a8b816484d8.jpg.h700](http://img.nodreame.cn/20200915090022c2dd2276a857106a69e80a8b816484d8.jpg.h700.jpg)

> 前置文章：
> [VuePress源码阅读(一)--初探 VuePress](https://juejin.cn/post/6917643530588389390)
> [VuePress源码阅读(二)--dev和build的执行流程分析](https://juejin.cn/post/6918015439922528269)

之前写了两篇文章来记录对 VuePress  dev 和 build 执行过程的源码阅读记录，不过那只是为了学习 VuePress 所做的一点小调研，目的是对 VuePress 的整个执行流程有个大概的把控.

而 VuePress 在我看来更重要的是通过整合一系列技术，实现了**仅靠 "markdown文件 + 配置" 快速构建网站**的效果.

在 VuePress 官方首页将 VuePress 定义为：

- 项目结构以 Markdown 为中心
- 是一个技术栈为"Vue + vue-router + Webpack"的静态网站生成器
- 会为每个页面预渲染生成静态 HTML，被加载后作为 SPA 运行（Vue SSR 赋能）
- 想要构建一个最简网站，甚至不用配置文件，只需要一个 README.md 就能成站

前面两篇Vuepress源码分析的文章已经把整个 dev 和 build 的流程走过一遍了，现在我们对整个流程做一次整理：

![VuePress流程](http://img.nodreame.cn/VuePress%E6%B5%81%E7%A8%8B.jpg)

现在来思考一下整个流程中，VuePress 如何整合技术来实现 markdown 文件的建站.

## 一. 寻找编译 markdown 文件的库

markdown 是一种可用于在纯文本文档中添加格式化元素标记语言，故使用合适的转换工具将 markdown 文件编译成 HTML 文件，即可完成 Markdown 文件作为网站的展示.

现在看看 VuePress 是使用什么库来完成 markdown 文件编译为 HTML 文件的.

首先来到共享流程中的 app.process 阶段（node_modules/@vuepress/core/lib/node/App.js）：

![截屏2021-01-16_下午7_45_58](http://img.nodreame.cn/%E6%88%AA%E5%B1%8F2021-01-16_%E4%B8%8B%E5%8D%887_45_58.png)

可以看到这里通过 createMarkdown 函数获取到了 markdown，然后我们先倒推看看 createMarkdown 来自哪里（node_modules/@vuepress/core/lib/node/createMarkdown.js）：

![image-20210116195200370](http://img.nodreame.cn/image-20210116195200370.png)

这里的 siteConfig 其实就是 config.js 的配置，这里其实就是在 @vuepress/markdown 提供的 createMarkdown 前包装了一层，目的就是给 markdownConfig 配置添加两个属性.

我们继续追溯 @vuepress/markdown，其内部使用了 [markdown-it-chain](markdown-it-chain) 实现对 npm 包 [markdown-it](https://github.com/markdown-it/markdown-it) 的内置插件管理，我们可以通过```require('@vuepress/markdown')``` 来查看或者改变内部插件的启用停用，最后使用 ```const md = config.toMd(require('markdown-it'), markdown)``` 来返回一个 markdown-it 对象（就是变量 md）.

那么什么是 [markdown-it](https://github.com/markdown-it/markdown-it) 呢？其实它是一个 markdown 的解析库，它能解析 markdown 文档，并将其渲染为 HTML 文档，官方 Demo 如下：

![image-20210116202447238](http://img.nodreame.cn/image-20210116202447238.png)

其用法简单来说就是使用一组配置来获取 markdown-it 提供的对象，这一步就是刚刚 markdown-it-chain 做的事情，然后用这个对象调用 render 即可~ 详细用法可见 <https://markdown-it.github.io/markdown-it/>

现在我们就已经找到了编译 markdown 文件的库了，接下来回头看看后续流程中是如何使用获得的 markdown-it 对象的.

## 二. markdown-it 对象的使用

回忆前面对于 VuePress dev 和 build 流程的分析（文章：[VuePress源码阅读(二)--dev和build的执行流程分析](https://juejin.cn/post/6918015439922528269)），在 dev 和 build 的 process 函数中都会经历一个构建 Webpack 配置的阶段 prepareWebpackConfig，但是 dev 和 build 的 webpack 构建流程略有不同（左侧为 dev ，右侧为 build）：

![image-20210116205430534](http://img.nodreame.cn/image-20210116205430534.png)

可以看到相同的是都调用了 "createClientConfig（即「客户端配置构建」）"，不同的是：

- dev 多了三个插件的引用
- build 多了一个"createServerConfig（即「服务器配置构建」）"，用于生成服务端渲染程序

无论是 createClientConfig还是createServerConfig 内部都调用了 createBaseConfig（即「基础配置构建」），之所以能够通过一个对象config来控制Webpack的配置，就是因为 config 对象是在 createBaseConfig 中通过 [webpack-chain](https://github.com/neutrinojs/webpack-chain) 构建，这是一个通过链式 API 来构建 webpack 配置的库，其生产的 config 对象就跟项目中的 webpack.config.js 一致，所以也可以把上面的 base、client、server 和平日的 webpack 配置文件联系起来~

createBaseConfig 全流程比较长就不截图了，大概就是把平时 webpack.config.js 中的配置写了一遍，这里只截一下在一堆 loader 中的 markdown-loader 的配置，这里显示它接收 ```.md``` 后缀文件然后使用 markdown-loader 处理：

![image-20210116211841930](http://img.nodreame.cn/image-20210116211841930.png)

这个 markdown-loader 也是 @vuepress 自带的，我们在里面找到了使用 markdown-it 对象调用 render函数生成 html 的代码：

![image-20210116212943752](http://img.nodreame.cn/image-20210116212943752.png)

ok，这样 markdown 文件"成站"的整个流程已经打通了~

> 欢迎拍砖，觉得还行也欢迎点赞收藏~
> 新开公号：「无梦的冒险谭」欢迎关注（搜索 Nodreame 也可以~）
> 旅程正在继续 ✿✿ヽ(°▽°)ノ✿
