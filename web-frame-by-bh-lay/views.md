# views.js
这个模块没仔细看，大致的作用是将一些句柄和一些component文件做一个转换
## import
```
var fs = require('fs'),
    component = require('./component'),
    utils = require('./utils/index.js'),
    baseRoot = './views/';
```

## getcomponentconfig
这个模块输入文本，匹配include语句，将每个include标签内的name=XXX转换为{}，最后返回的是一个[{},{},{}...]
```
function getComponentsConfig(input){
  var strArray = input.match(/\<include(.+?)\/>/g) || [],
      confArray = [];
  strArray.forEach(function(item,index){
    var data = {};
    //过滤多余的字符
    item = item.replace(/^<include\s+|"|'|\s*\/>$/g,'');
    //分离参数
    var dataArray = item.split(/\s+/) || [];

    dataArray.forEach(function(it){
      var itemSplit = it.split(/=/);
      var key = itemSplit[0];
      var value = itemSplit[1];
      data[key] = value;
    });
    confArray.push(data);
  });
  return confArray;
}
```

## replacecomponent
//这个函数中会将一个html文件中的include标签全部替换成对应的component
```
function replaceComponent(temp,callback){
  var need_temp = getComponentsConfig(temp),
      temp_result = {},
      over_count = 0;

  var total = need_temp.length;

  //没有用到components
  if(total == 0){
    callback(null,temp);
  }else{
    for(var i=0;i<total;i++){
      (function(i){
        var data = need_temp[i];
        var name = data.name;
        component.get(name,data,function(err,componentStr){
          temp_result[name] = componentStr;
          all_callBack();
        });//这个get函数其实就是将文件中的include语句替换为./component中的文件，有对应的脚本文件就执行脚本，没有的直接替换
      })(i);//闭包
    }
  }
  function all_callBack(){
    over_count++;
    if(over_count == total){
      var html = temp.replace(/\<include\s+name\s*=\s*(?:"|')(.+?)(?:"|')([^\/])*\/>/g,function(includeStr,name){
        return temp_result[name] || includeStr;//到末尾的时候返回
      });//frontend文件夹中的html中的include标签
      callback(null,html);
    }
  }
}
```

## export 

//juicer这歌模块没有什么了解，应该也是对html文件做一个转换吧
```
module.exports = function(URI,data,callback){
  var realPath = baseRoot + URI,//../views
      data = data || {};
  //增加文件配置
  data.frontEnd = this.config.frontEnd;//这个函数没有使用call调用，所以config在哪里？？？，这个frontend包含了一个图床地址   

  //读取模版
  fs.readFile(realPath + '.html', "utf8",function(err,fileStr){
    if(err){
      callback && callback(err);
      return;
    }
    //替换变量
    fileStr = utils.juicer(fileStr,data);//使用juicer

    //解析模版的component
    replaceComponent(fileStr,function(err,txt){
      callback && callback(err,txt);
    });
  });
};
```
