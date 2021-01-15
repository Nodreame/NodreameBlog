# VuePress源码阅读(一)--初探 VuePress

> 最近开发和写文章都用到了 VuePress 和 SSR，在深入学习的同时写点文章记录一下

## 一、最简起步

首先按照 VuePress 的指导创建一个最小的网站：

``` bash
mkdir vuepress-ssr
cd vuepress-ssr
yarn init -y
yarn add -D vuepress
mkdir docs
echo '# Hello VuePress' > docs/README.md
```

接着打开 packjge.json 并添加本地调试 & 打包的命令：

``` json
"scripts": {
  "docs:dev": "npx --node-arg=--inspect vuepress dev --debug docs",
  "docs:build": "npx --node-arg=--inspect vuepress build --debug docs"
}
```

ok 这样最简单的准备就搞定了，不过我们暂时不急着调试，可以先来看看代码.

## 二、CLI外壳 - vuepress

我们打开刚刚创建的项目的 node_modules，找到 vuepress 文件夹：

![image-20210114185426058](http://img.nodreame.cn/image-20210114185426058.png)

看到主逻辑文件只有 cli.js 和 index.js 两个，再看看 package.json：

![image-20210114185924797](http://img.nodreame.cn/image-20210114185924797.png)

可知我们平常使用的 ```vuepress``` 调用的其实就是 cli.js 文件. 再来看看 cli.js 干了什么：

![image-20210114200503449](http://img.nodreame.cn/image-20210114200503449.png)

> 这里的 cli 其实是npm包 [CAC](https://github.com/cacjs/cac) (命令行工具)返回的操作实例，这里用到command、option、action 三个方法，执行逻辑在 action 中

暂时忽略命令行相关的预处理和后续处理操作，这里的重点关注 registerCoreCommands 这个方法，上面脚本中的 ```vuepress dev docs``` 和 ```vuepress build docs``` 的逻辑就在这里的 action 中：

![image-20210114201402659](http://img.nodreame.cn/image-20210114201402659.png)

第 38 行开始就将运行方法作为参数放入 wrapcommand 准备执行，dev、build 都来自 node_modules/@vuepress 的 core 文件夹：

![image-20210114201607617](http://img.nodreame.cn/image-20210114201607617.png)

也就是说 node_modules/vuepress 其实只是个 CLI 壳子，而真正的 dev、build、eject 逻辑都在 node_modules/@vuepress 中！

接下来我们继续探究 @vuepress 内部的逻辑

### 三、@vuepress 探究

 现在通过 node_modules/@vuepress 的代码来进一步了解：

![image-20210114211314418](http://img.nodreame.cn/image-20210114211314418.png)

进入 @vuepress/core 的 index.js：

![image-20210114211430226](http://img.nodreame.cn/image-20210114211430226.png)

在这里能看到对 dev 和 build 的预处理相同，都是通过 createApp() 创建一个 app，然后对 app 进行 process 处理， 但是之后 dev 和 build 就开始进入不同逻辑了.

### 3.1 createApp

createApp 就在 index.js 中，内部返回了 new App(options) 的结果，App类的构造器如下，主要负责环境判断和解析目录参数：

![image-20210114212219741](http://img.nodreame.cn/image-20210114212219741.png)

当然 App类携带了很多很多的方法，主最要的 process、dev、build 当然是都包括在内的.

### 3.2 process

process 则包含比较多功能，经过一番阅读后将每个阶段做的事情描述如下：

![截屏2021-01-14_下午11_22_44](http://img.nodreame.cn/%E6%88%AA%E5%B1%8F2021-01-14_%E4%B8%8B%E5%8D%8811_22_44.png)

现在浏览器打开 <chrome://inspect/#devices>，然后运行 ```yarn docs:dev``` (直接运行 ```npx --node-arg=--inspect vuepress dev --debug docs``` 也行)，当 Remote Target 出现 Target 时 inspect 进到控制台：

![image-20210114224725966](http://img.nodreame.cn/image-20210114224725966.png)

进去之后应该什么都看不到，不慌，回刚刚代码的 process 函数里加上一行 debugger，然后重新运行 ```yarn docs:dev```，现在回到调控制台就可以按照自己的需要调试了：

![image-20210114224939498](http://img.nodreame.cn/image-20210114224939498.png)

当执行完 118 行的 ```await this.resolvePages()``` 之后就可以看到 pages 添加了一个 Page，也就是我们最简网站的 README 文件中包含的 Hello VuePress.

![image-20210114225759622](http://img.nodreame.cn/image-20210114225759622.png)

整个 process 函数走完之后，可以看到在 @vuepress/core 文件夹下生成了临时文件夹 .temp：

![image-20210114230232109](http://img.nodreame.cn/image-20210114230232109.png)

process 函数将项目资源 & 配置进行解析，然后通过加载&运行插件，最终生成了临时文件. ```vuepress dev docs``` 和 ```vuepress build docs``` 共享的逻辑就到此为止了.

接下来 ```vuepress dev docs``` 会通过引入 WebpackDevServer 来启动一个服务器， ```vuepress build docs``` 会根据 client & server 的不同配置执行 Webpack 打包，最后还会做静态HTML渲染，这些内容会放在下一篇文章中，敬请期待✿✿ヽ(°▽°)ノ✿

> 欢迎拍砖，觉得还行也欢迎点赞收藏~
> 新开公号：「无梦的冒险谭」欢迎关注（搜索 Nodreame 也可以~）
> 旅程正在继续 ✿✿ヽ(°▽°)ノ✿
