## 项目结构
```
├── utils/
│     ├── warning.js # 打酱油的，负责在控制台显示警告信息
├── applyMiddleware.js
├── bindActionCreators.js
├── combineReducers.js
├── compose.js
├── createStore.js
├── index.js # 入口文件
```
这个项目难度不大，英文注释挺不错的，英语好的话直接看英文就行了

## 数据流向
- 初始数据先放入creatstore（）中，在闭包中创建store，
- 组件的js代码先发出一个action
- 检查action的type之后，根据switch的case来改变store的值
- 在react中，是通过react-redux提供的函数来改变react的prop值