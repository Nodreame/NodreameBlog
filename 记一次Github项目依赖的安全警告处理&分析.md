# 记一次 Github 项目依赖的安全警告修复 & 分析

> 写这篇文章是觉得在解决问题的过程中 可以补全边缘知识 & 学习开源项目的方法。大家看看自己 Github 的项目如果有安全警的话可以参考本文思路一起练练手~

## 提要

**修复章节** 分别讲述了如何通过 **页面** & **命令行** 方式（git rebase）处理安全警告

**分析章节** 对 axios 本次版本升级（v0.21.0 => v0.21.1）所解决的 SSRF 问题进行分析，经历了：

- issue 分析 
- 本地重现 & 确定 Bug 原因
- 问题代码定位
- 修复代码编写 & 使用官方测试用例验证修复情况

## 零、起因

准备刷题准备面试，进到 [Github 刷题仓库](https://github.com/Nodreame/leetcode-js) 发现下面这个安全警告：

![](http://img.nodreame.cn/20210107170146.png)

翻译一下就是说有"潜在的安全漏洞"，打开来看看:

![](http://img.nodreame.cn/20210107163230.png)

居然是 axios 的漏洞修复小版本升级，作为上周刚刚阅读过 0.21.0 版本 axios 源码的我当然是要了解一下的啦（~~然后水篇文章~~

## 一、修复

可能文章的顺序有点奇怪，毕竟一般问题都是先"分析"后"修复"的。不过项目依赖的安全漏洞修复方法千篇一律，并且后面的分析过程一半以上可能只适合前端方向的同学看，所以就把"修复"环节前置了~

说说修复方法吧，一般就是手动修复完依赖版本提交到主分支或者当前分支嘛（请根据自己项目的发布流程而确定修改分支）。不过现在 Github 检测到 CVE（Common Vulnerabilities & Exposures”通用漏洞披露，本次是[CVE-2020-28168](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-28168)）后一般会基于主分支创建一个修复分支.

![](http://img.nodreame.cn/20210107171841.png)



既然官方已经帮我们改完了那就没必要自己搞了，直接 git rebase 一波杀穿（主分支做了提交限制 or 流程存在门禁就的就老老实实 merge 吧），可以通过网页或者命令行的方式来处理问题：

> P.S. 就算是我的分支之前刚做了提交也没问题，如下图所示

![](http://img.nodreame.cn/WechatIMG39.png)

### 1）网页操作方式

> Tip：如果对命令行的操作没有 200% 的信心真的真的请选择这个，规范易操作出错也有提示真心舒服

如果想在网页上解决的话就进入到项目的 Pull requests 标签页，确认无误之后选择合并方式：

![image-20210107174545267](http://img.nodreame.cn/image-20210107174545267.png)

这里我选择的是 Rebase adn merge，它会作为一次最新的提交合并入代码中（ 而且还可以直接删除掉Github 自动生成安全警告的分支，方便快捷），结果如下：

![image-20210107174858775](http://img.nodreame.cn/image-20210107174858775.png)

### 2）命令行方式

``` bash
git fetch # 拉项目最新信息
git pull origin master # 确保主分支最新
git checkout -b tmp origin/dependabot/npm_and_yarn/axios-0.21.1 # 把 "Github 自动修复分支"拉到本地 tmp 分支
git rebase master tmp # 变基操作：在 tmp 分支上执行，将 tmp 的更新内容尾接到 master 上，结果存储于 tmp 分支
git push origin tmp:master # 将 tmp 作为 master , 提交到远程仓库的 master 分支上
git push origin --delete dependabot/npm_and_yarn/axios-0.21.1  # 手动删除"Github 自动修复分支"
```

![image-20210107181414090](http://img.nodreame.cn/image-20210107181414090.png)

完成之后结果与网页方式是一致的，完成之后再进入到仓库页面，发现安全漏洞警告提示已经消失了（撒花✿✿ヽ(°▽°)ノ✿

![image-20210107182651308](http://img.nodreame.cn/image-20210107182651308.png)

## 二、分析

上面的 ~~Git 操作教学~~  **修复部分** 暂时就告一段落了，现在回过头来看看 axios 究竟是出了什么问题导致了安全警告，我们进入到 Pull request 标签页：

![image-20210107215126673](http://img.nodreame.cn/image-20210107215126673.png)



Release notes 展示了 v0.21.1 相对于 v0.21.0 发生的更新，其中 **Internal and Tests 部分** 都是测试用例的修复，与安全警告关系不大，所以让我们把重点放在 **Fixes and Functionality 部分**，第一项 ** Hotfix: Prevent SSRF** 就是对于安全警告的信息。

从后面的链接 [#3410](https://github-redirect.dependabot.com/axios/axios/issues/3410) 点进去，就能了解到这个修复的起因、讨论过程和处理方法。接下来我们就从最初始的 issue 来分析这个安全警告。

### 0. 前置知识：什么是「 跟随重定向(Follow Redirects) 」

> Tip：本来应该直接开始分析 issue 的，但是这个知识点可能会影响到对 issue 的分析阅读，所以就前置到这里来了

我先在本地8080启动一个返回 302 的服务器程序，大家脑补一下通过 **浏览器**、**Node.JS的内置http模块**、**Node.JS环境下的axios** 访问 [localhost:8080](localhost:8080) 链接分别会得到什么结果，程序如下：

``` js
const axios = require('axios')
const http = require('http')

http.createServer(function (req, res) { 
    res.writeHead(302, {location: 'http://example.com'})
    res.end()
}).listen(8080)
```

![AFewMomentsLater](http://img.nodreame.cn/AFewMomentsLater.jpeg)

现在揭晓答案：

- 浏览器：进入重定向提供的目标网站 http://example.com 

![image-20210108001016060](http://img.nodreame.cn/image-20210108001016060.png)  

基于 [MDN 的说法](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/HTTP_response_codes) ：

  > 302：该状态码表示所请求的URI资源路径临时改变,并且还可能继续改变.因此客户端在以后访问时还得继续使用该URI.新的URL会在响应的`Location:`头字段里找到.

加上观察网络请求的过程就能推导出 **浏览器** 的请求过程如下图所示，其中"请求过程 2"就是 **浏览器自动实现了「 跟随重定向(Follow Redirects) 」**：

![浏览器重定向过程](http://img.nodreame.cn/%E6%B5%8F%E8%A7%88%E5%99%A8%E9%87%8D%E5%AE%9A%E5%90%91%E8%BF%87%E7%A8%8B.png)

- Node.JS的内置http模块：得到 302 状态码，说明内置的http模块不具备 **「 跟随重定向(Follow Redirects) 」**能力，只是执行了第一个请求过程，也就是获取到 302 状态码及重定向地址即可：

  ![image-20210107235852668](http://img.nodreame.cn/image-20210107235852668.png)
  
- Node.JS环境下的axios：和**浏览器**的最终结果一样，状态码 200 并且能够获取到网页内容

  ![image-20210108014258352](http://img.nodreame.cn/image-20210108014258352.png)

那么为什么同样是 **Node.JS环境下**， **axios** 和 **内置HTTP模块** 对于重定向表现不同呢？在 [axios Github 文档](https://github.com/axios/axios) 中搜索 redirects ，可以再**请求配置** 小节找到这样的信息：

![image-20210108020451262](http://img.nodreame.cn/image-20210108020451262.png)

意思是 axios 默认开启了**「 跟随重定向(Follow Redirects) 」**能力（顺便考古到了2015 年是怎么给 axios 加上这个功能的 [follow redirects](https://github.com/axios/axios/pull/146/commits)），并且能够通过改变 ```maxRedirects``` 字段的值来决定 **「 跟随重定向(Follow Redirects) 」 **的开启或关闭。遇到 302 状态码时，如果开启了 **「 跟随重定向(Follow Redirects) 」 ** 就会获取重定向地址并继续跳转，如果不开启就是直接返回结果. 

现在把 ```maxRedirects``` 字段置为 0 ，再次运行程序，可以看到 axios 对于重定向的处理表现就和 NodeJS 一致了：

![image-20210108021423393](http://img.nodreame.cn/image-20210108021423393.png)

了解什么是 **「 跟随重定向(Follow Redirects) 」 ** 之后，我们来开始看看提 issue 的老哥遇到的问题~

### 1. 初始 issue  分析 

先看看这个安全警告的起因：

- 链接：[issue3369 - 重定向后的请求未通过代理传递](https://github.com/axios/axios/issues/3369)
- 说明：提 issue 的老哥提出他使用 "携带代理配置" 的 axios 进行访问，由于请求经过代理服务器，且代理服务器永远返回 302，所以老哥期待的运行现象是：
  - 1）首次请求由于配置了带来，故经过代理服务器地址 [localhost:8080](localhost:8080)，获得 302状态码，由于 axios 默认开启 **「 跟随重定向(Follow Redirects) 」 ** 所以暂时不打印结果，并且 axios 会自动尝试访问重定向后的目标地址 http://example.com；
  - 2）但是由于代理配置的存在，第二次访问还是会访问到  [localhost:8080](localhost:8080) ，再次获得 302 状态码；
  - 3）重复上面两个步骤直到到达 axios 的 Follow Redirects 模式次数上限，然后报错；


那么我们现在来看看提 issue 的老哥提供的完整重现代码：

``` js
const axios = require('axios')
const http = require('http')

const PROXY_PORT = 8080
let count = 0

// A fake proxy server
http.createServer(function (req, res) {
  	count++ // （我加入的计数逻辑）累加请求次数
    res.writeHead(302, {location: 'http://example.com'})
    res.end()
  }).listen(PROXY_PORT)

// 和我们的前置案例相比新增了一个"携带代理配置的 axios 请求"
axios({
  method: "get",
  url: "http://www.google.com/", // 随便写个链接，都会被代理对象取代
  proxy: { // 重点部分
    host: "localhost",
    port: PROXY_PORT,
  },
})
.then((r) => {
  console.log(count)   // (我加入的打印) 打印成功情况访问代理服务器次数
  console.log(r.status) // (我加入的打印) 用于观察返回状态码
  console.log(r.data)
})
.catch(e => {
  console.log(count) // 打印失败情况访问代理服务器次数
  console.error(e)
})
```

与我们期望的得到 302 报错结果不同，使用 axios v0.21.0 的执行效果如下：

![image-20210108171230057](http://img.nodreame.cn/image-20210108171230057.png)

现象为：**进入代理服务器一次，然后 axios 的请求结果状态码是 200，内容是 http://example.com 的页面内容**. 

根据现象推导执行过程为：首次请求成功进入了代理服务器并且累加了一次计数（且只有这一次），但是步骤 2请求却绕过了 axios 配置里的代理 proxy 直接访问了重定向地址  http://example.com. 

那么现在问题就很明显了，当获得 302 返回时应该 **携带代理配置** 重新发起请求，然而 axios v0.21.0 在重新发起请求时却丢失了**代理配置**，所以要做的事情就是研究下 axios **「 跟随重定向(Follow Redirects) 」 ** 之后为何丢失了**代理配置**.

### 2. 问题定位 & 修复方案制定

出现问题的是 NodeJS 环境，那么自然要找到 axios 源码中的 NodeJS 重定向请求配置的位置。

来到 axios 项目的 /lib/default.js 位置，下面的代码赋予了 axios 实现**跨平台网络请求能力**，它会自动判断运行平台并使用不同平台逻辑实现网络请求：

``` js
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') {
    adapter = require('./adapters/xhr'); // 浏览器环境走这里
  } else if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') {
    adapter = require('./adapters/http'); // Node.JS 环境走这里
  }
  return adapter;
}
```

接下来进入到 /lib/adapters/http.js 后发现只有一个函数，接收 config 参数并返回一个 Promise：

![image-20210109183012339](http://img.nodreame.cn/image-20210109183012339.png)

请求逻辑实现应该都在这个函数里面了，接着直接搜索 config.proxy 查找 **代理逻辑**，找到相关代码并得到如下分析：

![截屏2021-01-08_下午5_46_43](http://img.nodreame.cn/%E6%88%AA%E5%B1%8F2021-01-08_%E4%B8%8B%E5%8D%885_46_43.png)

这里的 171 行的 httpFollow 来自 [follow-redirects 模块](https://github.com/follow-redirects)，模块官方的描述是：

> Drop-in replacement for Node's `http` and `https` modules that automatically follows redirects.

也就是说该模块兼具内置 http & https 模块的能力，且还具备了 **「 跟随重定向(Follow Redirects) 」 ** 能力（默认 **「 跟随重定向(Follow Redirects) 」 ** 上限为 21 次，即 ```maxRedirects``` 属性）。

了解了这些之后，想要修复**"代理配置丢失"**的问题，那么就要去了解 follow-redirects 模块的使用方法了，这里找到官方demo：

```js
const url = require('url');
const { http, https } = require('follow-redirects');

const options = url.parse('http://bit.ly/900913');
options.maxRedirects = 10;
options.beforeRedirect = (options, { headers }) => {
  // 重定向时调整options
  if (options.hostname === "example.com") {
    options.auth = "user:password";
  }
};
http.request(options);
```

那么这个 ```options.beforeRedirect``` 就是我们要找的东西了，它在执行请求前传入options，运行函数体实现对 options 的修改，所以需要 axios 项目的  /lib/adapters/http.js 中添加 beforeRedirect，在重定向的时候将原本的 **代理配置** 加入 options 即可.

### 4. 以 TDD 的方式修复 issue

为了测试修复效果，我们将 axios v0.21.0 版本的源码 (commit 为 94ca24b5b23f343769a15f325693246e07c177d2) 拉到本地，并复制 axios v0.21.1 版本的测试用例 [/test/unit/regression/SNYK-JS-AXIOS-1038255.js](https://github.com/axios/axios/blob/master/test/unit/regression/SNYK-JS-AXIOS-1038255.js) 的代码，到项目中新建同名文件夹并黏贴，下面是该测试用例的代码及分析：

``` js
// https://snyk.io/vuln/SNYK-JS-AXIOS-1038255
// https://github.com/axios/axios/issues/3407
// https://github.com/axios/axios/issues/3369

const axios = require('../../../index');
const http = require('http');
const assert = require('assert');

const PROXY_PORT = 4777; // 代理服务器端口
const EVIL_PORT = 4666; // 重定向 location 地址的端口，代码逻辑正确的话不应该进入该端口

describe('Server-Side Request Forgery (SSRF)', () => {
  let fail = false;
  let proxy;
  let server;
  let location;
  beforeEach(() => {
    server = http.createServer(function (req, res) {
      fail = true;
      res.end('rm -rf /');
    }).listen(EVIL_PORT);
    proxy = http.createServer(function (req, res) {
      // 第一次请求到达代理服务器时，req.url 为 http://www.google.com/，走返回 302 的逻辑
      // 第二次请求由 axios 的「 跟随重定向(Follow Redirects) 」能力发出，url 应为 http://localhost:4666
      if (req.url === 'http://localhost:' + EVIL_PORT + '/') {
        return res.end(JSON.stringify({
          msg: 'Protected',
          headers: req.headers, // 返回请求头
        }));
      }
      res.writeHead(302, { location }) // 第一次请求达返回状态码 302 和 http://localhost:4666
      res.end()
    }).listen(PROXY_PORT);
  });
  afterEach(() => {
    server.close();
    proxy.close();
  });
  it('obeys proxy settings when following redirects', async () => {
    location = 'http://localhost:' + EVIL_PORT;
    let response = await axios({
      method: "get",
      url: "http://www.google.com/",
      proxy: {
        host: "localhost",
        port: PROXY_PORT,
        auth: {
          username: 'sam',
          password: 'password',
        }
      },
    });

    assert.strictEqual(fail, false);
    assert.strictEqual(response.data.msg, 'Protected');
    assert.strictEqual(response.data.headers.host, 'localhost:' + EVIL_PORT);
    assert.strictEqual(response.data.headers['proxy-authorization'], 'Basic ' + Buffer.from('sam:password').toString('base64'));

    return response;
  });
});

```

然后在本地用 ```npm test``` 或者 ```yarn test``` 跑测试，结果如下：

![image-20210109021851406](http://img.nodreame.cn/image-20210109021851406.png)

确实是新的测试用例炸了，因为 axios 中并没有实现相应能力所以没有提示任何问题，现在我们来验证编写修复代码：

问题出在 **代理配置** 丢失，所以到 /lib/adapters/http.js  找到代理相关代码(148~152行)：

![截屏2021-01-09_上午2_22_20](http://img.nodreame.cn/%E6%88%AA%E5%B1%8F2021-01-09_%E4%B8%8A%E5%8D%882_22_20.png)

复制一份到 beforeRedirect 中（axios 项目有 lint 检查所以改 options 为 tmpOptions）这样应该就不会出现代理丢失的情况了~

```js
if (proxy) {
  options.beforeRedirect = function(tmpOption) {
    tmpOption.hostname = proxy.host;
    tmpOption.host = proxy.host;
    tmpOption.headers.host = parsed.hostname + (parsed.port ? ':' + parsed.port : '');
    tmpOption.port = proxy.port;
    tmpOption.path = protocol + '//' + parsed.hostname + (parsed.port ? ':' + parsed.port : '') + options.path;
  };
}
```

将上面代码加入到  /lib/adapters/http.js 的160 行处，再次用 ```npm test``` 或者 ```yarn test``` 跑测试，这次成功地进行**「 跟随重定向(Follow Redirects) 」** 请求并且由于超过最大次数而报错了：

![image-20210109022614643](http://img.nodreame.cn/image-20210109022614643.png)

之所以没有达到想要的效果，是因为没有为**「 跟随重定向(Follow Redirects) 」** 请求配置正确的请求目标链接。原本测试用例期望：

- 第一次请求到达代理服务器时，req.url 为 http://www.google.com/，走返回 302 的逻辑并将**「 跟随重定向(Follow Redirects) 」** 的目标链接设置为 http://localhost:4666
- 第二次请求由 axios 的「 跟随重定向(Follow Redirects) 」能力发出，req.url 应为 http://localhost:4666 

但是实际情况是第一次请求获得 302 返回后并没有更新目标链接，所以还是要阅读下  [follow-redirects 模块](https://github.com/follow-redirects) 中 ```options.beforeRedirect``` 的调用位置（根目录下的 [index.js](https://github.com/follow-redirects/follow-redirects/blob/master/index.js)）：

![image-20210109235835302](http://img.nodreame.cn/image-20210109235835302.png)

在 357 行可以看到调用 ```options.beforeRedirect``` 时传入了 options & 包含重定向响应体的 responseDetails，于是我们从 response 中获取重新获取目标配置并填充 path 和 headers.host ：

``` js
if (proxy) {
  options.beforeRedirect = function(tmpOption, response) { // response 对应重定向响应体的 responseDetails
    // hostname(host) & port 是请求发送到的服务器的域名 & 端口，现在都是代理配置不变
    // path & headers.host 是目标路径 & 目标host，遇到 302 时应该读 location 然后重新填充
    tmpOption.hostname = proxy.host;
    tmpOption.host = proxy.host;
    tmpOption.port = proxy.port;
    // tmpOption.headers.host = parsed.hostname + (parsed.port ? ':' + parsed.port : '');
    // tmpOption.path = protocol + '//' + parsed.hostname + (parsed.port ? ':' + parsed.port : '') + options.path;
    var redirectInfo = url.parse(response.headers.location);
    tmpOption.path = redirectInfo.href; // 重定向链接
    tmpOption.headers.host = redirectInfo.host; // 重定向的目标 host
  };
}
```

再次运行测试，成功通过：

![image-20210109023712032](http://img.nodreame.cn/image-20210109023712032.png)

### 5. 对比官方的问题解决方法

问题已经处理完毕并且通过测试用例，但是可能存在疏漏所以一定要与官方的修复进行对比验证，这样才符合学习闭环。

可以看到 [#3410相关的提交](https://github.com/axios/axios/pull/3410/commits) ：

![image-20210110003745855](http://img.nodreame.cn/image-20210110003745855.png)

这也是一个 TDD 的过程，首先是编写测试用例重现了 issue，然后对问题进行修复，然后再将代码重构。

当然也可以直接点击最后 File changed 的选项卡，直接看整体修改了哪些代码。

可以看到官方的处理方法除了重构部分之外与我们上面的修复方法基本一致，不过其中有一个点引起了我的兴趣：

![image-20210110004324267](http://img.nodreame.cn/image-20210110004324267.png)

 ```options.beforeRedirect``` 方法体中居然只用一个 redirection 就完成了 redirection.header.host 的赋值，而我们上面是用到了第二个参数的，这里我选择继续到   [follow-redirects 模块](https://github.com/follow-redirects) 中找找，果然在 RedirectableRequest.prototype._processResponse 中找到了这段逻辑：

![截屏2021-01-10_上午12_49_37](http://img.nodreame.cn/%E6%88%AA%E5%B1%8F2021-01-10_%E4%B8%8A%E5%8D%8812_49_37.png)

于是刚刚的修复代码可以不再使用第二个参数 response，并且也不用再重新解析一次 location 了：

``` js
if (proxy) {
  options.beforeRedirect = function(tmpOption) {
    var redirectHost = tmpOption.host; // 先拿出来，防止被覆盖
    tmpOption.hostname = proxy.host;
    tmpOption.host = proxy.host;
    tmpOption.port = proxy.port;
    tmpOption.path = tmpOption.href; // 重定向的目标路径
    tmpOption.headers.host = redirectHost; // 重定向的目标 host
  };
}
```

再次运行测试，成功通过：

![image-20210110005622423](http://img.nodreame.cn/image-20210110005622423.png)

OK 到这里对于 [#3410](https://github-redirect.dependabot.com/axios/axios/issues/3410)  的分析就全部完成了~ 对第二个问题 Protocol not parsed when setting proxy config from env vars [#3070](https://github-redirect.dependabot.com/axios/axios/issues/3070) 也可以尝试用这样的方法解析学习哟~

## 撒花✿✿ヽ(°▽°)ノ✿

本来想着周四写完就发了，没想到没把握好 **分析部分** 的度然后越写越多，写完又重构了一下  =。=（累了累了

欢迎拍砖，觉得还行也点赞收藏~ 新开公号："无梦清秋" 欢迎关注（搜索 Nodreame 也可以~）