```

//不好说 工厂类中的callback和writecookie是什么方法还不清楚
var fs = require('fs');

//保存session
function save_session(){
  var pathname = this.path;
  var data = JSON.stringify(this);//???这个data不是一系列的原声属性和方法？
  var status = fs.writeFile(pathname,data,function( err ){//写入文件中
    if( err ){
      console.error('write seesion file error', err );
    }
  });
}
//生成session id
function createSessionID(){
  return new Date().getTime() + Math.ceil(Math.random()*1000);
}
//检测是否为正常session id
function isNormalSessionID(ID){
  if(ID == +ID && ID.length > 3){
    return true;
  }else{
    return false;
  }
}


var proto = {
  set : function (param){
    for(var i in param){
      if(i == 'power_data'){
        this.power_code = param[i];
      }else{
        this.data[i] = param[i];
      }
    }
    save_session.call(this); 
  },
  get : function (name){
    var this_session = this.data;
    var getData = this_session[name] || null;
    return getData;
  },
  power : function (code){
    if(code && this.power_code[code]=='1'){
      return true;
    }else{
      return false;
    }
  }
};

/**
 * session工厂函数
 *
 **/
function session_factory(param){
  param = param || {};
  var session_root = param.root;
  //检测是否配置session存储目录
  if(!session_root){
    console.error('need seesion path');
    return;
  }
  /**
   * session主类
   *
   **/
  function SESSION(cookieObj,writeCookie,callback){
    //检测session id 或创建
    this.sessionID = isNormalSessionID(cookieObj['session_verify']) ? cookieObj['session_verify'] : createSessionID();
    this.path = session_root + this.sessionID + '.txt';
    this.power_code = [];

    var that = this;
    // find sessionID in session library
    fs.exists(this.path, function(exists) {
      if(exists){
        //read session file
        fs.readFile(that.path,'UTF-8',function(err,file){
          if(err){
            console.error('read session file error',err);
            callback && callback(err);
            return;
          }
          var JSON_file = JSON.parse(file);
          for(var i in JSON_file){
            that[i] = JSON_file[i];
          }
          callback&&callback();
        });
      }else{
        //create session file
        that.time_cerate = new Date();

        that.data = {
