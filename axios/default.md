## default.js
不难，但是为什么transformresponse和transformrequest要处理成数组；

最后几行中，defaults.headers[method] = {}，在default中，merge函数的之后的返回值似乎不对啊

```
//default写的应该都是请求头
//初始的content-type
ar DEFAULT_CONTENT_TYPE = {
  'Content-Type': 'application/x-www-form-urlencoded'
};
//header没有设置content-type时，设置它
function setContentTypeIfUnset(headers, value) {
  if (!utils.isUndefined(headers) && utils.isUndefined(headers['Content-Type'])) {
    headers['Content-Type'] = value;
  }
}
//如果没有XMLHttpRequest，调用/adapter/xhr里的那个promise
function getDefaultAdapter()  {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') {
    // For browsers use XHR adapter
    adapter = require('./adapters/xhr');
  } else if (typeof process !== 'undefined') {//这个process也不知道是哪个环境里的变量
    // For node use HTTP adapter
    adapter = require('./adapters/http');
  }
  return adapter;
}

var defaults = {
  adapter: getDefaultAdapter(),
//这个函数是将传入的data检测一下，并且转化成字符或者二进制，或者其他形式的数据
//这里的参数还没有传入
//transformRequest是在dispatch中使用，transformRequest（config.data,config.headers)
  transformRequest: [function transformRequest(data, headers) {
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
    if (utils.isObject(data)) {
      setContentTypeIfUnset(headers, 'application/json;charset=utf-8');
      return JSON.stringify(data);
    }
    return data;
  }],
//data是string，而且要能parse
  transformResponse: [function transformResponse(data) {
    /*eslint no-param-reassign:0*/
    if (typeof data === 'string') {
      try {
        data = JSON.parse(data);
      } catch (e) { /* Ignore */ }
    }
    return data;
  }],

  timeout: 0,//请求超时

  xsrfCookieName: 'XSRF-TOKEN',
  xsrfHeaderName: 'X-XSRF-TOKEN',

  maxContentLength: -1,

  validateStatus: function validateStatus(status) {
    return status >= 200 && status < 300;
  }
};

defaults.headers = {
  common: {
    'Accept': 'application/json, text/plain, */*'
  }//common最后会和config.headers合并
};

utils.forEach(['delete', 'get', 'head'], function forEachMethodNoData(method) {
  defaults.headers[method] = {};//如果是这三个方法，那么响应头里的方法就是{}
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
//这样写的话，default={header：{method：{'Content-Type': 'application/x-www-form-urlencoded'}}}了吗
  defaults.headers[method] = utils.merge(DEFAULT_CONTENT_TYPE);
});

module.exports = defaults;
//最后将处理好的default传出去
