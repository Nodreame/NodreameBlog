# Webpack(一)

![image-20210120022523406](http://img.nodreame.cn/image-20210120022523406.png)

> 复习到 Webpack 部分顺便写一篇文章，我是这样将其知识点串起来的，希望对你也有帮助.

> Tip：本文时当前最新版本为 v5.15.0，不保证永久更新所以过时请勿服用谢谢

## 零、Webpack 是一个构建工具

Webpack 能实现帮助我们完成项目的打包，也叫构建，无论是写网站项目还是NodeJS 项目都用得上，三大框架的脚手架也大都是基于它实现的（emm...Vite 已经带着新的风暴出现了）

创建一个最简的项目如下：

![image-20210120030820932](http://img.nodreame.cn/image-20210120030820932.png)

为演示打包效果故项目引入了 lodash， 逻辑文件仅有一个使用了lodash功能的 index.js. 接着运行```npx webpack``` 完成打包，结果在 dist 目录下生成了一个看不懂的 main.js 文件：

![image-20210120031216595](http://img.nodreame.cn/image-20210120031216595.png)

初学时瞬间懵逼.jpg，比高数课上捡支笔就再也听不懂发生得还快...

![image-20210120031623271](http://img.nodreame.cn/image-20210120031623271.png)

OK让我们从这里开始学习 webpack，目标就是 **边学边填充上面这个流程图**.

从刚开始阅读 webpack官方文档时，文档就告诉了我们 webpack 的核心：

![image-20210120032436010](http://img.nodreame.cn/image-20210120032436010.png)

最后一个暂时忽略，这个是浏览器兼容相关的故暂时排除，剩下的五个核心概念我愿称它们为四大天王，所以接下来逐个讲解一下.

![small_201910271445172637](http://img.nodreame.cn/small_201910271445172637.jpg)

## 一、输入输出--Entry&Output

有输入就有输出是宇宙永恒的规律（貔貅直呼内行），Webpack 自然也不例外.

作为一个打包工具，Webpack 的基础能力就是 **读取项目文件 && 输出打包结果**，分别对应着核心概念中的 entry 和 output.

## 二、专家级打工人--Loaders

loader 的出现就是为了实现将某类型的文件转换为 JS 代码或 XX 资源（MD->HTML->JS，图片？）

### 1. 普通打工人

最基础的认知，loader 是个函数，输入待处理的内容，输出处理结果

### 2. 摸鱼打工人

async & call

## 三、见缝插针--插件Plugins

### 1. 插件的能力

插件的能力是赋能，给 webpack 带来原先不具备的新功能，例如通过 html-webpack-plugin 生成与打包文件匹配的 html，又例如 clean-webpack-plugin 能够在构建前清除旧的构建结果. 通常其使用方法就是往配置文件的插件数组里面塞入插件的实例.

## 2. 你知道「插件的执行顺序」吗？

上面聊 loader 的时候说到过，某类文件对应的loader数组，其执行顺序是倒序的，那么你知道「plugin 的执行顺序」吗？

与 loader 不同，plugin 的执行顺序和数组中 plugin 实例的顺序并无直接关系，与执行顺序相关的是 「webpack 构建过程的所有阶段」，webpack 作为一个"基于事件流的打包工具"，整个构建过程有许多 hook，这些 hook可以被插件所使用，在指定 hook 被触发的时候执行响应的插件.

例如 html-webpack-plugin 的能力是生成与构建结果 index.js 匹配的HTML文件，那么就要在 index.js 生成之后执行，查阅文档之后知道当「构建过程」完成后会触发 'emit hook'，所以 html-webpack-plugin 就是 emit hook 触发之后执行 HTML的生成逻辑的

我简单过了一遍文档和源码，画出下面的流程图仅供参考：

- [ ] 这里加入整个 Webpack 构建过程的流程图 with hook

又例如 clean-webpack-plugin 的能力是清除旧的构建结果，那么就应该在开始构建 ~ 构建完成之前执行即可，由上图可知是在 xx hook ~ xx hook 之间，查看 clean-webpack-plugin 的源码，恩果然是在这中间触发的

## 四、组合游戏--SourceMap

之所以把 SourceMap 这个官方没在「Core Concepts」中提到的概念放在这里，其作为一个开发中的重要概念为人所知，却不为人所熟知，Webpack 官方文档中一下子抛出了十几种可选项让初学者一脸懵圈，但掌握 SourceMap 对开发和部署后的调试都很有帮助，故将其列入四大天王之一（恩四天王有五个

Webpack 中的 SourceMap 十几个可选配置其实说到底是个组合数个基础能力的把戏而已，基础能力如下：

- eval
- cheap
- module
- nosources
- inline

## 五、隐蔽的角落--Mode

### 1. 零配置的 webpack 意味着什么

零配置的 webpack 就是和文章一开始的演示一样，不写配置文件直接运行 ```npx webpack```，也就是使用「默认配置」来运行项目构建.

想要了解「默认配置」本来是要看源码的，但是从演示和日志已经能够看出一些东西了~

- 默认的 entry 和 output 已经很明显了，分别是 src/index.js 和 dist/main.js
- 根据日志可以看到，当 mode 没被设置时，默认回退为 'production'

![截屏2021-01-20_上午4_31_55](http://img.nodreame.cn/%E6%88%AA%E5%B1%8F2021-01-20_%E4%B8%8A%E5%8D%884_31_55.png)

那么剩下没弄清楚的就是对应的 SourceMap、Loaders 和 Plugins 了，让我们继续往下看.

### 2. mode配置隐藏着什么

> 参考文档：[Mode](https://webpack.js.org/configuration/mode/)

mode 按照官方的说法，是用来告诉 Webpack 运行与其对应的内置优化的.

总所周知 mode 有三种模式：none、production、development

那么问题来了，「mode 三种模式」分别对应什么优化呢？

#### 1）不常用的 none

这里先说说不常用的 none，文档对于该模式的说法是「Opts out of any default optimization options」，意味着「无任何优化」.

这不代表着什么都不做，而是照常执行构建过程，但是**不做任何优化**.

下面看看我将 mode 设为 none 之后的构建结果，右侧结果代码中连 lodash 库的注释都放进来了，这就叫「无任何优化」.

![image-20210120045224088](http://img.nodreame.cn/image-20210120045224088.png)

TODO：未完

#### 2）终焉兵器 production

一般项目最终打包上线总是会用 production 模式进行打包，所以 production 自然意味着「仅使用 webpack 内置模块情况下最极致的优化」，那么来看看分别使用了什么优化手段.

- SourceMap：默认不启用，可以选 @拉钩

- optimization：moduleIds、chunkIds、mangleExports 全 deter（@webpack5）
- plugins：TerserPlugin、optimize.ModuleConcatenationPlugin、NoEmitOnErrorsPlugin

#### 3）开发之友 development

项目开发过程中为调试方便一般会选用 development 模式

- SourceMap：eval （方便文件映射）
- 缓存：开
- output：true
- 其他基本全关
