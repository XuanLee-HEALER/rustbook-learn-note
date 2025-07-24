# rpc

基于Google的RPC框架`tarpc`学习，后补概念内容（learn by doing）

## `tarpc`

### 什么是RPC框架

“RPC”表示“Remote Procedure Call”，一个函数调用，但是产生返回值的过程在别处发生。但一个rpc函数被调用，函数会联系其它的进程来获取函数计算的结果。然后原函数会返回其它进程生成的值

RPC框架是大多数面向微服务架构的基本构建块。`tarpc`和`gRPC`以及`Cap'n Proto`的区别是，它在代码中定义模式（schema），而不是使用单独的语言，例如`.proto`。所以没有单独的编译过程，在不同语言之间没有上下文切换
