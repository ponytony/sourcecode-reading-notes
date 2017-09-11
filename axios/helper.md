这个md放入的是helper文件夹里的函数，这些函数大多可以重复使用

## btoa.js将string转为base64

## bind(fn,thisarg)(args)
讲fn的this指向this指向thisarg，并执行args

## buildURL(url, params, paramsSerializer)

如果有参数三，则paramsSerializer（params）
如果检测出来params已经经过处理，就先不动，
其他情况下，就会对param进行处理，

最后，将的出来的结果与url粘合，返回

## combineURLs(baseURL, relativeURL)

将参数中的\替代为‘’，并且在两个参数中加上\

## cookie.js

### 标准浏览器环境下：
write(name, value, expires, path, domain, secure)
```
//查看chromedevtool后发现，cookie的值可以分为name，value，domain，path，expires，size，http，secure，samesite，大部分已经在这个函数中涉及到了
function write(name, value, expires, path, domain, secure) {
        var cookie = [];
        cookie.push(name + '=' + encodeURIComponent(value));//将value编码

        if (utils.isNumber(expires)) {
          cookie.push('expires=' + new Date(expires).toGMTString());//将expires转换为格林威治时间的字符串
        }

        if (utils.isString(path)) {
          cookie.push('path=' + path);
        }//插入path

        if (utils.isString(domain)) {
          cookie.push('domain=' + domain);
        }//插入域名

        if (secure === true) {
          cookie.push('secure');
        }//设置secure，if true ，只能通过https传输

        document.cookie = cookie.join('; ');
      }
```
      
  ```
      read: function read(name) {
        var match = document.cookie.match(new RegExp('(^|;\\s*)(' + name + ')=([^;]*)'));
        return (match ? decodeURIComponent(match[3]) : null);//在正则中，直接match一个文本只能获得一个len为1的列表，用括号抱起来之后就不一样了
      }
```
      
```
      remove: function remove(name) {
        this.write(name, '', Date.now() - 86400000);
      }//将时间往前推一天就能删除cookie了，因为过期了
```
### 不支持非标准浏览器环境下（web workers, react-native），web workers指的是运行在后台的js代码

## isAbsoluteURL(url)
用正则
检测url是不是绝对url，这里只能检测前面是http：//,//  这一类的

## isUrlSameOrigin.js

只能在浏览器环境下使用
将传入的requestURL植入到一个《a》节点中
通过比对两边（window.location,传入的requestURL）的协议和host


这里有个resolveURL函数，可以传入requestURL，输出下面一个对象,这个对象包含了URL中的信息
```
{
        href: urlParsingNode.href,//href
        protocol: urlParsingNode.protocol ? urlParsingNode.protocol.replace(/:$/, '') : '',//协议
        host: urlParsingNode.host,
        search: urlParsingNode.search ? urlParsingNode.search.replace(/^\?/, '') : '',//问号后的内容
        hash: urlParsingNode.hash ? urlParsingNode.hash.replace(/^#/, '') : '',
        hostname: urlParsingNode.hostname,
        port: urlParsingNode.port,//端口
        pathname: (urlParsingNode.pathname.charAt(0) === '/') ?
                  urlParsingNode.pathname :
                  '/' + urlParsingNode.pathname
      };
```

## normalizeHeaderName(headers, normalizedName)
这个函数的意思是迭代headers，如果header的键和normalizedName不一样，但是双方都大写了之后又一样了，就重新创建一个键值对normalizedName:value,对比时有冲突的那个name:value就删除掉

```
function normalizeHeaderName(headers, normalizedName) {
  utils.forEach(headers, function processHeader(value, name) {
    if (name !== normalizedName && name.toUpperCase() === normalizedName.toUpperCase()) {
      headers[normalizedName] = value;
      delete headers[name];
    }
  });
}
```

## parseHeaders(headers)

无论headers是数组还是对象，都用split分割开，经过substr，trim，tolowercase，再组装成一个数组

##  spread(callback)
这个是这么用的：
```
spread(fn)(args)
```
这样写大概是为了能够分两次放入参数


