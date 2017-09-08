## createError和enhanseError

```
function createError(message, config, code, request, response) {
  var error = new Error(message);//赋值error，是一个e
  return enhanceError(error, config, code, request, response);
};//不是高级函数，也不用去关心要分几次执行
```
    
```
enhanceError(error, config, code, request, response) {
  error.config = config;
  if (code) {
    error.code = code;
  }
  error.request = request;
  error.response = response;
  return error;//最后这个error有了config,code,request,response
};
```
