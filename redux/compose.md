   ## compose.js
   ```
   compose(funcA, funcB, funcC) 形象为 compose(funcA(funcB(funcC())))
   ```
   源码：
   ```
   export default function compose(...funcs) {
     if (funcs.length === 0) {
       return arg => arg
     }
   
     if (funcs.length === 1) {
       return funcs[0]
     }//参数检验
   
     return funcs.reduce((a, b) => (...args) => a(b(...args)))//使用reducer从后往前执行参数
   }
   ```
   这个挺简单的,可以用来增强middleware