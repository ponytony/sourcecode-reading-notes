[文章来源]（https://zhuanlan.zhihu.com/p/27659059
反正我要谢谢这个作者写了这么多好文章

[文章来源]（https://zhuanlan.zhihu.com/p/27659059
反正我要谢谢这个作者写了这么多好文章

## 递归
### 希望分次处理一堆数据，又懒得引入Promise的方法
```
var ids = [34112, 98325, 68125];
(function sendRequest(){
    var id = ids.shift();//切割数组
    if(id){
        $.ajax({url: "/get", data: {id}}).always(function(){
            //do sth.
            console.log("finished");
            sendRequest();//这个是关键，调用了自身
        });
    } else {
        console.log("finished");
    }
})(); 
```

### DOM树
#### 使用递归
```
function getElementById(node, id){
    if(!node) return null;//递归到底部还没找到节点就return null
    if(node.id === id) return node;//找到了节点，这两个if是用来检测传上来的参数的
    for(var i = 0; i < node.childNodes.length; i++){//还没找到就继续找吧
        var found = getElementById(node.childNodes[i], id);//这个found参数应该是从递归的底层一层层传上来的
        if(found) return found;
    }
    return null;
}
getElementById(document, "d-cal");
```

我自己写的，挺烂的：
```
 var result

    function getElementById(node, id){
        for(var i = 0; i < node.childNodes.length; i++){
            if(node.childNodes[i].id===id){
                result=node.childNodes[i]
                return
            }
            if(node.childNodes[i].id!==id){
                getElementById(node.childNodes[i], id)
            }

        }
    }
    getElementById(document,'errorproject')
    console.log(result)
    
```

#### 不使用递归

这里的注释中，兄弟节点指的都是指定节点的下一个节点
```
function nextElement(node){
    if(node.children.length) {
        return node.children[0];//如果有子阶段，就返回子节点
    }
    if(node.nextElementSibling){
        return node.nextElementSibling;//探到最底部的一个子节点时，如果有相邻节点，返回这个子节点的相邻元素
    }
    while(node.parentNode){
    //同级的相邻元素都已经遍历了一遍
        if(node.parentNode.nextElementSibling) {//如果父节点有兄弟节点
            return node.parentNode.nextElementSibling;//返回这个兄弟节点
        }
        node = node.parentNode;//node.parentNode.nextElementSibling到底了，就直接改变node继续查找，直到找到有兄弟节点的那一个
    }
    return null;
}
```
```
function getByElementId(node, id){
    //遍历所有的Node
    while(node){
        if(node.id === id) return node;
        node = nextElement(node);//这里如果不符合条件，就换一个node
    }
    return null;
}
```


### 复杂选择器

```
function nextElement(node){
        if(node.children.length) {
            return node.children[0];//如果有子阶段，就返回子节点
        }
        if(node.nextElementSibling){
            return node.nextElementSibling;//探到最底部的一个子节点时，如果有相邻节点，返回这个子节点的相邻元素
        }
        while(node.parentNode){
            //同级的相邻元素都已经遍历了一遍
            if(node.parentNode.nextElementSibling) {//如果父节点有兄弟节点
                return node.parentNode.nextElementSibling;//返回这个兄弟节点
            }
            node = node.parentNode;//node.parentNode.nextElementSibling到底了，就直接改变node继续查找，直到找到有兄弟节点的那一个
        }
        return null;
    }
```
```
    function match(node, selector){
        if(node === document) return false;
        switch(selector.matchType){
            //如果是类选择器
            case "class"://如果节点找不到selector.value，返回-1，相当于false
                return node.className.trim().split(/ +/).indexOf(selector.value) >= 0;

            //如果是标签选择器
            case "tag"://标签名和selector.value不一样就继续找，返回false
                return node.tagName.toLowerCase() === selector.value. toLowerCase();

            default:
                throw "unknown selector match type";
        }
    }
```
```
    function nextTarget(node, selector){//处理selector的relation，根据relation返回当前节点的父节点或者兄弟节点，以及返回hasnext
        if(!node || node === document) return null;
        switch(selector.relation){
            case "descendant":
                return {node: node.parentNode, hasNext: true};
            case "child":
                return {node: node.parentNode, hasNext: false};
            case "sibling":
                return {node: node.previousSibling, hasNext: true};
            default:
                throw "unknown selector relation type";
            //hasNext表示当前选择器relation是否允许继续找下一个节点
        }
    }
    
 ```
```

    function querySelector(node,selector){
        while(node){或者整个dom全部遍历了还没找到，node会变成null，结束循环返回null（最后第二行），
            var currentNode=node;
            if(!match(node,selector[0])){//找不到就继续找，找到了就继续后面的代码，后面的代码不成功，就继续找符合的node
                node=nextElement(currentNode)
                continue//如果暂时找不到第一个节点，继续循环，下面的代码先放着
            }//找到了第一个节点，node已经变成了那个节点
            var next=null;
            for(var i=0;i<selector.length-1;i++){//遍历selecter
                var matchIt=false;
                do{//这里为什么用do。。。while，是因为这样可以先执行代码再检查判断条件。虽然我觉得直接用while好像也可以
                    next=nextTarget(node,selector[i]);
                    node=next.Node;//这个已经是原来搜索出来的节点的父节点，在do while内一直冒泡到if（！node）
                    if(!node){//已经是最上面的节点了，就break
                    //还有就是这个node是整个函数的变量，如果这个node已经探索到了顶部，还没找到节点，接下去的for循环也就没有意义了，matchit都会返回false
                        break
                    }
                    if(match(node,selector[i+1])){//一直往上冒泡，知道符合，如果没有节点了，就按照上面的那个if（！node），break，然后再从前面那个while（node）开始，利用currentnode继续循环
                        matchIt=true;
                        break
                    }
                }while(next.hasNext);
                if(!matchIt){
                    break
                }
            }
            if(matchIt&&i===selector.length-1){//符合条件,这里的i===selector.length-1是为了确定是不是经过了最后一个循环，matchIt还是true
                return currentNode//返回想要的节点
            }
            node=nextElement(currentNode)//这里是为了在找不到想要的祖先节点的情况下，重新设置while的条件，继续循环
        }
        return null//在整个node都循环了一遍之后还是找不到的情况下，返回
    }
```

#### 原作者的selector是这么写的，第一行是最底部的元素，先是找最底部的元素，找到后一层层冒泡，去找符合第二层的元素，第三层同理，如果冒泡的时候失败了，就让node=currentnode，还可以继续while（node）这个循环
```
var selectors = [
{relation: "descendant",  matchType: "class", value: "copyright-content"},
{relation: "child",       matchType: "tag",   value: "div"},
{relation: "subSelector", matchType: "class", value: "mls-info"}];
```

### 重复值查询

#### 方法1 两个for
```
var lastHouses = [];
filterHouse: function(houses){
    if(lastHouses === null){
        lastHouses = houses;
        return {
            remainsHouses: [], 
            newHouses: houses
        };  
    }   
    var remainsHouses = [], 
        newHouses = []; 

    for(var i = 0; i < houses.length; i++){
        var isNewHouse = true;
        for(var j = 0; j < lastHouses.length; j++){
            if(houses[i].id === lastHouses[j].id){
                isNewHouse = false;
                remainsHouses.push(lastHouses[j]);
                break;
            }   
        }   
        if(isNewHouse){
            newHouses.push(houses[i]);
        }   
    }   
    lastHouses = remainsHouses.concat(newHouses);
    return {
        remainsHouses: remainsHouses,
        newHouses: newHouses
    };  
}
```

#### 方法2 set
```
var lastHouses = new Set();
function filterHouse(houses){
    var remainsHouses = [],
        newHouses = [];
    for(var i = houses.length - 1; i >= 0; i--){
        if(lastHouses.has(houses[i].id)){//set 的方法has
            remainsHouses.push(houses[i]);
        } else {
            newHouses.push(houses[i]);
        }
    }
    for(var i = 0; i < newHouses.length; i++){
        lastHouses.add(newHouses[i].id);//用add方法添加到lasthouses
    }
    return {remainsHouses: remainsHouses, 
            newHouses: newHouses};
}
```

es6的写法
```
function filterHouse(houses){
    var remainsHouses = [],
        newHouses = []; 
    houses.map(house => lastHouses.has(house.id) ?//用了map和三元运算 remainsHouses.push(house) 
                        : newHouses.push(house));
    newHouses.map(house => lastHouses.add(house.id));
    return {remainsHouses, newHouses};
}
```

### 方法三 map
```
var lastHouses = new Map();
function filterHouse(houses){
    var remainsHouses = [],
        newHouses = [];
    houses.map(house => lastHouses.has(house.id) ? remainsHouses.push(house)
			: newHouses.push(house));
    newHouses.map(house => lastHouses.set(house.id, house));
    return {remainsHouses, newHouses};
}
```

### 数组去重

#### 方法1 使用Set + Array
```
function uniqueArray(arr){
    return Array.from(new Set(arr));//利用set的特性
}
```

#### 方法2 使用splice
```
function uniqueArray(arr){
    for(var i = 0; i < arr.length - 1; i++){
        for(var j = i + 1; j < arr.length; j++){//两次循环arr，这里的j这么设定是因为节约性能，j<i的部分已经检查过了
            if(arr[j] === arr[i]){
                arr.splice(j--, 1);//之前还在奇怪为什么这里可以动态改变arr的值，仔细想想是应为我喜欢设定一个len的值，这个j--也是为了在删除值后把j-1
            }
        }
    }
    return arr;
}
```
#### 方法三 只用Array 
```
function uniqueArray(arr){//用了indexof和push
    var retArray = [];
    for(var i = 0; i < arr.length; i++){
        if(retArray.indexOf(arr[i]) < 0){
            retArray.push(arr[i]);
        }
    }
    return retArray;
}
```
#### 方法四
```
goog.array.removeDuplicates = function(arr, opt_rv, opt_hashFn) {
  var returnArray = opt_rv || arr;
  var defaultHashFn = function(item) {
    // Prefix each type with a single character representing the type to
    // prevent conflicting keys (e.g. true and 'true').
    return goog.isObject(item) ? 'o' + goog.getUid(item) :
                                 (typeof item).charAt(0) + item;//这一段看不懂
  };
  var hashFn = opt_hashFn || defaultHashFn;

  var seen = {}, cursorInsert = 0, cursorRead = 0;
  while (cursorRead < arr.length) {
    var current = arr[cursorRead++];
    var key = hashFn(current);
    if (!Object.prototype.hasOwnProperty.call(seen, key)) {
      seen[key] = true;
      returnArray[cursorInsert++] = current;
    }
  }
  returnArray.length = cursorInsert;
};
```

