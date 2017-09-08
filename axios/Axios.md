## 这个md文件中，前面的axios.js，是模块的输出文件，

## 后面写Axios.js，是模块的核心
//总体来说axios.js没看懂，createinstance中，bind弄得人糊涂，axios的继承方法以前也是没见过

## axios.js
```
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  var instance = bind(Axios.prototype.request, context);//这里其实就是Axios.prototype.request.apply（context（Axios的实例），现在还没有设置的参数），Axios.prototype.request本来就是context实例的方法呀，调用request的时候也能改变this.interceptors的值的呀？？？

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);//这里又把 Axios.prototype绑定到了context实例上，还把 Axios.prototype复制给了instance

  // Copy context to instance
  utils.extend(instance, context);//把context复制给了instance，这回没有绑定了

  return instance;
}

// Create the default instance to be exported
var axios = createInstance(defaults);//创建默认实例

// Expose Axios class to allow class inheritance
axios.Axios = Axios;//???继承？

// Factory for creating new instances
axios.create = function create(instanceConfig) {
  return createInstance(utils.merge(defaults, instanceConfig));
};//这里给出了create方法

// Expose Cancel & CancelToken
axios.Cancel = require('./cancel/Cancel');//？？？还没看
axios.CancelToken = require('./cancel/CancelToken');
axios.isCancel = require('./cancel/isCancel');

// Expose all/spread
axios.all = function all(promises) {//？？？这里的promise也是一个可迭代对象，不是一个函数
  return Promise.all(promises);
};
axios.spread = require('./helpers/spread');//装饰，spread（callback）（args）

module.exports = axios;

// Allow use of default import syntax in TypeScript
module.exports.default = axios;

```

## Axios.js
//总的来说这个js文件是个决定axios运行方式的文件，提供了request方法
```
function Axios(instanceConfig) {
  this.defaults = instanceConfig;//复制config
  this.interceptors = {
    request: new InterceptorManager(),//添加管理器，是一个数组，还有三个方法
    response: new InterceptorManager()
  };
}
```

```
Axios.prototype.request = function request(config) {
  /*eslint no-param-reassign:0*/
  // Allow for axios('example/url'[, config]) a la fetch API
  if (typeof config === 'string') {
    config = utils.merge({
      url: arguments[0]
    }, arguments[1]);//这里的arguments[1]指的是什么
  }

  config = utils.merge(defaults, this.defaults, { method: 'get' }, config);//合并，四个参数中越后面越重要，默认的方法是get
  config.method = config.method.toLowerCase();//方法是小写字母

  // Support baseURL config
  if (config.baseURL && !isAbsoluteURL(config.url)) {
    config.url = combineURLs(config.baseURL, config.url);//合并出url
  }

  // Hook up interceptors middleware
  var chain = [dispatchRequest, undefined];//???
  var promise = Promise.resolve(config);//初始化promise，不过resolve后的括号内不应该是一个函数吗？？？

  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });//从头部添加request

  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });从尾部添加response

  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }//从头部开始执行

  return promise;
};
//添加Axios的别名
// Provide aliases for supported request methods
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, config) {
    return this.request(utils.merge(config || {}, {
      method: method,
      url: url
    }));
  };
});
//这三个还需要提供data
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
```