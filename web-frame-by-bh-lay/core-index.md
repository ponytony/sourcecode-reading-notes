## web框架 的实现

```
var http = require('http'),
    connect = require('./connect.js'),
    views = require('./views.js'),
    cache = require('./cache.js'),
    staticFile = require('./staticFile.js'),
    url_redirect = require('../conf/301url'),
    config = require('../conf/app_config.js'),
    utils = require('./utils/index.js'),
    session_factory = require('./session.js');

function APP(){
  var me = this;
  
  this.MAPS = {};//maps里面装的是.get方法中获取的url和callback的键值对
  this.fileReader = new staticFile(config.static);//引入staticFile类
  // server start
  var server = http.createServer(function (req,res) {
    if(isNormalVisitor(req)){//检测是否是正常用户，就是检测req.url中是否有../如果有，返回true？？？但是为什么要返回状态码500，内部服务器错误；
      res.writeHead(500);
      res.end('hello I\'m bh-lay !');
      return;
    }
    //实例化一个connect对象
    var new_connect = new connect(req,res,me.session),//prototype   
        path = new_connect.url,
        pathNode = pathParser(path.pathname),
        result = findUrlInMaps(pathNode,me.MAPS);//首先要知道maps中储存的是get方法设置的键（path）值（callback）对，那么这个findinmap函数就是用来获取这个键值对

    if(result){
      //第一顺序：执行get方法设置的回调
      var data = result.data;
      result.mapsItem.call(this,data,new_connect);//mapsitem是个函数
    }else{
      //第二顺序：使用静态文件
      me.fileReader.read(path.pathname,req,res,function(){
        //第三顺序：查找301重定向
        if(url_redirect[path.pathname]){
          new_connect.write('define',301,{
              'location' : url_redirect[path.pathname]
          });
        }else{
          //404了
          me.views('system/404',{
              content : '文件找不到啦！'
          },function(err,html){
              new_connect.write('notFound',html);
          });
        }
      });
    }
  });

  server.listen(config.port, 0,0,0,0);//监听端口和host
  console.log('server start with port ' + config.port);
};

/**
 * 设置前端请求路径
 */
APP.prototype.get = function(urls,callback){
  var me = this,
      routerNames = [].concat(urls);
  
  if(typeof(callback) != 'function'){
    return;
  }
  routerNames.forEach(function(url,a,b){
    if(typeof(url) != 'string'){
      return;
    }
    me.MAPS[url] = callback;
  });
};

APP.prototype.views = views;//主要用到component的文件夹，用来解析页面
APP.prototype.cache = new cache({
  useCache: config.cache.use ? true : false,
  max_num: config.cache.max_num,
  root: config.cache.root
});//cache用来设置储存解析后页面的类，储存到一个temp文件夹中，有use和clear方法
APP.prototype.session = new session_factory({
  root : config.session.root
});
APP.prototype.utils = utils;//一些加密方法，parse方法
APP.prototype.config = config;//默认的设置

module.exports = APP;
```