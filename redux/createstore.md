## createStore(reducer, initialState, enhancer)
用于创建store

源码：
```
import isPlainObject from 'lodash/isPlainObject'
import $$observable from 'symbol-observable'


export const ActionTypes = {
  INIT: '@@redux/INIT'//redux的私有常亮，自己不用管，如果有不知道的action或者未定义的state，就会返回当前的state或者初始化的state
}


export default function createStore(reducer, preloadedState, enhancer) {
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }//这是用于当preloadedstate不存在时将第二个参数改为enhencer

  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }/当第三个参数有值但不是function时thirow error

    return enhancer(createStore)(reducer, preloadedState)//用enhencer增强creatstore，然后执行参数1,2
  }

  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }

  let currentReducer = reducer
  let currentState = preloadedState
  let currentListeners = []
  let nextListeners = currentListeners//复制一个listen列表，不过为什么只是个shadow clone？因为deepclone在ensureCanMutateNextListeners中
  let isDispatching = false

  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()//deepclone
    }
  }

  
  function getState() {
    return currentState
  }

  
  function subscribe(listener) {//subscribe（）是订阅，subscribe()()是取消订阅
    if (typeof listener !== 'function') {
      throw new Error('Expected listener to be a function.')
    }

    let isSubscribed = true//这个变量是subscribe自己的，不是全局那个

    ensureCanMutateNextListeners()//复制，确保安全
    nextListeners.push(listener)//listener列表增加事件

    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }//这是为了保险？无论怎么看，在subscribe函数执行了一次之后，isSubscribed的值都是true啊

      isSubscribed = false

      ensureCanMutateNextListeners()//复制到nextlisteners
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1//移除
    }
  }


  //要注意listener是subscribe函数的参数，action是dispatch函数的参数，subscribe我没用过，看代码应该是用来做中间件的，或者是trigger给别的订阅的对象的吧
  function dispatch(action) {
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
        'Use custom middleware for async actions.'
      )
    }//分发action时先检验action是否是一个简单对象（键值对）

    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
        'Have you misspelled a constant?'
      )
    }//action必须有一个type属性

    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }//检验isdispatch，如果执行其他函数出了问题，isdispatch可能会变成true

    try {
      isDispatching = true//这个是全局的那个isDispatching
      currentState = currentReducer(currentState, action)//所以我们写的reducer是  reducer=(state,action)=>{switch...
    } finally {
      isDispatching = false//将状态又变回了false
    }

    const listeners = currentListeners = nextListeners//currentListeners=nextListeners，再让listeners=currentListeners
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }//将listener全部执行一遍

    return action//dispatch返回action，同参数一样，在这个函数中，action只是被检查了格式，然后在reducer（state，action）中使用了一下，这个返回值我没用到过，
  }

  //当你有多个reducer需要使用时
  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }//类型检测

    currentReducer = nextReducer//更改这个createstore的reducer
    dispatch({ type: ActionTypes.INIT })// 触发生成新的 state 树
  }

  //据说是给可观察/响应式库 的接口，不懂，rxjs也没接触过
  function observable() {
    const outerSubscribe = subscribe
    return {
    //这个是es6的语法，相当于又定义了一个subscribe
      subscribe(observer) {
        if (typeof observer !== 'object') {
          throw new TypeError('Expected the observer to be an object.')
        }
           //subscribe里面定义了一个函数，如果有observer.next,就执行这个，参数是当前state
        function observeState() {
          if (observer.next) {
            observer.next(getState())
          }
        }

        observeState()
        const unsubscribe = outerSubscribe(observeState)//将上一行的结果订阅到listener中
        return { unsubscribe }
      },

      [$$observable]() {
        return this
      }
    }
  }


  dispatch({ type: ActionTypes.INIT })//初始化state树

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
}
```