```
var fs = require('fs'),
    zlib = require("zlib");

//处理静态资源，是否有静态资源，有没有改变，随着状态的不同会返回不同的状态码和不同的数据
function Filer(param){//传入的是一个config.static
  this.staticFileRoot = param.root;//是个相对链接
  this.mime = param.mime;
  this.maxAge = param.maxAge;
}
Filer.prototype.read = function(path,req,res,notFound) {
  //匹配文件扩展名
  var maxAge = this.maxAge;
  var pathname_split = path.match(/.\.([^.]+)$/);
  var ext = pathname_split ? pathname_split[1] : null;

  var realPath;
  //add a default files for directory
  if(ext == null) {
    ext = 'html';
    realPath = this.staticFileRoot + path + '/index.html';//指向index
  }else{
    realPath = this.staticFileRoot + path;//指向静态资源的地址
  }
  var content_type = this.mime[ext]||'unknown';

  fs.exists(realPath, function(exists) {
    if(!exists){
      notFound();//这是参数中传入的回调函数，如果用error就一目了然了
      return ;
    }//？？？功能似乎与fs.stat的error处理重复了

    fs.stat(realPath, function(err, stat) {//这个函数返回文件的状态对象，包括各种时间，uid，gid等
      if(err) {
        /**
         * 500 server error 
         */
        res.writeHead(500);
        res.end('500');
        return;
      }
      var lastModified = stat.mtime.toUTCString();
      //检验文件的最后修改时间和ajax传上来的if-modified-since是否一致
      if(req.headers['if-modified-since'] && (lastModified == req.headers['if-modified-since'])) {
          // 去读缓存
        res.writeHead(304);
        res.end();
      } else {
        var expires = new Date(new Date().getTime() + maxAge * 1000),
            headers = {
              'Content-Type': content_type,
              'Expires' : expires.toUTCString(),
              'Cache-Control' : "max-age=" + maxAge,
              'Last-Modified' : lastModified,
              'Access-Control-Allow-Origin' : "*"
            },
            acceptEncoding = req.headers['accept-encoding'],
            stream = fs.createReadStream(realPath),
            gzipStream = zlib.createGzip();
        
        if(acceptEncoding && acceptEncoding.indexOf('gzip') != -1) {
          headers['Content-Encoding'] = 'gzip';
          res.writeHead(200, headers);//修改header
          stream.pipe(gzipStream).pipe(res);//压缩
        }else{
          res.writeHead(200, headers);
          stream.pipe(res);
        }
      }
    });
    
  });
};
module.exports = Filer;
```