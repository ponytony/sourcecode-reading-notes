## bindActionCreators不写了,太鸡肋，而且挺简单的

## applyMiddleware(...middlewares)
Middleware是为了使数据做一样的事情


enhencer是给store增加功能

applyMiddleware就是一个enhencer，增强了dispatch
applyMiddleware（middlewares）（creatstore）（reducer, preloadedState, enhancer）

```
export default function applyMiddleware(...middlewares) {
  return (createStore) => (reducer, preloadedState, enhancer) => {
    const store = createStore(reducer, preloadedState, enhancer)//生成一个store，内含多种方法和数据
    let dispatch = store.dispatch//取出dispatch方法
    let chain = []

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)//为什么不写成store.dispatch?
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))//从middlewares中取出值，并且对上面刚刚定义的middlewareAPI进行改造
    dispatch = compose(...chain)(store.dispatch)//用...将chain分解，再一个个执行

    return {
      ...store,
      dispatch
    }
  }
}
```

这是网上抄来的例子：
```
<!DOCTYPE html>
<html>
<head>
  <script src="//cdn.bootcss.com/redux/3.5.2/redux.min.js"></script>
</head>
<body>
<script>
/** Action Creators */
function inc() {
  return { type: 'INCREMENT' };
}
function dec() {
  return { type: 'DECREMENT' };
}

function reducer(state, action) {
  state = state || { counter: 0 };

  switch (action.type) {
    case 'INCREMENT':
      return { counter: state.counter + 1 };
    case 'DECREMENT':
      return { counter: state.counter - 1 };
    default:
      return state;
  }
}

function printStateMiddleware(middlewareAPI) {
  return function (dispatch) {
    return function (action) {
      console.log('dispatch 前：', middlewareAPI.getState());
      var returnValue = dispatch(action);
      console.log('dispatch 后：', middlewareAPI.getState(), '\n');
      return returnValue;
    };
  };
}

var enhancedCreateStore = Redux.applyMiddleware(printStateMiddleware)(Redux.createStore);
var store = enhancedCreateStore(reducer);

store.dispatch(inc());
store.dispatch(inc());
store.dispatch(dec());
</script>
</body>
</html>
```