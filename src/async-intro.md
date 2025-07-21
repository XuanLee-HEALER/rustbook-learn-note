# 前言

## 什么是异步编程以及为什么你要使用它

在并发编程中，程序在同一时间做许多事（看起来是这样）。使用线程是并发编程的一种形式。线程中的代码以顺序运行的风格写成，操作系统并发执行这些线程。对于异步编程来说，并发完全发生在你的程序中（操作系统不参与）。一个异步运行时（Rust中的一个crate）和程序员使用`await`关键字来显式让出控制权的方式结合起来管理异步任务

因为没有操作系统的参与，异步世界的上下文切换非常快。而且异步任务的内存占用比操作系统线程更低。因此异步编程非常适合需要处理非常多并发任务，并且这些任务会花费大量时间等待（例如客户端响应或者IO操作）的系统。异步编程也很适合内存数量有限并且不提供线程的操作系统的微控制器

异步编程也给了程序员在任务的执行方式上更细粒度的控制权（并行和并发级别、控制流、排布等）。这意味着异步编程可以既有表现力，又符合人体工程学。特别是，Rust中的异步编程有强大的取消概念，支持不同风格的并发（使用spawn和它的变体的结构、join、select、for_each_concurrent等）。通过组合和重用产生了timeout、pausing和throttling等概念

## Hello, World

```rust
// Define an async function.
async fn say_hello() {
    println!("hello, world!");
}

#[tokio::main] // Boilerplate which lets us write `async fn main`, we'll explain it later.
async fn main() {
    // Call an async function and await its result.
    say_hello().await;
}
```

## Aysnc Rust开发

Rust的异步特性已经开发了一段时间，但是它还不处于这门语言的“结束”阶段。异步Rust（稳定编译器和标准库中的一部分）是可靠的和高性能的。它应用于最大的技术公司的要求最严苛的环境中。然而还有缺失和粗糙的部分（人体工程学意义上的粗糙，而非稳定性）

目前，使用异步迭代器（流）是大多数用户觉得粗糙的地方。一些在特质中异步的应用也没有很好的支持。还没有对于异步解构的好的解决方案

异步Rust正在处理这些事，如果你想跟踪开发进程，你可以检查Async Working Group的主页，里面有他们的路线图。你也可以阅读Rust项目中的异步项目目标
