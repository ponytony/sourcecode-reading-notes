## xhr.js
### 总结
//返回的数据格式在设置onreadystatechange中就设定好了
以下按顺序：

- 确认request data是不是formdata，是的话就删除content-type
- 创建ajax实例，如果是ie8/9就创建XDomainRequest
- 有config.auth就设置requestHeaders.Authorization
- 初始化实例
- 设置timeout，根据config
- 设置onload或者onreadystatechange，规定了return条件和返回的{}是怎样的
- 设置onerror
- 设置ontimeout
- 如果是浏览器环境，就设置xsrf相关的xsrfCookierName和xsrfHeaderName
- 设置请求头，条件是看有没有request data
- 如果config有withCredentials，就设置request。withCredentials
- 设置responseType
- 设置中断上传和下载的listener
- 如果config有cancelToken，就abort，reject
- 如果request data是undefine，requestData = null;
- 发送请求
- 
总的来说很杂
```
//涉及到的config：data,headers,url,auth:{username,password},methed,params,timeout,responseType,withCredentials,
//xsrfCookieName,xsrfHeaderName,onDownloadProgress,onUploadProgress,cancelToken,paramsSerializer
function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    var requestData = config.data;
    var requestHeaders = config.headers;

    if (utils.isFormData(requestData)) {
      delete requestHeaders['Content-Type']; // Let the browser set it
    }//如果是formdata，浏览器会自己设置type


    var request = new XMLHttpRequest();//创建ajax实例
    //request一共设置了ontimeout（ie也是），onreadystatechange（onload），onerror，withCredentials，responseType，addEventListener('progress'（onprogress），upload.addEventListener('progress'，
    var loadEvent = 'onreadystatechange';
    var xDomain = false;

    // For IE 8/9 CORS support
    // Only supports POST and GET calls and doesn't returns the response headers.
    // DON'T do this for testing b/c XMLHttpRequest is mocked, not XDomainRequest.
    if (process.env.NODE_ENV !== 'test' &&//ie8或者9
        typeof window !== 'undefined' &&
        window.XDomainRequest && !('withCredentials' in request) &&
        !isURLSameOrigin(config.url)) {
      request = new window.XDomainRequest();
      loadEvent = 'onload';
      xDomain = true;
      request.onprogress = function handleProgress() {};
      request.ontimeout = function handleTimeout() {};
    }

    // HTTP basic authentication
    if (config.auth) {
      var username = config.auth.username || '';
      var password = config.auth.password || '';
      requestHeaders.Authorization = 'Basic ' + btoa(username + ':' + password);//转化成base64
    }

    request.open(config.method.toUpperCase(), buildURL(config.url, config.params, config.paramsSerializer), true);//初始化请求

    // Set the request timeout in MS
    request.timeout = config.timeout;

    // Listen for ready state
    request[loadEvent] = function handleLoad() {
      if (!request || (request.readyState !== 4 && !xDomain)) {
        return;
      }

      // The request errored out and we didn't get a response, this will be
      // handled by onerror instead
      // With one exception: request that using file: protocol, most browsers
      // will return status as 0 even though it's a successful request
      if (request.status === 0 && !(request.responseURL && request.responseURL.indexOf('file:') === 0)) {
        return;
      }

      // Prepare the response
      var responseHeaders = 'getAllResponseHeaders' in request ? parseHeaders(request.getAllResponseHeaders()) : null;//看request中有没有这个属性，有的话就返回headers
      var responseData = !config.responseType || config.responseType === 'text' ? request.responseText : request.response;//看是哪个request返回的数据
      var response = {
        data: responseData,
        // IE sends 1223 instead of 204 (https://github.com/mzabriskie/axios/issues/201)
        //ie是1123相当于204
        status: request.status === 1223 ? 204 : request.status,
        statusText: request.status === 1223 ? 'No Content' : request.statusText,
        headers: responseHeaders,
        config: config,//传入的参数
        request: request//request实例
      };

      settle(resolve, reject, response);//response中没有status或者没有设置config.validateStatus，或者validdataStatus返回的是resolve,否则就是reject

      // Clean up request
      request = null;
    };

    // Handle low level network errors
    //网络错误
    request.onerror = function handleError() {
      // Real errors are hidden from us by the browser
      // onerror should only fire if it's a network error
      reject(createError('Network Error', config, null, request));

      // Clean up request
      request = null;
    };

    // Handle timeout
    //超时
    request.ontimeout = function handleTimeout() {
      reject(createError('timeout of ' + config.timeout + 'ms exceeded', config, 'ECONNABORTED',
        request));

      // Clean up request
      request = null;
    };

    // Add xsrf header
    // This is only done if running in a standard browser environment.
    // Specifically not if we're in a web worker, or react-native.
    if (utils.isStandardBrowserEnv()) {
      var cookies = require('./../helpers/cookies');

      // Add xsrf header
      //isrilsameorigin只传入一个参数是因为函数中已经获取了原本的网站的url 
     //cookieies。read是用正则取出正确的cookie
      var xsrfValue = (config.withCredentials || isURLSameOrigin(config.url)) && config.xsrfCookieName ?
          cookies.read(config.xsrfCookieName) :
          undefined;

      if (xsrfValue) {
        requestHeaders[config.xsrfHeaderName] = xsrfValue;//改成cookie的值
      }
    }

    // Add headers to the request
    if ('setRequestHeader' in request) {//setRequestHeader是ajax自带的方法，用来跨域的
      utils.forEach(requestHeaders, function setRequestHeader(val, key) {
        if (typeof requestData === 'undefined' && key.toLowerCase() === 'content-type') {
          // Remove Content-Type if data is undefined
          delete requestHeaders[key];//如果没有request data，并且规定了content-type，就删除这个contenttype，因为既然都不发送数据了，干嘛还要规定发送的数据的类型
        } else {
          // Otherwise add header to the request
          request.setRequestHeader(key, val);//将RequestHeaders（也就是原来的config.headers）加入到request实例中
        }
      });
    }

    // Add withCredentials to request if needed
    if (config.withCredentials) {
      request.withCredentials = true;
    }

    // Add responseType to request if needed
    if (config.responseType) {
      try {
        request.responseType = config.responseType;
      } catch (e) {
        // Expected DOMException thrown by browsers not compatible XMLHttpRequest Level 2.
        // But, this can be suppressed for 'json' type as it can be parsed by default 'transformResponse' function.
        if (config.responseType !== 'json') {
          throw e;
        }
      }
    }

    // Handle progress if needed
    //下载进度事件，有需要的可以自己定义
    if (typeof config.onDownloadProgress === 'function') {
      request.addEventListener('progress', config.onDownloadProgress);
    }

    // Not all browsers support upload events
    //上传进度事件，有需要可以自己定义
    if (typeof config.onUploadProgress === 'function' && request.upload) {
      request.upload.addEventListener('progress', config.onUploadProgress);
    }

    if (config.cancelToken) {//取消请求，自己定义
      // Handle cancellation
      config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
          return;
        }

        request.abort();//request定义的取消请求
        reject(cancel);
        // Clean up request
        request = null;
      });
    }

    if (requestData === undefined) {
      requestData = null;
    }

    // Send the request
    request.send(requestData);
  });
};
```

## settle.js
看条件来决定是resolve还是reject
```
settle(resolve, reject, response) {
  var validateStatus = response.config.validateStatus;
  // Note: status is not exposed by XDomainRequest
  //response中没有status或者没有设置config.validateStatus，或者validdataStatus返回的是true
  if (!response.status || !validateStatus || validateStatus(response.status)) {
    resolve(response);//直接处理response
  } else {//否则就抛出错误
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