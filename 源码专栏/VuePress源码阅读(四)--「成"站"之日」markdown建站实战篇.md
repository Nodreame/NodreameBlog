# VuePress源码阅读(四) -- 「成"站"之日」markdown建站实战篇

![d0290e21c28dc3a047ce9121a5f21ae4c057f905.jpg@1320w_760h](http://img.nodreame.cn/d0290e21c28dc3a047ce9121a5f21ae4c057f905.jpg@1320w_760h.webp)

系列文章：

- [VuePress源码阅读(一)--初探 VuePress](https://juejin.cn/post/6917643530588389390)
- [VuePress源码阅读(二)--dev和build的执行流程分析](https://juejin.cn/post/6918015439922528269)
- [VuePress源码阅读(三) -- 「成"站"之日」markdown建站分析篇](https://juejin.cn/post/6918382886202802190)

既然在上篇文章完成了对 markdown建站的分析，那么肯定要来个实战了，话不多说马上开始.

## 一、用 markdown 文件实现最简建站

这次实战的目标是使用一个 README.md 文件通过 Webpack 构建网站，支持简单的预览和打包.

``` bash
mkdir vuepress-study
cd vuepress-study
mkdir docs
echo '# Hello VuePress' > docs/README.md
touch webpack.config.js
# 安装依赖
yarn init -y
yarn add -D webpack webpack-cli markdown-it html-webpack-plugin clean-webpack-plugin html-loader
```

这里先通过 webpack.config.js 配置来编写，之后再用 webpack-chain 实现一遍.

使用 markdown-it 的 demo 作为 README.md 的内容方便测试 markdown-it 的效果，链接为：<https://markdown-it.github.io/>

由于现在只有一个 markdown 文件（docs/README.md），所以作为 Webpack 入口的 js 和 markdown-loader 都需要由我们自行提供：

``` bash
mkdir pkg
mkdir pkg/client
touch pkg/client/clientEntry.js
mkdir pkg/markdown-loader
touch pkg/markdown-loader/index.js
```

将 Webpack 官网例子改成引入 README.md 就形成了入口文件 clientEntry.js ，目标是网站能够将 markdown 转换结果添加到页面上：

``` js
import mdHTML from '../../docs/README.md'

function component() {
  var element = document.createElement('div');
  console.log(typeof mdHTML)
  element.innerHTML = mdHTML
  return element;
}

document.body.appendChild(component());
```

接着简单地引入 markdown-it 编写出 markdown-loader：

``` js
'use strict'

const md = require('markdown-it')

module.exports = function (content) {
  const markdown = md()
  const html= markdown.render(content)
  return html
}
```

最后编写 webpack.config.js 如下：

``` js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');

module.exports = {
  mode: 'development',
  entry: path.resolve(__dirname, 'pkg/client/clientEntry.js'),
  output: {
    filename: 'app.js',
    path: path.resolve(__dirname, 'dist'),
  },
  module: {
    rules: [
      { 
        test: /\.md$/,
        use: [
          {
            loader: 'html-loader',
          },
          {
            // 使用项目中自带的 loader
            loader: require.resolve('./pkg/loader/markdown-loader'),
          }
        ],
      },
    ],
  },
  plugins: [
    new CleanWebpackPlugin(),
    new HtmlWebpackPlugin(),
  ],
}
```

执行打包命令 ```npx webpack``` 完成打包：

![](http://img.nodreame.cn/image-20210117182520391.png)

用命令行查看打包结果，并使用 http-server 运行本地服务器：

![](http://img.nodreame.cn/image-20210117182731257.png)

OK，除了一些 emoji 没显示图像，大部分布局基本一致.

![](http://img.nodreame.cn/image-20210117182918098.png)

这部分代码已经放到 Github 上了有兴趣可以看看：

- 链接：<https://github.com/Nodreame/vuepress-study>
- 提交：b3a7015abb44b042dc80892c43525f23d921f092

## 二、整合 Vue 构建网站

### 1. 使用 Webpack + Vue 构建网站

上面只是个纯静态的最简网站搭建，而 VuePress 是基于 Vue 构建的，那么现在我们也来模拟这个流程.

首先是跑起在当前项目加入 vue，然后创建Vue实例并完成相应打包.

所以修改入口文件 pkg/client/clientEntry.js 为：

``` js
import { createApp } from './app'

const { app } = createApp()

app.$mount('#app')
```

其中的 createApp 工厂函数就是用来创建新的 Vue 实例的，所以编写 pkg/client/app.js 如下：

``` js
import Vue from 'vue'
import App from './App.vue'

// 导出一个工厂函数，用于创建新实例
export function createApp () {
  const app = new Vue({
    // 根实例简单的渲染应用程序组件。
    render: h => h(App)
  })
  return { app }
}
```

创建 App.vue 如下：

``` vue
<template>
    <div id="vueapp">{{ msg }}</div>
</template>
<script>
    export default {
        data() {
            return {
                msg: 'Hello world!'
            }
        }
    }
</script>
```

并且准备好 index.html 以供 html-webpack-plugin 使用：

``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <title></title>
  </head>
  <body>
    <div id="app"></div>
  </body>
</html>
```

接着安装对应依赖，其中 vue-loader 和 vue-template-compiler 需要一起安装，保证处于同个版本：

``` bash
yarn add vue
yarn add -D vue-loader vue-template-compiler
```

最后修改 webpack.config.js 如下：

![](http://img.nodreame.cn/image-20210117205553060.png)

接着运行 ```npx webpack``` 完成打包，可以看到页面已经成功显示出 App.vue 的内容：

![](http://img.nodreame.cn/image-20210117205839278.png)

这部分代码已经放到 Github 上了有兴趣可以看看：

- 链接：<https://github.com/Nodreame/vuepress-study>
- 提交：778e893dda1808434ff97340217d12638ca04550

### 2. 将 markdown 编译结果插入 Vue 网站

上面我们已经将 Vue 整合到项目中并成功地完成了构建，但是由于入口文件 pkg/client/clientEntry.js 的逻辑变化导致网站并没有引用 markdown 的打包结果.

因为现在还没有加入 vue-router 来做路由管理，所以暂时使用比较原始的方法做过渡 -- 直接在 App.vue 做引入（由于直接返回字符串所以用 v-html 实现展示）：

![](http://img.nodreame.cn/image-20210117212321731.png)

效果如下：

![](http://img.nodreame.cn/image-20210117212456633.png)

OK，至此我们已经使用 markdown 文件成功地创建了一个Vue 驱动的 SPA 网站，并且实现客户端渲染(CSR)方式的打包.

## 三. 实现服务端渲染(SSR)方式打包

回忆一下前几篇文章里面 VuePress 打包都是构建为 SSR 形式的客户端激活程序+服务端渲染程序，再运行服务端渲染程序生成SSR HTML，这里我们也来实践一下.

首先完成依赖的安装：

``` bash
yarn add vue vue-server-renderer
yarn add -D webpack-node-externals
```

构建 SSR 的客户端激活部分的入口文件依旧使用 client/clientEntry.js 即可，而服务端渲染程序则需要替换为 client/serverEntry.js：

``` js
import { createApp } from './app'

export default context => {
  const { app } = createApp()
  return app
}
```

由于CSR 和 SSR 的配置并不相同，所以这里将公共配置提取到 webpack.base.config.js ，配置文件 webpack.ssrclient.config.js 和 webpack.ssrserver.config.js 合并公共配置，然后用来处理 SSR 打包,，以此来生成 client.json 和 server.json.

### 1. 提取公共配置文件

webpack.base.config.js 的配置如下，主要是提取了 loader 和 VueLoaderPlugin：

``` js
const VueLoaderPlugin = require('vue-loader/lib/plugin')

module.exports = {
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
      },
      { 
        test: /\.md$/,
        use: [
          {
            loader: 'html-loader',
          },
          {
            // 使用项目中自带的 loader
            loader: require.resolve('./pkg/loader/markdown-loader'),
          }
        ],
      },
    ],
  },
  plugins: [
    new VueLoaderPlugin(),
  ]
}
```

### 2. SSR客户端程序打包

webpack.ssrclient.config.js 的配置如下，这是参考 [Vue SSR -- 构建配置-客户端配置](https://ssr.vuejs.org/zh/guide/build-config.html#%E5%AE%A2%E6%88%B7%E7%AB%AF%E9%85%8D%E7%BD%AE-client-config) 写的，但是由于 webpack4 后期已经取消掉 webpack.optimize.CommonsChunkPlugin，所以这里我使用 splitChunks 来替代：

``` js
const merge = require('webpack-merge')
const baseConfig = require('./webpack.base.config.js')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')

module.exports = merge(baseConfig, {
  mode: 'production',
  entry: './pkg/client/clientEntry.js',
  plugins: [
    // new webpack.optimize.CommonsChunkPlugin({
    //   name: "manifest",
    //   minChunks: Infinity
    // }),
    // 此插件在输出目录中生成 `vue-ssr-client-manifest.json`。
    new VueSSRClientPlugin()
  ],
  optimization: {
    splitChunks: {
      name: 'manifest',
      minChunks: Infinity
    }
  },
})
```

运行 npx webpack --config webpack.ssrclient.config.js 进行打包：

![](http://img.nodreame.cn/image-20210117225750010.png)

结果生成了 *main.js* 和对应的 *vue-ssr-client-manifest.json*:

![](http://img.nodreame.cn/image-20210117225656303.png)

### 3. SSR服务端程序打包

webpack.ssrserver.js 的配置如下：

```js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
const baseConfig = require('./webpack.base.config.js')
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')

module.exports = merge(baseConfig, {
  mode: 'production',
  entry: './pkg/client/serverEntry.js',
  target: 'node',
  devtool: 'source-map',
  output: {
    libraryTarget: 'commonjs2',
  },
  externals: nodeExternals({
    allowlist: /\.css$/
  }),
  // 这是将服务器的整个输出
  // 构建为单个 JSON 文件的插件。
  // 默认文件名为 `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```

第一次运行的结果：

![](http://img.nodreame.cn/image-20210117231427323.png)

检查我的配置编写应该没有问题之后，想起 VuePress 用的 Webpack4，而我用的是 Webpack5 所以可能会出一些问题，所以网上搜索了一下找到了解决方法 <https://github.com/vuejs/vue/issues/11718>，为了方便大家查看我就把重点截一下图：

![](http://img.nodreame.cn/image-20210117233227867.png)

刚好一个人问了这个问题然后上面这个老哥回复了解决方案，需要修改 node_modules/vue-server-renderer/server-plugin.js 的代码，当然这不是太工程化的处理方案，但现在暂时也只能这样解决了~（提出解决方案的老哥也提出 Webpack4 现在暂时是得到更广泛的支持的，如果想少遇点问题可以暂时切回 Webpack4）.

修改之后的运行结果：

![](http://img.nodreame.cn/image-20210117233242604.png)

现在打包结果目录的情况如下：

![](http://img.nodreame.cn/image-20210117233350998.png)

### 4. 执行服务端渲染

上面我们已经成功获取到了两个json 文件，那么现在我们来编写服务端渲染程序，生成一个SSR HTML.

创建 pkg/client/index.ssr.html 如下：

``` html
<!DOCTYPE html>
<html lang="{{ lang }}">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <title>{{ title }}</title>
    <meta name="generator" content="VuePress {{ version }}">
  </head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

然后编写渲染程序 pkg/client/renderSSRHTML.js：

``` js
const fs = require('fs')
const path = require('path')
const { createBundleRenderer } = require('vue-server-renderer')

const serverBundle = require(path.resolve('../../dist/vue-ssr-server-bundle.json'))
const clientManifest = require(path.resolve('../../dist/vue-ssr-client-manifest.json'))

const ssrHTML = `<!DOCTYPE html>
<html lang="{{ lang }}">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <title>{{ title }}</title>
    <meta name="generator" content="VuePress {{ version }}">
  </head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
`

async function main () {
  // create server renderer using built manifests
  const renderer = createBundleRenderer(serverBundle, {
    clientManifest,
    runInNewContext: false,
    inject: false,
    shouldPrefetch: () => true,
    template: ssrHTML
  })

  const context = {
    title: 'VuePress',
    lang: 'en',
    version: '1.0'
  }

  let html = await renderer.renderToString(context)
  fs.writeFile('../../dist/index.html', html, () => { console.log('finish') })
}

main()
```

执行服务端渲染程序：

![](http://img.nodreame.cn/image-20210118001958783.png)

生成 index.html 成功：

![](http://img.nodreame.cn/image-20210118002026722.png)

来看看效果，控制台中又看到了熟悉的 data-server-rendered 了~

![](http://img.nodreame.cn/image-20210118002155478.png)

至此服务端渲染(SSR)方式打包渲染的方式已经基本实现，markdown经过层层处理终于成为一个网站了~

这部分代码已经放到 Github 上了有兴趣可以看看：

- 链接：<https://github.com/Nodreame/vuepress-study>
- 提交：37efffa2b580842e0a2342ec81d18b957172dab6

> 欢迎拍砖，觉得还行也欢迎点赞收藏~
> 新开公号：「无梦的冒险谭」欢迎关注（搜索 Nodreame 也可以~）
> 旅程正在继续 ✿✿ヽ(°▽°)ノ✿
