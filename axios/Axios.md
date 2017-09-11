## 这个md文件中，依次是axios.js,Axios.js,transforndata.js,dispatchrequest.js

axios是给外面的接口，transforndata只是个用来转化data的函数（給config.transformresponse和config.transformrequest服务的）

Axios是个关键，因为这个函数会对config进行一些检验和处理，并且也是在这个文件中调用了dispatchRequest。

dispatchRequest是最重要的函数，这个函数中调用了adapter，对headers做了一点修改，也是在这个文件中给服务器请求的

//总体来说axios.js没看懂，只能写个大概过程

## axios.js
```
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);//创建Axios实例
  var instance = bind(Axios.prototype.request, context);//这里其实就是Axios.prototype.request.apply（context（Axios的实例），现在还没有设置的参数），Axios.prototype.request本来就是context实例的方法呀，调用request的时候也能改变this.interceptors的值的呀？？？

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);//这里又把 Axios.prototype绑定到了context实例上，还把 Axios.prototype复制给了instance(Axios.prototype.request)

  // Copy context to instance
  utils.extend(instance, context);//把context复制给了instance，这回没有绑定了

  return instance;//这个instance是一个Axios.prototype.request，还有Axios.prototype中的方法，还有一个Axios实例
}//这样写应该是为了axios写法的多样性，比如：axios（），axios.get(),axios.request(),

// Create the default instance to be exported
var axios = createInstance(defaults);//创建默认实例

// Expose Axios class to allow class inheritance
axios.Axios = Axios;//这个应该是为了在完成一个ajax之后用response再来创建一个ajax

// Factory for creating new instances
axios.create = function create(instanceConfig) {
  return createInstance(utils.merge(defaults, instanceConfig));
};//这里给出了create方法

// Expose Cancel & CancelToken
axios.Cancel = require('./cancel/Cancel');//取消请求
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
//总的来说这个js文件是个重要的文件，提供了request方法，主要是对config的处理，检验
```
function Axios(instanceConfig) {
  this.defaults = instanceConfig;//复制config
  this.interceptors = {
    request: new InterceptorManager(),//添加管理器，是一个数组，还有三个方法
    response: new InterceptorManager()//new Promise其实就在这个实例中
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
    }, arguments[1]);//这里的英文也写了，参数实际上应该是axios('example/url'[, config])
  }

  config = utils.merge(defaults, this.defaults, { method: 'get' }, config);//合并，四个参数中越后面越重要，默认的方法是get
  config.method = config.method.toLowerCase();//方法是小写字母

  // Support baseURL config
  if (config.baseURL && !isAbsoluteURL(config.url)) {
    config.url = combineURLs(config.baseURL, config.url);//合并出url
  }

  // Hook up interceptors middleware
  //这个dispatchrequest才是执行ajax的关键，这个chain中的其他参数都是对config的修改，或者其他处理
  var chain = [dispatchRequest, undefined];//这里为什么是undefine？大概是因为dispatchRequest中已经做过了reject的处理，不用再这个列表中再处理一遍reject
  var promise = Promise.resolve(config);//初始化promise，自己去试了一下，原来promise可以用Promise.resolve(data)来使用的
//要注意的是这里的foreach不是js定义的那个foreach，而是interceptorManager中定义的那个
//这里的foreach默认设置中是不会执行的，因为没有把拦截器添加到而是interceptorManager中定义的那个this.handler中
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });//从头部添加拦截器
//还有要注意的是这里没有使用interceptorManager中的use和reject，因为这里已经将需要处理的数据准备好了，只要复制进interceptorManager中的handler就好
  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });从尾部添加response的拦截器

  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }//执行拦截器和dispatchrequest

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


## transformData(data, headers, fns)
这个函数就是将fns拆分开来，fn(data,henders)
```
function transformData(data, headers, fns) {
  /*eslint no-param-reassign:0*/
  utils.forEach(fns, function transform(fn) {
    data = fn(data, headers);
  });

  return data;
};
```

## dispatchRequest.js

    
这个函数传入config，用config.transformRequest函数来处理config.data,config.headers,然后取出config.headers[method]的值，放入config.headers，再用adaptor来处理config，再用config.transformRequest函数来处理response.data,response.headers（response是.then中传入的参数，这时候config.headers已经改过一次了），最后返回response或者一个reject
```
function throwIfCancellationRequested(config) {
  if (config.cancelToken) {
    config.cancelToken.throwIfRequested();
  }
}
```

```
function dispatchRequest(config) {
  throwIfCancellationRequested(config);

  // Ensure headers exist
  config.headers = config.headers || {};

  // Transform request data
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest//transform在default.js中有设置
  );

  // Flatten headers
  config.headers = utils.merge(
    config.headers.common || {},
    config.headers[config.method] || {},//从这一行可以看出config有个method，headers里面还有个对应的函数
    config.headers || {}
  );
//删除header里面的各种方法，不过上面不是才刚刚把这些方法添加进去吗？？？
//仔细看了一下，这里其实就是把header中 的各种get，post这类下面的obj取出来，比如'Content-Type': 'application/x-www-form-urlencoded'，这个是default中的一个DEFAULT_CONTENT_TYPE，default会在结尾时根据method类型决定是否加入到defaults.headers[method]，放到了
  utils.forEach(
    ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
    function cleanHeaderConfig(method) {
      delete config.headers[method];
    }
  );

  var adapter = config.adapter || defaults.adapter;//这个adapter是看环境来决定使用浏览器的那个adapter还是用于node的atapter

//关键的来了，下面这一样是整个axios的最核心，这里会提交请求并且获取response，也可能会获取reject
  return adapter(config).then(function onAdapterResolution(response) {
    throwIfCancellationRequested(config);

    // Transform response data
    //转化response，    具体看default.js
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );

    return response;
  }, function onAdapterRejection(reason) {//这个函数是处理拒绝之后的data的
    if (!isCancel(reason)) {
      throwIfCancellationRequested(config);

      // Transform response data
      //这里也要转化一下
      if (reason && reason.response) {
        reason.response.data = transformData(
          reason.response.data,
          reason.response.headers,
          config.transformResponse
        );
      }
    }

    return Promise.reject(reason);
  });
};
```