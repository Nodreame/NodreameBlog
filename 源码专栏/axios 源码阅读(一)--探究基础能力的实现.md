# axios 源码阅读(一)--探究基础能力的实现

> axios 是一个通用的宝藏请求库，此次探究了 axios 中三个基础能力的实现，并将过程记录于此.

![833684](http://img.nodreame.cn/833684.png)

## 零. 前置

- axios项目地址：<https://github.com/axios/axios>
- 阅读代码commit hash：[fe52a611efe756328a93709bbf5265756275d70d](https://github.com/axios/axios/commit/fe52a611efe756328a93709bbf5265756275d70d)
- 最近 Release 版本：[v0.21.1](https://github.com/axios/axios/releases/tag/v0.21.1)

## 一. 目标

阅读源码肯定是带着问题来学习的，所以以下是本次源码阅读准备探究的问题:

- Q1. 如何实现同时支持 ```axios(config)``` 及 ```axios.get(config)``` 语法
- Q2. 浏览器和 NodeJS 请求能力的兼容实现
- Q3. 请求 & 响应拦截器实现

## 二. 项目结构

简单提炼下项目中比较重要的部分:

``` md
├─ dist         最终生成的 axios.js 及其 sourcemap
├─ test        测试用例
├─ examples       浏览器测试用例
├─ index.js       入口文件
├─ Gruntfile.js     定义了"test"、"build"、"version" 命令的能力
├─ karma.conf.js   测试逻辑，运行 "grunt test" 会运行该逻辑
├─ webpack.config.js  生成 axios.js 用的 Webpack 配置文件
└─ lib        axios库功能实现代码
  ├─ adapters      - 适配器
  ├─ cancel       - 取消请求
 ├─ core        - 核心代码（包含 Axios 类、路径&配置合成、请求实现、拦截实现、错误处理）
  ├─ helpers      - 工具方法实现
  ├─ axios.js      - 入口文件
 ├─ defaults.js    - 默认配置
 └─ utils.js      - 工具方法
```

## 三. 从 axios 对象开始分析

当在项目中使用 axios 库时，总是要通过 axios 对象调用相关能力，在项目的 test/typescipt/axios.ts 中有比较清晰的测试用例（以下为简化掉 then 和 catch 后的代码）：

```js
axios(config)
axios.get('/user?id=12345')
axios.get('/user', { params: { id: 12345 } })
axios.head('/user')
axios.options('/user')
axios.delete('/user')
axios.post('/user', { foo: 'bar' })
axios.post('/user', { foo: 'bar' }, { headers: { 'X-FOO': 'bar' } })
axios.put('/user', { foo: 'bar' })
axios.patch('/user', { foo: 'bar' })
```

从上面代码可以看出 axios 库 **同时支持 ```axios(config)``` 和 ```axios.get(config)``` 语法**， 并且支持多种请求方法；

接下来逐个分析每个能力的实现过程~

### 3.1. 弄清 axios 对象

同时支持 ```axios(config)``` 和 ```axios.get(config)``` 语法，说明 axios 是个函数，同时也是一个具备方法属性的对象. 所以接下来我们来分析一下 axios 库暴露的 axios对象.

从项目根目录可以找到入口文件 index.js，其指向 lib 目录下的 axios.js, 这里做了三件事：

- 1）使用工厂方法创建实例axios

  ``` js
  function createInstance(defaultConfig) {
    // 对 Axios 类传入配置对象得到实例，并作为 Axios.prototype.request 方法的上下文
    // 作用：支持通过 axios('https://xxx/xx') 语法实现请求
    var context = new Axios(defaultConfig);
    var instance = bind(Axios.prototype.request, context); // 手撸版本的 `Function.prototype.bind`
  
    // 为了实例具备 Axios 的能力，故将 Axios的原型 & Axios实例的属性 复制给 instance
    utils.extend(instance, Axios.prototype, context);
    utils.extend(instance, context);
    return instance; // instance 的真面目是 request 方法
  }
  
  var axios = createInstance(defaults);
  ```

- 2）给实例axios 挂载操作方法

  ``` js
  axios.Axios = Axios; // 可以通过实例访问 Axios 类
  axios.create = function create(instanceConfig) {
    // 使用新配置对象，通过工厂类获得一个新实例
    // 作用：如果项目中有固定的配置可以直接 cosnt newAxios = axios.create({xx: xx})
    //   然后使用 newAxios 按照 axios API 实现功能
    return createInstance(mergeConfig(axios.defaults, instanceConfig));
  };
  
  // 取消请求三件套
  axios.Cancel = require('./cancel/Cancel');
  axios.CancelToken = require('./cancel/CancelToken');
  axios.isCancel = require('./cancel/isCancel');
  
  axios.all = function all(promises) {
    return Promise.all(promises); // 相当直接，其实就是 Promise.all
  };
  axios.spread = require('./helpers/spread'); // `Function.prototype.apply` 语法糖
  
  // 判定是否为 createError 创建的 Error 对象
  axios.isAxiosError = require('./helpers/isAxiosError');
  ```

- 3）暴露实例axios

  ``` js
  module.exports = axios;
  module.exports.default = axios;
  ```

从以上分析可以得到结论，之所以能够 "同时支持 ```axios(config)``` 和 ```axios.get(config)``` 语法" ，是因为：

- 暴露的 axios 实例本来就是 request 函数，所以支持 ```axios(config)``` 语法
- 工厂方法 ```createInstance``` 最终返回的对象，具备复制得的 Axios的原型 & Axios实例的属性，所以也能像 Axios 实例一样直接使用 ```axios.get(config)``` 语法

### 3.2. 支持多种请求方法

axios 支持 get | head | options | delete | post | put | patch 当然并不是笨笨地逐个实现的，阅读文件 lib/core/Axios.js 可以看到如下结构：

```js
function function Axios(instanceConfig) {} // Axios 构造函数

// 原型上只有两个方法
Axios.prototype.request = function request(config) {};
Axios.prototype.getUri = function getUri(config) {};

// 为 request 方法提供别名
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  Axios.prototype[method] = function(url, config) {
    return this.request(mergeConfig(config...)); // 这里省略了 mergeConfig 对入参的整合
  };
});
utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) { 
  // 基本同上
});

module.exports = Axios;
```

其中核心实现自然是通用请求方法 Axios.prototype.request，其他函数通过调用 request 方法并传入不同的配置参数来实现差异化.

## 四、封装请求实现--"适配器"

总所周知，浏览器端请求一般是通过 XMLHttpRequest 或者 fetch 来实现请求，而 NodeJS 端则是通过内置模块 http 等实现. 那么 axios 是如何实现封装请求实现使得一套 API 可以在浏览器端和 NodeJS 端通用的呢？让我们继续看看 Axios.prototype.request 的实现，简化代码结构如下：

``` js
Axios.prototype.request = function request(config) {
  // 此处省略请求配置合并代码，最终得到包含请求信息的 config 对象

  // chain 可以视为请求流程数组，当不添加拦截器时 chain 如下
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);
 
  // 下方是拦截器部分，暂时忽略
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {});
  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {});
  // 执行请求
  while (chain.length) {
    // 请求流程数组前两个出栈，当前分别为 dispatchRequest 和 undefined
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
};
```

当没有添加任何拦截器的时候，请求流程数组 chain 中就只有 ``` [dispatchRequest, undefined] ``` ，此时在下方的 while 只运行一轮，dispatchRequest 作为 then 逻辑接收 config 参数并运行，继续找到 dispatchRequest 的简化实现（/lib/core/dispatchRequest.js）：

``` js
module.exports = function dispatchRequest(config) {
  // 取消请求逻辑，稍后分析
  throwIfCancellationRequested(config);
  
  // 此处略去了对 config 的预处理
  
  // 尝试获取 config 中的适配器，如果没有则使用默认的适配器
  var adapter = config.adapter || defaults.adapter;

  // 将 config 传入 adapte 执行，得到一个 Promise 结果
  // 如果成功则将数据后放入返回对象的 data 属性，失败则放入返回结果的 response.data 属性
  return adapter(config).then(function onAdapterResolution(response) {
    throwIfCancellationRequested(config);
    response.data = transformData(...); // 此处省略入参
    return response; // 等同于 Promise.resolve(response)
  }, function onAdapterRejection(reason) {
    if (!isCancel(reason)) {
      throwIfCancellationRequested(config);
      if (reason && reason.response) {
        reason.response.data = transformData(...); // 此处省略入参
      }
    }
    return Promise.reject(reason);
  });
};

```

简单过完一遍 dispatchRequest 看到重点在于 ```adapter(config)``` 之后发生了什么，于是找到默认配置的实现（/lib/default.js）：

``` js
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') {
    // 提供给浏览器用的 XHR 适配器
    adapter = require('./adapters/xhr');
  } else if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') {
    // 提供给 NodeJS 用的内置 HTTP 模块适配器
    adapter = require('./adapters/http');
  }
  return adapter;
}

var defaults = {
  adapter: getDefaultAdapter(),
}
```

显而易见，适配器通过判断 XMLHttpRequest 和 process 对象来判断当前平台并获得对应的实现. 接下来继续进入 /lib/adapters 目录，里面的 xhr.js 和 http.js 分别对应适配器的浏览器实现和NodeJS 实现，而 README.md 介绍了实现 adapter 的规范：

``` js
// /lib/adapters/README.md
module.exports = function myAdapter(config) {
  // 使用 config 参数构建请求并实现派发，获得返回后则交给 settle 处理得到 Promise
  return new Promise(function(resolve, reject) {
    var response = {
      data: responseData,
      status: request.status,
      statusText: request.statusText,
      headers: responseHeaders,
      config: config,
      request: request
    };

    // 根据 response 的状态，成功则执行 resolve，否则执行 reject并传入一个 AxiosError
    settle(resolve, reject, response);
  });
}

// /lib/core/settle.js
module.exports = function settle(resolve, reject, response) {
  var validateStatus = response.config.validateStatus;
  if (!response.status || !validateStatus || validateStatus(response.status)) {
    resolve(response);
  } else {
    // 省略参数，createError 创建一个 Error 对象并为其添加 response 相关属性方便读取
    reject(createError(...));
  }
};

```

实现自定义适配器要先接收 config , 并基于 config  参数构建请求并实现派发，获得结果后返回 Promise，接下来的逻辑控制权就交回给 /lib/core/dispatchRequest.js 继续处理了.

## 五. 拦截器实现

### 5.1. 探究"请求|响应拦截器"的实现

axios 的一个常用特性就是拦截器，只需要简单的一行 ```axios.interceptors.request.use(config => return config)```就能实现请求/响应的拦截，在项目有鉴权需求或者返回值需要预处理时相当常用. 现在就来看看这个特性是如何实现的，回到刚刚看过的 Axios.js：

``` js
// /lib/core/Axios.js
Axios.prototype.request = function request(config) {
  // 此处省略请求配置合并代码，最终得到包含请求信息的 config 对象

  // chain 可以视为请求流程数组，当不添加拦截器时 chain 如下
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);
 
  // 拦截器实现
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    // 往 chain 数组头部插入 then & catch逻辑
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });
  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected); // 往 chain 尾部插入处理逻辑
  });
  // 执行请求
  while (chain.length) {
    // 请求流程数组前两个出栈，当前分别为 dispatchRequest 和 undefined
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
};
```

之前说过了项目中使用的 axios 其实就是 Axios.prototype.request，所以当 Axios.prototype.request 触发时，会遍历 ```axios.interceptors.request``` 及 ```axios.interceptors.response``` 并将其中的拦截逻辑添加到"请求流程数组 chain" 中.

在 Axios.prototype.request 中并没有 interceptors 属性的实现，于是回到 Axios 构造函数中寻找对应逻辑（之前说过，工厂函数 createInstance 会将 Axios的原型 & Axios实例的属性 复制给生成的 axios 对象）：

``` js
// /lib/core/Axios.js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  // 实例化 Axios 时也为其添加了 interceptors 对象，其携带了 request & response 两个实例
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}

// /lib/core/InterceptorManager.js
function InterceptorManager() {
  this.handlers = []; // 数组，用于管理拦截逻辑
}
InterceptorManager.prototype.use = function use(fulfilled, rejected) {} // 添加拦截器
InterceptorManager.prototype.eject = function eject(id) {} // 删除拦截器
InterceptorManager.prototype.forEach = function forEach(fn) {} // 遍历拦截器
```

Axios 构造函数在创建实例时会完成  interceptors 属性的创建，实现 ```axios.interceptors.request``` 及 ```axios.interceptors.response``` 对于拦截逻辑的管理.

### 5.2.  实验：同 axios 内多个拦截器的执行顺序

由于 ```axios.interceptors.request``` 遍历添加到 "请求流程数组 chain" 向数组头插入 request 拦截器，所以越后 use 的 request  拦截器会越早执行. 相反，越后 use 的 response 拦截器会越晚执行.

现在假设为 axios 添加两个请求拦截器和两个响应拦截器，那么 "请求流程数组 chain" 就会变成这样（请求拦截器越后 use 的会越先执行）：

```js
[
 request2_fulfilled, request2_rejected,
 request1_fulfilled, request1_rejected,
 dispatchRequest, undefined
 response3_fulfilled, response3_rejected,
  response4_fulfilled, response4_rejected,
]
```

为此编写个单测用于打印验证：

``` js
it('should add multiple request & response interceptors', function (done) {
  var response;

  axios.interceptors.request.use(function (data) {
    console.log('request1_fulfilled, request1_rejected')
    return data;
  });
  axios.interceptors.request.use(function (data) {
    console.log('request2_fulfilled, request2_rejected')
    return data;
  });
  axios.interceptors.response.use(function (data) {
    console.log('response3_fulfilled, response3_rejected')
    return data;
  });
  axios.interceptors.response.use(function (data) {
    console.log('response4_fulfilled, response4_rejected')
    return data;
  });

  axios('/foo').then(function (data) {
    response = data;
  });

  getAjaxRequest().then(function (request) {
    request.respondWith({
      status: 200,
      responseText: 'OK'
    });

    setTimeout(function () {
      expect(response.data).toBe('OK');
      done();
    }, 100);
  });
});
```

结果如下，拦截器的执行打印与期望的结果一致：

![](http://img.nodreame.cn/20201223033418.png)

### 5.3. 探究"取消拦截器"实现

使用以下与 setTimeout 和 clearTimeout 相似的语法，即可将一个定义好的拦截器给取消掉：

``` js
var intercept = axios.interceptors.response.use(function (data) { return data; }); // 定义拦截器
axios.interceptors.response.eject(intercept); // 取消拦截器
```

这里用到了 use 和 eject 两个方法，所以到 InterceptorManager.js 中找找相关实现：

``` js
// /lib/core/InterceptorManager.js
function InterceptorManager() {
  this.handlers = []; // 数组，用于管理拦截逻辑
}

// 添加拦截器
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  // 将拦截逻辑包装为对象，推入管理数组 handles 中
  this.handlers.push({ fulfilled, rejected });
  return this.handlers.length - 1; // 返回当前下标
};

// 通过取消拦截器
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null; // 根据下标判断，存在则置空掉
  }
};
```

这里逻辑就比较简单了，由 InterceptorManager 创建一个内置数组的实例来管理所有拦截器，use 推入一个数组并返回其数组下标，eject 时也用数组下标来置空，这样就起到了拦截器管理的效果了~

## 小结

至此文章就告一段落了，还记得开始提出的三个问题吗？

- Q1. 如何实现同时支持 ```axios(config)``` 及 ```axios.get(config)``` 语法
- A1. axios 库暴露的 axios 对象本来就是一个具备 Axios 实例属性的 Axios.prototype.request 函数. 详见[第三节.从 axios 对象开始分析](## 三. 从 axios 对象开始分析).
- Q2. 浏览器和 NodeJS 请求能力的兼容实现
- A2. 通过判断平台后选择对应平台的适配器实现. 详见[第四节. 封装请求实现--"适配器"](## 四、封装请求实现--"适配器").
- Q3. 请求 & 响应拦截器实现
- A3. 通过数组的形式管理, 将请求拦截器、请求、响应拦截器都放在"请求流程数组 chain" 中，请求时依次执行直到 "请求流程数组 chain" 为空. 详见[第五节. 拦截器实现](## 五. 拦截器实现).
