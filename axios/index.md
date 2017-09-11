## axios的config流向
- 将config，method，default，url，data合并为一个config
- 将config通过request拦截器处理
- 进入dispatchrequest，并进行一些处理之后，交给一个ajax的配适器
- 在配适器内将一些config赋值给request（ajax初始化实例），send
- 返回response之后通过response拦截器的处理
- 最后返回一个promise，promise的数据格式按照ajax配适器的规定
- 接下去就能使用response了