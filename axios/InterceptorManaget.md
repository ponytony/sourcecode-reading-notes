## InterceptorManager

通过handler数组来管理拦截器，源码还是很简单的。

```
function InterceptorManager() {
  this.handlers = [];
}

```

```
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1;//返回比当前长度小1的数值，应该是用来做index的
};
```

```
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;//取消的时候，是将数据变成null
  }
};
```

```
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};//执行数组
```
