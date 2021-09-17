---
title: axios源码笔记
date: 2021-09-17 16:10:21
tags: 前端技术
---


# 概述

axios 是一个基于promise的支持浏览器环境和node环境的HTTP客户端。在服务器端它使用node.js的`http`模块，在浏览器端它使用`XMLHttpRequests`。它具有以下特性：

- 在浏览器端使用`XMLHttpRequests`发起请求
- 在服务器端它使用node.js的`http`模块发起请求
- 支持Promise API
- 支持拦截请求和响应
- 自动转换请求和响应的数据
- 支持取消请求
- 自动转换JSON数据
- 浏览器端支持抵御`XSRF`攻击


axios源码整体目录结构如下：

```
├── adapters
│   ├── README.md
│   ├── http.js               #node环境http对象
│   └── xhr.js                #web环境http对象
├── axios.js                  #入口
├── cancel
│   ├── Cancel.js             #取消的构造对象类
│   ├── CancelToken.js        #取消操作的包装类
│   └── isCancel.js           #工具类
├── core
│   ├── Axios.js              #Axios实例对象
│   ├── InterceptorManager.js #拦截器控制器
│   ├── README.md
│   ├── buildFullPath.js      #拼接请求的url、baseURL
│   ├── createError.js        #创建异常信息类工具
│   ├── dispatchRequest.js    #默认的拦截器（分发完全请求的拦截器）
│   ├── enhanceError.js       #异常信息类实体
│   ├── mergeConfig.js        #合并配置文件（用户设置的和默认的）
│   ├── settle.js             #根据http-code值来resolve/reject状态
│   └── transformData.js      #转换请求或相应的数据的工具类
├── defaults.js               #默认配置类
├── helpers/                  #一些辅助方法
└── utils.js                  #通用的工具类
```

我们可以从入口文件`axios.js`开始，整体发起一次请求的主要流程如下图所示：

![](/images/axios/flow.png)


# 拦截器Interceptor

所以 axios 提供了请求拦截器和响应拦截器来分别处理请求和响应，它们的作用如下：

请求拦截器：该类拦截器的作用是在请求发送前统一执行某些操作，比如在请求头中添加 token 字段。

响应拦截器：该类拦截器的作用是在接收到服务器响应后统一执行某些操作，比如发现响应状态码为 401 时，自动跳转到登录页。


以下是设置一个请求拦截器和响应拦截器的方法：

```javascript
// 添加一个请求拦截器
axios.interceptors.request.use(function (config) {
    // Do something before request is sent
    return config;
  }, function (error) {
    // Do something with request error
    return Promise.reject(error);
  });

// 添加一个响应拦截器
axios.interceptors.response.use(function (response) {
    // Any status code that lie within the range of 2xx cause this function to trigger
    // Do something with response data
    return response;
  }, function (error) {
    // Any status codes that falls outside the range of 2xx cause this function to trigger
    // Do something with response error
    return Promise.reject(error);
  });
```

每个拦截器都包含以下结构：
```javascript
    fulfilled: fulfilled,  //处理resolve的方法
    rejected: rejected,    //处理reject的方法
    synchronous: options ? options.synchronous : false,  //拦截器是同步还是异步的标记，通过options传入，默认异步
    runWhen: options ? options.runWhen : null  （仅针对请求拦截器） //判断拦截器是否在何种情况下执行的函数，参数时axios对象的config（仅针对请求拦截器）
```

请求的拦截器可以设置同步执行或异步执行，当设置多个请求拦截器的时候，只有所有拦截器都为同步才会同步执行。


请求拦截器按照后进（调用use）先出的顺序（栈）执行，响应拦截器按照先进先出的顺序执行。首先先分别按照以上顺序构建一个请求拦截器队列和响应拦截器队列。

如果请求拦截器按照同步的方式执行，则先按顺序同步执行请求拦截器的队列以及网络请求，之后再构建响应拦截器的执行promise链。

axios 发起 request()整体的执行流程如下图所示：

![](/images/axios/libaxios.png)

```javascript
// axios/lib/core/Axios.js
  var promise;
  var newConfig = config;
  while (requestInterceptorChain.length) {
    var onFulfilled = requestInterceptorChain.shift();
    var onRejected = requestInterceptorChain.shift();
    try {
      newConfig = onFulfilled(newConfig);
    } catch (error) {
      onRejected(error);
      break;
    }
  }

  try {
    promise = dispatchRequest(newConfig);
  } catch (error) {
    return Promise.reject(error);
  }

  while (responseInterceptorChain.length) {
    promise = promise.then(responseInterceptorChain.shift(), responseInterceptorChain.shift());
  }

  return promise;
  
```
如果请求拦截器按照异步的方式执行，则需要将请求拦截器、请求和响应拦截器构建一个Promise执行链。首先构造一个如下存储格式的队列，用于存储每次执行promise的resolve方法和reject方法。执行的时候按照顺序每次从队首弹出一个拦截器的fullfilled方法和reject方法作为Promise的参数执行。

```javascript
  var promise;
  if (!synchronousRequestInterceptors) {
    var chain = [dispatchRequest, undefined];

    Array.prototype.unshift.apply(chain, requestInterceptorChain);
    chain = chain.concat(responseInterceptorChain);

    promise = Promise.resolve(config);
    while (chain.length) {
      promise = promise.then(chain.shift(), chain.shift());
    }

    return promise;
  }
```

这里面的dispatchRequest方法，主要包含三个部分：请求header和data的处理、发起请求、收到响应data的处理。

# 转换器Transformer

请求转换器主要用于处理发起请求和收到响应的报头和body data。这部分逻辑详见axios提供的默认配置。

对于请求，Transformer会根据请求的method设置默认报头，通过分析请求的body格式针对不用的请求数据类型加上对应的报头Content-Type。


```javascript
// axios/lib/defaults.js 
utils.forEach(['delete', 'get', 'head'], function forEachMethodNoData(method) {
  defaults.headers[method] = {};
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  defaults.headers[method] = utils.merge(DEFAULT_CONTENT_TYPE);
});
```

```javascript
// axios/lib/defaults.js
  transformRequest: [function transformRequest(data, headers) {
    normalizeHeaderName(headers, 'Accept');
    normalizeHeaderName(headers, 'Content-Type');

    if (utils.isFormData(data) ||
      utils.isArrayBuffer(data) ||
      utils.isBuffer(data) ||
      utils.isStream(data) ||
      utils.isFile(data) ||
      utils.isBlob(data)
    ) {
      return data;
    }
    if (utils.isArrayBufferView(data)) {
      return data.buffer;
    }
    if (utils.isURLSearchParams(data)) {
      setContentTypeIfUnset(headers, 'application/x-www-form-urlencoded;charset=utf-8');
      return data.toString();
    }
    if (utils.isObject(data) || (headers && headers['Content-Type'] === 'application/json')) {
      setContentTypeIfUnset(headers, 'application/json');
      return stringifySafely(data);
    }
    return data;
  }],
```
对于收到的响应，Transformer会根据设置和响应的类型进行JSON.parse。
```javascript
// axios/lib/defaults.js
  transformResponse: [function transformResponse(data) {
    var transitional = this.transitional;
    var silentJSONParsing = transitional && transitional.silentJSONParsing;
    var forcedJSONParsing = transitional && transitional.forcedJSONParsing;
    var strictJSONParsing = !silentJSONParsing && this.responseType === 'json';

    if (strictJSONParsing || (forcedJSONParsing && utils.isString(data) && data.length)) {
      try {
        return JSON.parse(data);
      } catch (e) {
        if (strictJSONParsing) {
          if (e.name === 'SyntaxError') {
            throw enhanceError(e, this, 'E_JSON_PARSE');
          }
          throw e;
        }
      }
    }

    return data;
  }],
```
# 适配器Adapter

axios的适配器提供了发起网络请求的方法。官方的库里面主要提供了两种，已经基本覆盖了大部分应用场景。针对浏览器环境的`xhr.js`和针对Node环境的`http.js`。如果用户声明了使用的适配器，则使用用户的适配器，如果没找到用户声明，axios会根据所在的环境平台自动判断使用浏览器环境的`xhr.js`还是针对Node环境的`http.js`。

除此以外，用户可以根据自己的需要进行自定义拓展。`axios-mock-adapter`就是一个基于axios实现mock功能的适配器。

以下是官方给的一个Adapter模板：

```javascript
var settle = require('./../core/settle');
module.exports = function myAdapter(config) {
  // 当前时机点：
  //  - config配置对象已经与默认的请求配置合并
  //  - 请求转换器已经运行
  //  - 请求拦截器已经运行
  
  // 使用提供的config配置对象发起请求
  // 根据响应对象处理Promise的状态
  return new Promise(function(resolve, reject) {
    var response = {
      data: responseData,
      status: request.status,
      statusText: request.statusText,
      headers: responseHeaders,
      config: config,
      request: request
    };

    settle(resolve, reject, response);

    // 此后:
    //  - 响应转换器将会运行
    //  - 响应拦截器将会运行
  });
}
```

settle函数主要用于处理response返回的status。逻辑如下：

```javascript
// settle.js
module.exports = function settle(resolve, reject, response) {
  var validateStatus = response.config.validateStatus;
  if (!response.status || !validateStatus || validateStatus(response.status)) {
    resolve(response);
  } else {
    reject(createError(
      'Request failed with status code ' + response.status,
      response.config,
      null,
      response.request,
      response
    ));
  }
};
```
一个Adapter需要返回一个包含响应数据的promise对象。这是因为 axios 内部是通过 Promise 链式调用来完成请求调度。在自定义适配器的时候还应该充分考虑前置后置转换器和拦截器对于数据的处理操作。

# Cancel Token

Cancel token 用户处理请求取消的情况。官方给出的用法如下：

```javascript
const CancelToken = axios.CancelToken;
const source = CancelToken.source();

// get请求用法
axios.get('/user/12345', {
  cancelToken: source.token
}).catch(function (thrown) {
  if (axios.isCancel(thrown)) {
    console.log('Request canceled', thrown.message);
  } else {
    // handle error
  }
});

// post请求用法
axios.post('/user/12345', {
  name: 'new name'
}, {
  cancelToken: source.token
})

// 取消请求
source.cancel('Operation canceled by the user.');
```
以下是CancelToken声明的完整代码，可以看出，每次调用一次`source()`的时候都返回的对象都包含两个属性，一个是CancelToken对象，用于绑定在axios每次请求的配置里，还有一个取消请求的Promise函数。这个Promise函数执行完成即CancelToken内部的promise执行完成。

```javascript
// axios/lib/cancel/CancelToken.js

var Cancel = require('./Cancel');

/**
 * A `CancelToken` is an object that can be used to request cancellation of an operation.
 *
 * @class
 * @param {Function} executor The executor function.
 */
function CancelToken(executor) {
  if (typeof executor !== 'function') {
    throw new TypeError('executor must be a function.');
  }

  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) {
    if (token.reason) {
      // Cancellation has already been requested
      return;
    }

    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}

/**
 * Throws a `Cancel` if cancellation has been requested.
 */
CancelToken.prototype.throwIfRequested = function throwIfRequested() {
  if (this.reason) {
    throw this.reason;
  }
};

/**
 * Returns an object that contains a new `CancelToken` and a function that, when called,
 * cancels the `CancelToken`.
 */
CancelToken.source = function source() {
  var cancel;
  var token = new CancelToken(function executor(c) {
    cancel = c;
  });
  return {
    token: token,
    cancel: cancel
  };
};

module.exports = CancelToken;
```
以下是Adapter里面处理CancelToken的逻辑：

如果检测到Config里头包含已经声明的CancelToken对象，就将处理的事件绑在CancelToken上。当用户手动触发`cancel()`的时候，绑定在promise then后面的Adapter的请求取消函数会继续执行。这就实现了取消网络请求的操作（观察者模式？）。

```javascript
// axios/lib/adapters/xhr.js

    if (config.cancelToken) {
      // Handle cancellation
      config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
          return;
        }

        request.abort();
        reject(cancel);
        // Clean up request
        request = null;
      });
    }
```
# XSRF 跨域请求伪造防御

关于XSRF的介绍详见参考文献4[77.9K Star 的 axios 项目有哪些值得借鉴的地方]
axios通过设置config里面的xsrfCookieName读取站点cookie里面对应的value，然后将value写入请求的header，来实现防御XSRF攻击的目的。这部分逻辑主要是在Adapter里面实现的，仅适用于浏览器环境：

```javascript
// axios/lib/adapters/xhr.js
    if (utils.isStandardBrowserEnv()) {
      // Add xsrf header
      var xsrfValue = (config.withCredentials || isURLSameOrigin(fullPath)) && config.xsrfCookieName ?
        cookies.read(config.xsrfCookieName) :
        undefined;

      if (xsrfValue) {
        requestHeaders[config.xsrfHeaderName] = xsrfValue;
      }
    }
```

# 参考文献
1. [axios](https://github.com/axios/axios)
2. [axios源码简读](https://mp.weixin.qq.com/s/ENlwEbGlp3_Ps6ozj4x3rQ)
3. [axios-mock-adapter](https://www.npmjs.com/package/axios-mock-adapter)
4. [77.9K Star 的 axios 项目有哪些值得借鉴的地方](https://juejin.cn/post/6885471967714115597#heading-12)


