# VuePress源码阅读(二)--dev和build的执行流程分析

![006yt1Omgy1gj6rutgcpaj31210la4qp](http://img.nodreame.cn/006yt1Omgy1gj6rutgcpaj31210la4qp.jpg)

> 前置文章：[VuePress源码阅读(一)--初探 VuePress](https://juejin.cn/post/6917643530588389390)

上篇文章对 VuePress 的源码进行了简单分析，了解到：

- vuepress 包负责 CLI 命令注册及处理，@vuepress 包含着主要的逻辑处理部分
- 无论是 ```vuepress dev docs``` 还是  ```vuepress build docs``` 都会先执行 实例创建createApp 和 解析处理process 两个环节
- processs 环节可以分为五个阶段，分别是：
    - 1）配置解析 & 模板加载
    - 2）插件入队 & 初始化
    - 3）Markdown 相关处理
    - 4）源文件搜集
    - 5）插件执行，最终生成临时文件

但是接下来的阶段， ```vuepress dev docs``` 和 ```vuepress build docs``` 就开始走不通的处理路径了。先对两个最主要的执行流程进行分析有助于我们对整个项目有一个大体的了解，也有助于之后在此基础上分析实现细节.

## 一、dev 的执行过程

使用过 VuePress 制作网站的同学应该都见过运行 ```vuepress dev docs``` 的结果：

![image-20210115165436573](http://img.nodreame.cn/image-20210115165436573.png)

前面的 ```[wds]``` 表明了 VuePress 的 ```vuepress dev docs``` 内部就是使用了 [webpack-dev-server](https://v4.webpack.js.org/guides/development/#using-webpack-dev-server) 的能力来开启本地服务器并支持调试的~

为了验证这一点，我们来看看 ```vuepress dev docs``` 的逻辑：

![image-20210115170453911](http://img.nodreame.cn/image-20210115170453911.png)

可以看到通过 createApp 创建实例，并且等待资源处理过程 process 执行完成之后，开始进入 dev 的逻辑了：

![image-20210115170620743](http://img.nodreame.cn/image-20210115170620743.png)

进入 dev 之后执行按下面三个步骤执行：

- 第一步. 通过 DevProcess类构建一个实例 devProcess
- 第二步. 接下来用实例 devProcess 调用 process (不同于之前 App 类的 process)
- 第三步. 最后文件变化监听，并在指定端口启动一个本地服务器

第一步可以从构造器看出只是单纯的上下文赋值，而第二步的 process 就包含了比较多的内容了：

![image-20210115195207183](http://img.nodreame.cn/image-20210115195207183.png)

其中上面三个 watch 负责监听变化，中间的 setupDebugTip 是提供给用户方便调试的，最后在处理完端口和 host 之后就开始生成 Webpack 配置并放入实例的 webpackConfig 属性了.

由上可知，第二步主要做的是设置监听 & 构建Webpack配置，完成之后就可以进入第三步开启本地服务器了：

![image-20210115202728602](http://img.nodreame.cn/image-20210115202728602.png)

对于上面的写法可以参考官网的Demo链接：<https://www.webpackjs.com/guides/hot-module-replacement/#%E9%80%9A%E8%BF%87-node-js-api>

当在 NodeJS 中使用 webpack-dev-server 时，则应该通过 addDevServerEntrypoints(config, options 在代码中为其写配置，第一个参数config 即 Webpack 配置，第二个参数 options 则是我们平常在 webpack.config.js 中写给 devServer 字段的值对象. 剩余的编译和服务器启动就和官网的Demo基本一致无需赘述了.

到这里使用 ```vuepress dev docs``` 的执行过程已经简要地过一遍了：在通用的实例创建 createApp 和 解析处理process 两个环节之后进入 dev流程，设置观察文件变化 & 构建 Webpack 配置，最终通过 webpack-dev-server 启动一个本地服务器.

## 二、build 的执行过程

接下来让我们继续来看看 ```vuepress build docs``` 的执行过程. 首先我们看看执行 ```vuepress build docs``` 的结果：

![image-20210115221429580](http://img.nodreame.cn/image-20210115221429580.png)

可以看到在插件流程（app.process）之后，分别发生了Client 和 Server 两个编译记录，之后还进行了"静态HTML渲染"，最终生成静态资源到目标文件夹中.

之所以Client 和 Server 两个编译记录，按照基于 SSR 的理解应该是：

- 生成了一份服务端渲染代码，用于生成 HTML
- 生成了一份对应的客户端代码，用于**激活服务端生成的 HTML**，使其支持响应式

而后面进行的"静态HTML渲染"，基于 VuePress 通过固定的 markdown 文件生成静态网页这一特性，可以理解为由于网页全静态故无需每次都由服务器生成，直接打包时生成一份固定的 "服务端HTML" 即可.

为验证上面的说法故查看生成结果，可以看到左侧的 dist 目录下都是静态的 HTML 文件和资源，index.html 文件中也包含了 data-server-rendered="true" 这个 SSR 页面才具备的特殊属性，故论证了上面的观点~

![截屏2021-01-15_下午10_26_39](http://img.nodreame.cn/%E6%88%AA%E5%B1%8F2021-01-15_%E4%B8%8B%E5%8D%8810_26_39.png)

OK 上面从执行日志和生成结果倒推出了如下执行过程：

- 第一步. 编译服务端渲染程序 & 客户端激活程序
- 第二步. 执行服务端渲染程序 -> 生成服务端渲染的 HTML

那我们来看看相关的代码是怎么样的，我们找到 build 方法的位置（@vuepress/core/lib/node/App.js）：

![image-20210115223503431](http://img.nodreame.cn/image-20210115223503431.png)

可以看到基于 BuildProcess 创建实例之后确实是分成  process 和 render 两步执行的. 由于我们之前看过 dev 的源码，所以可以猜测 process 方法应该只是用于构建 Webpack配置和准备其他配置，具体执行的环节应该都放在 render 里面了，接下来我们一个个看~

### 1. process 分析

![image-20210115224119530](http://img.nodreame.cn/image-20210115224119530.png)

与 dev 相同的是都是用 resolveCacheLoaderOptions 方法做开始 & 用构建 Webpack 配置的 prepareWebpackConfig 方法做结尾，但是 build 这边中间少去了一堆监听和调试处理，只做了结果目录创建这一件事情.

### 2. render 分析

接下来进入 render 方法，简单浏览发现每个阶段都有简答的注释，大致和我们倒推出来的执行过程是一致的. 所以接下来以 ```logger.wait('Rendering static HTML...')``` 这一行打印为分界线，把程序分成两部分来看.

### 1）第一步. 编译服务端渲染程序 & 客户端激活程序

![image-20210115231006119](http://img.nodreame.cn/image-20210115231006119.png)

这里我们将 render 方法内除了 compile() 之外的代码都屏蔽掉，重新运行 ```vuepress build docs``` 之后，就可以看到生成了一个临时的 mainfest 文件夹，里面存放着

![image-20210115231226335](http://img.nodreame.cn/image-20210115231226335.png)

联想一下 Vue SSR 官方的流程图，没错这就是最右边的上下两个 Bundle：

![786a415a-5fee-11e6-9c11-45a2cfdf085c](http://img.nodreame.cn/786a415a-5fee-11e6-9c11-45a2cfdf085c.png)

接下来就是读取 bundle, 然后准备好创建一个 Bundle Render，以供后续步骤渲染 HTML 页面了（这一部分逻辑中重要部分我都给了序号，没给序号的部分可以暂时忽略）：

![image-20210115232636231](http://img.nodreame.cn/image-20210115232636231.png)

### 2）第二步. 执行服务端渲染程序 -> 生成服务端渲染的 HTML

参考 [Vue SSR官网 -- Bundle Renderer 指引](https://ssr.vuejs.org/zh/guide/bundle-renderer.html#%E4%BC%A0%E5%85%A5-bundlerenderer) 这一节，当 bundle render 已经创建完成，下一步想要生成静态页面，肯定就是调动 ```renderToString``` 函数了（`renderToString` 函数的能力是 **自动执行「由 bundle 创建的应用程序实例」所导出的函数（传入`上下文`作为参数），然后渲染它**）. 现在回到代码逻辑：

![image-20210115233331530](http://img.nodreame.cn/image-20210115233331530.png)

后半部分最终要的其实就只有 renderPage 的执行过程，renderPage 完成渲染后获得 HTML字符串，最终写入目标文件：

![image-20210115233516161](http://img.nodreame.cn/image-20210115233516161.png)

这样整个 build 的流程就结束了~

现在我们可以直接进入 doc/.vuepress/dist 目录执行 http-server 启动一个服务器，查看 build 出来的网页哟~

![image-20210115233934449](http://img.nodreame.cn/image-20210115233934449.png)

查看请求页面，虽然是用静态资源的方式启动的，不过依旧还是 SSR 的形式呢~

![image-20210115234001294](http://img.nodreame.cn/image-20210115234001294.png)

## 小结

VuePress 的 dev 和 build 两个流程前面部分都会先执行 实例创建createApp 和 解析处理process 两个环节，但是 dev 和 build 后半部分就不太一样了.

首先他们通过不同的条件判断获得不同的 Webpack 配置，然后：

- dev 基于 webpack-dev-server 创建本地调试服务器
- build 通过 webpack 读取 Client、Server两份配置打包出两个 bundle，再用 bundle render 提前完成 HTML 的生成，实现直接生成可用的 SSR HTML 的目的（具备 SEO 优化和首屏加载优化的特点）.

> 欢迎拍砖，觉得还行也欢迎点赞收藏~
> 新开公号：「无梦的冒险谭」欢迎关注（搜索 Nodreame 也可以~）
> 旅程正在继续 ✿✿ヽ(°▽°)ノ✿
