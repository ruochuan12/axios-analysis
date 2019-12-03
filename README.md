# 学习 axios 源码整体架构，打造属于自己的请求库

## 前言

>这是`学习源码整体架构系列`第六篇。整体架构这词语好像有点大，姑且就算是源码整体结构吧，主要就是学习是代码整体结构，不深究其他不是主线的具体函数的实现。本篇文章学习的是实际仓库的代码。

`学习源码整体架构系列`文章如下：
>1.[学习 jQuery 源码整体架构，打造属于自己的 js 类库](https://juejin.im/post/5d39d2cbf265da1bc23fbd42)<br>
>2.[学习 underscore 源码整体架构，打造属于自己的函数式编程类库](https://juejin.im/post/5d4bf94de51d453bb13b65dc)<br>
>3.[学习 lodash 源码整体架构，打造属于自己的函数式编程类库](https://juejin.im/post/5d767e1d6fb9a06b032025ea)<br>
>4.[学习 sentry 源码整体架构，打造属于自己的前端异常监控SDK](https://juejin.im/post/5dba5a39e51d452a2378348a)<br>
>5.[学习 vuex 源码整体架构，打造属于自己的状态管理库](https://juejin.im/post/5dd4e61a6fb9a05a5c010af0)<br>

感兴趣的读者可以点击阅读。下一篇可能是学习 `axios` 源码。

TODO:
**导读**<br>

TODO: 提问

4.怎么实现
5.简述 `axios` 流程

axios-analysis

[umi-request](https://github.com/umijs/umi-request/blob/master/README_zh-CN.md)

## chrome 和 vscode  调试 axios 源码方法

前不久，笔者在知乎回答了一个问题[一年内的前端看不懂前端框架源码怎么办？](https://www.zhihu.com/question/350289336/answer/910970733)
主要有四点：<br>
>1.借助调试<br>
>2.搜索查阅相关高赞文章<br>
>3.把不懂的地方记录下来，查阅相关文档<br>
>4.总结<br>

看源码，调试很重要，所以笔者写下 `axios` 源码调试方法，帮助一些可能不知道如何调试的读者。

### chrome 调试浏览器环境 的 axios

调试方法

`axios`打包后有`sourcemap`文件。

```bash
# 可以克隆我的这个仓库代码
git clone https://github.com/lxchuan12/axios-analysis.git
cd axios-analaysis/axios
npm install
npm start
# open [http://localhost:3000](http://localhost:3000)
# chrome F12 source 控制面板  webpack//   .  lib 目录下，根据情况自行断点调试
```

### vscode 调试 node 环境的 axios

在根目录下 `axios-analysis/`
创建`.vscode/launch`文件如下：

```json
{
    // 使用 IntelliSense 了解相关属性。
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "Launch Program",
            "program": "${workspaceFolder}/axios/sandbox/client.js",
            "skipFiles": [
                "<node_internals>/**"
            ]
        },
    ]
}
```

按`F5`开始调试即可，按照自己的情况断点调试。

## 先看 axios 结构是怎样的

说完了调试方法，直接在 `chrome` 浏览器中调试。

```bash
git clone https://github.com/lxchuan12/axios-analysis.git
cd axios-analaysis/axios
npm install
npm start
```

打开 http://localhost:3000，在控制台打印出，`axios`。

```js
console.log({axios: axios});
```

点开来看，`axios` 的结构是怎样的，先有一个大概印象。

TODO: 画图。笔者画了一张图表示。


## axios 原理

## axios 初始化

看源码第一步，先看`package.json`。一般都会申明`main`主入口文件。

```json
// package.json
{
  "name": "axios",
  "version": "0.19.0",
  "description": "Promise based HTTP client for the browser and node.js",
  "main": "index.js",
  // ...
}
```

主入口文件

```js
// index.js
module.exports = require('./lib/axios');
```

### `lib/axios`文件

```js
// lib/axios
'use strict';

var utils = require('./utils');
var bind = require('./helpers/bind');
var Axios = require('./core/Axios');
var mergeConfig = require('./core/mergeConfig');
var defaults = require('./defaults');

/**
 * Create an instance of Axios
 *
 * @param {Object} defaultConfig The default config for the instance
 * @return {Axios} A new instance of Axios
 */
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  var instance = bind(Axios.prototype.request, context);

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);

  // Copy context to instance
  utils.extend(instance, context);

  return instance;
}

// Create the default instance to be exported
// 导出 创建默认实例
var axios = createInstance(defaults);

// Expose Axios class to allow class inheritance
// 暴露 Axios calss 允许 class 继承
axios.Axios = Axios;

// Factory for creating new instances
// 工厂模式 创建新的实例 用户可以自定义一些参数
axios.create = function create(instanceConfig) {
  return createInstance(mergeConfig(axios.defaults, instanceConfig));
};

// Expose Cancel & CancelToken
// 导出 Cancel 和 CancelToken
axios.Cancel = require('./cancel/Cancel');
axios.CancelToken = require('./cancel/CancelToken');
axios.isCancel = require('./cancel/isCancel');

// Expose all/spread
// 导出 all 和 spread API
axios.all = function all(promises) {
  return Promise.all(promises);
};
axios.spread = require('./helpers/spread');

module.exports = axios;

// Allow use of default import syntax in TypeScript
// 也就是可以以下方式引入
// import axios from 'axios';
module.exports.default = axios;
```

### Axios

`lib/core/Axios.js`

```js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
Axios.prototype.request = function(){
  // 省略，这个是核心方法，后文结合例子详细描述
  // code ...
  var promise = Promise.resolve(config);
  // code ...
  return promise;
}
Axios.prototype.getUri = function(){}
// 提供一些请求方法的别名
// Provide aliases for supported request methods
// 遍历执行
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, config) {
    return this.request(utils.merge(config || {}, {
      method: method,
      url: url
    }));
  };
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, data, config) {
    return this.request(utils.merge(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});

module.exports = Axios;
```

### InterceptorManager 拦截器

```js
function InterceptorManager() {
  this.handlers = [];
}
// 声明了三个方法：使用、移除、遍历
InterceptorManager.prototype.use = function(){}
InterceptorManager.prototype.eject = function(){}
InterceptorManager.prototype.forEach = function(){}
```

## 实例结合

### 调用栈流程

知道 `axios` 使用了`XMLHttpRequest`。
可以在项目中搜索：`new XMLHttpRequest`。
定位到文件 `axios/lib/adapters/xhr.js`
在这条语句 `var request = new XMLHttpRequest();`
`chrome` 浏览器中 打个断点，再根据调用栈来细看具体函数等实现。

`Call Stack`

```bash
dispatchXhrRequest (xhr.js:19)
xhrAdapter (xhr.js:12)
dispatchRequest (dispatchRequest.js:60)
Promise.then (async)
request (Axios.js:54)
wrap (bind.js:10)
submit.onclick ((index):138)
```

```js
Axios.prototype.request = function request(config) {
  /*eslint no-param-reassign:0*/
  // Allow for axios('example/url'[, config]) a la fetch API
  if (typeof config === 'string') {
    config = arguments[1] || {};
    config.url = arguments[0];
  } else {
    config = config || {};
  }

  config = mergeConfig(this.defaults, config);

  // Set config.method
  if (config.method) {
    config.method = config.method.toLowerCase();
  } else if (this.defaults.method) {
    config.method = this.defaults.method.toLowerCase();
  } else {
    config.method = 'get';
  }

  // Hook up interceptors middleware
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);

  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });

  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });

  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
};
```

### dispatchRequest

```js
var adapter = config.adapter || defaults.adapter;

return adapter(config)
```

### adapter

## 总结

`Axios` 源码相对不多，打包后一千多行，比较容易看完，非常值得学习。

如果读者发现有不妥或可改善之处，再或者哪里没写明白的地方，欢迎评论指出。另外觉得写得不错，对您有些许帮助，可以点赞、评论、转发分享，也是对笔者的一种支持，非常感谢呀。

## 推荐阅读

[@叫我小明呀：Axios 源码解析](https://juejin.im/post/5cb5d9bde51d456e62545abc)<br>
[@尼库尼库桑：深入浅出 axios 源码](https://zhuanlan.zhihu.com/p/37962469)<br>
[@小贼先生_ronffy：Axios源码深度剖析 - AJAX新王者](https://juejin.im/post/5b0ba2d56fb9a00a1357a334)<br>
[逐行解析Axios源码](https://juejin.im/post/5d501512518825159e3d7be6)<br>
[[译]axios 是如何封装 HTTP 请求的](https://juejin.im/post/5d906269f265da5ba7451b02)<br>

## 笔者精选文章

[面试官问：JS的继承](https://juejin.im/post/5c433e216fb9a049c15f841b)<br>
[面试官问：JS的this指向](https://juejin.im/post/5c0c87b35188252e8966c78a)<br>
[面试官问：能否模拟实现JS的call和apply方法](https://juejin.im/post/5bf6c79bf265da6142738b29)<br>
[面试官问：能否模拟实现JS的bind方法](https://juejin.im/post/5bec4183f265da616b1044d7)<br>
[面试官问：能否模拟实现JS的new操作符](https://juejin.im/post/5bde7c926fb9a049f66b8b52)<br>

## 关于

作者：常以**若川**为名混迹于江湖。前端路上 | PPT爱好者 | 所知甚少，唯善学。<br>
[个人博客-若川](https://lxchuan12.cn/posts/)，使用`vuepress`重构了，阅读体验可能更好些<br>
[掘金专栏](https://juejin.im/user/57974dc55bbb500063f522fd/posts)，欢迎关注~<br>
[`segmentfault`前端视野专栏](https://segmentfault.com/blog/lxchuan12)，欢迎关注~<br>
[知乎前端视野专栏](https://zhuanlan.zhihu.com/lxchuan12)，欢迎关注~<br>
[github blog](https://github.com/lxchuan12/blog)，相关源码和资源都放在这里，求个`star`^_^~

## 欢迎加微信交流 微信公众号

可能比较有趣的微信公众号，长按扫码关注。欢迎加我微信lxchuan12（注明来源，基本来者不拒），拉您进【前端视野交流群】，长期交流学习~

![若川视野](https://github.com/lxchuan12/blog/raw/master/docs/about/wechat-official-accounts-mini.jpg)
