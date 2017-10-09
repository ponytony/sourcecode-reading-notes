```
var fs = require('fs'),
    zlib = require("zlib"),
    parse = require('./utils/parse.js');

//这个模块的用处就是从request中获取cookie，ip等信息，并传出相应的response（write），其他的几个方法用于获取ip，cookie，设置cookie或者session
//统一返回客户端信息方法
function send (status,headers,content){
  
  var _this = this,
      headers = headers || {};
  headers['server'] = 'nodejs';
  headers['Connection'] = 'keep-alive';
  headers['Content-Encoding'] = 'gzip';
  this.response.statusCode = status;
  for(var i in headers){
    this.response.setHeader(i,headers[i]);//应为这个函数是用call调用的，所以这个this其实是从调用方传入的，因此可以有setheader方法9
  }
  var content = content || null;
  
  if(content){
    zlib.gzip(content, function(err, result) {
      _this.response.end(result);//？？？为什么用——this
    });
  }else{
    this.response.end();//该方法会通知服务器，所有响应头和响应主体都已被发送，即服务器将其视为已完成
  }
}
/**
 * response json data
 *   data can be object or string
 */
function sendJSON( data ){
  var json_str = typeof(data) == 'string' ? data : JSON.stringify(data);
  
  send.call(this,200,{
    'Content-Type' : 'application/json',
    'charset' : 'utf-8',
  },json_str);
}
/**
 * response jsonp data
 *   data can be object or string
 */
function sendJSONP( data ){
  var json_str = typeof(data) == 'string' ? data : JSON.stringify(data),
      callbackName = this.url.search.callback || 'jsonpCallback';//？？？
  send.call(this,200,{
    'Content-Type' : 'application/x-javascript',
    'charset' : 'utf-8',
  },callbackName + '(' + json_str + ');');
}
/**
 * response html page
 */
function sendHTML(status,content){
  send.call(this,status,{
    'Content-Type' : 'text/html',
    'charset' : 'utf-8'
  },content);
}

/**
 * 单次连接类
 *
 */
function CONNECT(req,res,session_factory){
  this.url = parse.url(req.url);//返回一个包含path，search，filename，pathnode，root的obj
  this.request = req;
  this.response = res;
  this._session = null;
  this._session_factory = session_factory;
  //是否已向客户端发送信息体
  this._sended = false;
}
/**  //向客户端发送信息
 *  connect.write('json',{
 *    code:1,
 *    msg:'msg text !'
 *  });
 *  connect.write('html,200,'html text');
 *  connect.write('define',200,{
 *    content-type:'text/html'
 *  },'<html></html>');
 *  connect.write('notFound','not found');
 *  connect.write('error','something wrong');
 */
CONNECT.prototype['write'] = function(type,a,b,c){
  if(this._sended){
    //FIXME runing here maybe somthing wrong
    return;
  }
  this._sended = true;
  switch(type){
    case 'json'://data
      sendJSON.call(this,a);
    break;
    case 'jsonp'://data
      sendJSONP.call(this,a);
    break;
    case 'html'://status,content
      sendHTML.call(this,a,b);
    break;
    case 'define':
      send.call(this,a,b,c);//status,headers,content
    break;
    case 'notFound'://content
      sendHTML.call(this,404,a);
    break;
    case 'error'://,content
      sendHTML.call(this,500,'<h1>' + a + '</h1><p>hello! my name is BUG !</p>');
    break;
  }
};
/**
 *  //获取完整cookie
 *  connect.cookie();
 *
 *  //获取特定cookie
 *  connect.cookie('sessionverify');
 *
 * //写入cookie
 *  connect.cookie({
 *    'session_verify' : sessionID,
 *    'path' : '/',
 *    'Max-Age' : 60*60*24*2
 *  },{
 *    'UID':'23w',
 *    'path':'/admin'
 *  }); 
 */
CONNECT.prototype['cookie'] = function(input){
    var cookieInRequest = this.request.headers.cookie || '';
  if(typeof(input) == 'object'){
    //写入cookie
    var cookieObj = arguments;
    for(var i in cookieObj){
      var cookie_this = cookieObj[i];
      var cookie_str = '';
      for(var i in cookie_this){
        cookie_str += i + '=' + cookie_this[i] +';';
      }
      this.response.setHeader('Set-Cookie',cookie_str);  
    }
  }else if(typeof(input) == 'string'){
        //获取特定cookie
    return parse.cookie(cookieInRequest)[input];
  }else{
    //获取完整cookie
    return parse.cookie(cookieInRequest);//parse.cookie返回一个cookie的键值对
  }
};
/**
 *  session相关操作
 *
 *  //启动session模块
 *  connect.session(function(methods){
 *    //获取session
 *    var UID = methods.get('UID');
 *    //设置session
 *    methods.set({
 *      'UID' : 324
 *    });
 *    //校验权限
 *    var canIDo = methods.power(23);
 *  });
 */
CONNECT.prototype['session'] = function(callback){
  var me = this;
  if(this._session){
    //session已被打开,直接使用
    callback&&callback(this._session);
  }else{
    //启用session
    var cookie = this.cookie();//获取全部cookie
    this._session = new this._session_factory(cookie,function(cookieObj){
      me.cookie(cookieObj);//获取指定cookie
    },function(){
      callback&&callback(me._session);//callback先放一边？？？
    });
  }
};
//  var IP = req['connection']['remoteAddress'];
CONNECT.prototype['ip'] = function(){
  return this.request['headers']['x-forwarded-for'] || this.request['connection']['remoteAddress'];
};
//  var userAgent = req['headers']['user-agent'];

module.exports = CONNECT;
```