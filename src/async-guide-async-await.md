# async和await

async是函数（还有特质）的一个注解。await是表达式上的一个操作符

## Rust的异步概念

### 运行时

异步任务必须被管理和排布，通常任务都会比核心多，它们不能同时运行。当一个停止执行后，另一个必须被选中执行。如果一个任务在等待IO或其它事件，它就不应该被排布，但是当它完成IO之后就应该被排布。这就需要和OS交互来管理IO工作

很多编程语言提供了运行时。通常，这个运行时做了比管理异步任务更多的事情，它可能会管理内存（GC），处理异常，对OS提供一个抽象层，或者就是虚拟机。Rust是一个低级语言，追求最小的运行时开销。因此相比其它语言的运行时，异步运行时的限制很多。有很多方法来设计和实现一个异步运行时，所以Rust让你根据自己的需求来选择运行时，而不是提供一个

一个运行时必须和OS交互来管理异步IO，和运行以及排布任务一样。它必须为任务提供定时器功能（与IO管理有交叉）。对于运行时如何构造没有强制性规则，但是一些术语和指责划分是共通的

* *reactor/event loop/driver*（相同含义的术语）：分发IO和定时器事件，和OS交互，做最低级的执行推进
* *scheduler*：决定什么时候任务可以执行，在哪个OS线程上
* *executor or runtime*：reactor和scheduler的组合，也是运行异步任务面向用户的API，运行时也用来表示整个功能库（例如所有东西都在Tokio的crate，不只是`Runtime`类型表示的Tokio executor）

一个运行时crate通常包括很多工具的特质和函数。它们可能是特质（例如`AsyncRead`）和对IO的实现，对通用IO任务的功能，例如网络或者访问文件系统、锁、管道和其它同步原语，处理时间，处理和OS的工作（例如信号处理），处理futures和streams的工具函数，监控和观察工具

有很多异步运行时可选。它们的排布策略有很大区别，或者对某种任务或领域进行了优化。本指导手册中我们使用Tokio运行时。这是一个通用目的（general purpose）运行时，也是生态中最受欢迎的运行时。对于起步和生产环境都是适合的。一些情况下，你可能想要使用其它运行时获得更好的性能或者写更简单的代码

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
#[tokio::main]
async fn main() { ... }
```

`#[tokio::main]`注解初始化了Tokio运行时，开启了一个异步任务来运行main函数中的代码。之后我们会解释这个注解做了什么，以及如何在不使用它的条件下使用异步代码（更多的灵活性）

### futures-rs和生态

TODO

### futures和tasks

Rust中异步并发的基本单元是future。一个future只是一个普通的老的实现了“Future”特质的Rust对象（一个结构体或者枚举）。一个future代表了一个延迟的计算。意思是这个计算在未来的某个时间点会准备好

我们先不去真的定义future或者直接使用它们。future的一个重要方面是它们可以组合起来构成新的，更“大”的future。“async task”目前是用来描述逻辑序列的执行过程、线程的近义词，但是在程序中被管理而不是OS。对任务（task）的思考通常是有用的，但是Rust本身并没有task的概念，这个术语可以用来描述很多不同的事情。这让人困惑，更糟的是，运行时也有任务的概念，不同的运行时对于任务的理解也有一些细微差别

现在我会让task这个术语的含义更准确。当我使用“task”时，我的意思是一个可能和其它任务并发发生的计算过程的抽象概念。使用“async task”表示相同的事情，但是作为和那些以OS线程实现的作为对比。“runtime的task”表示一个运行时认为的任何类型的任务，“tokio task”表示Tokio的任务概念

Rust中的异步任务只是一个future（通常是由很多其它future组成的“大”future）。一个任务是一个被执行的future。然而也会出现一个future不作为一个运行时的任务被执行。这种类型的future也被看作是一个任务，但不是一个运行时的任务

## 异步函数

async是函数声明的一个修饰符。例如我们可以写`async fn send_to_server(...)`。一个异步函数只是使用`async`关键词声明的函数，它意味着这个函数可以被异步地执行，即调用者可以选择不等待这个函数执行完成就去执行其它函数

更具体地说，当一个异步函数被调用，它的函数体的执行和作为普通函数执行是不一样的。函数体和它的参数被打包进一个future，作为一个真正的结果返回。调用者之后可以决定如何处理这个future（如果调用想“直接”获取结果，那么它会await这个future）

在异步函数中，代码还会照常执行，以顺序的方式，成为异步没有什么影响。你可以从异步函数中调用同步函数，执行过程也是相同的。在异步函数中能做的唯一不一样的是使用await来等待其它异步函数（或者future），这会让出控制权，这样其它任务就可以运行了

## `await`

如果结果立即准备好或者可以不等待就计算完成，那么await只是触发计算并返回结果。然而，如果结果没有准备好，那么await会将控制权传给scheduler，此时其它任务可以继续（协作式多任务）

使用await的语法是`some_future.await`，这是一个后缀关键字，使用`.`操作符。这意味着它可以在方法调用和成员访问上链式调用

```rust
// An async function, but it doesn't need to wait for anything.
async fn add(a: u32, b: u32) -> u32 {
  a + b
}

async fn wait_to_add(a: u32, b: u32) -> u32 {
  sleep(1000).await;
  a + b
}
```

如果我们调用`add(15, 3).await`，它会立即返回。如果我们调用`wait_to_add(15, 3).await`，我们最终获得了相同的答案，但是在等待过程中，其它任务有机会运行

这个例子中，对`sleep`的调用是一个假装做长时间运行任务的占位符，我们必须等待任务。这通常是一个IO操作，结果是从外部源读取的数据或者确认写入到外部目标。读数据类似`let data = read(...).await?`，这个例子中，`await`会导致当前任务在读取发生过程中等待。一旦读取完成（其它任务在读取任务等待的时候完成了一些事情）这个任务会继续。读取结果可能是成功读取到的数据或者一个错误

如果我们调用这些函数但是不使用`.await`，我们不会得到任何答案

调用一个异步函数会返回一个future，它不会在函数中执行代码。此外一个future在它被等待（awaited）之前不会做任何事。这和那些异步函数返回一个立即执行的future的语言不同

这是一个Rust中异步编程的重点。一段时间后它会成为第二天性，但是它会绊倒初学者，特别是由其它语言异步编程体验的人

在Rust中future的一个重要直觉是它们是惰性对象（inert objects）。要做任何工作，它们必须被外力驱动（通常是运行时）

我们非常实际地描述了await（它运行了future，返回结果），但是我们之前讨论的异步任务和并发，await时如何适配这个精神模型的？首先，来考虑完全顺序执行的代码：逻辑上，调用一个函数就是执行函数中的代码（一些变量的赋值），换句话说，当前任务持续执行下一个函数定义的代码块。类似的，在异步上下文中，调用一个非异步函数也是持续执行这个函数。调用一个异步函数会**找到**运行的代码，但是不运行它。await是一个操作符，它继续当前任务的运行或者如果当前任务现在不能继续，就给其它任务机会继续执行

await只能在async上下文中使用，现在来说就是在async函数中（之后会看到更多类型的async上下文）。要理解为什么，记住await可能会把控制权交还给运行时，让另外的任务执行。在异步上下文中只有一个运行时可以将控制权交给它。现在你可以想象运行时像一个全局变量，只能在异步函数中访问

最后，再来看一个await的视角：我们之前提到future可以组合到一起构成“更大的”future。异步函数是定义future的一种方法，await时一种组合future的方式。在future中使用await将future组合进异步函数产生的future中

## 一些`async/await`例子

```rust
// Define an async function.
async fn say_hello() {
    println!("hello, world!");
}

#[tokio::main] // Boilerplate which lets us write `async fn main`, we'll explain it later
async fn main() {
    // Call an async function and await its result.
    say_hello().await;
}
```

main函数附近的模版代码是用来初始化Tokio运行时和创建一个运行异步main函数的任务

`say_hello`是一个异步函数，当我调用它的时候，我们必须跟上`.await`来把它作为当前任务的一部分运行。如果你移除`.await`部分，那么运行这个程序都什么都不会做。调用`say_hello`返回一个future，但是它永远不会执行

```rust
#[tokio::main]
async fn main() -> Result<()> {
    // Open a connection to the mini-redis address.
    let mut client = client::connect("127.0.0.1:6379").await?;

    // Set the key "hello" with value "world"
    client.set("hello", "world".into()).await?;

    // Get key "hello"
    let result = client.get("hello").await?;

    println!("got value from the server; result={:?}", result);

    Ok(())
}
```

对现在为止谈到的所有关于并发、并行和异步的这些例子都是顺序的。调用和等待异步函数并没有引入任何并发，除非有另一个任务在等待任务时被排布。要证明这一点，再看一个例子

```rust
use std::io::{stdout, Write};
use tokio::time::{sleep, Duration};

async fn say_hello() {
    print!("hello, ");
    // Flush stdout so we see the effect of the above `print` immediately.
    stdout().flush().unwrap();
}

async fn say_world() {
    println!("world!");
}

#[tokio::main]
async fn main() {
    say_hello().await;
    // An async sleep function, puts the current task to sleep for 1s.
    sleep(Duration::from_millis(1000)).await;
    say_world().await;
}
```

在打印“hello”和“world”之间，我们让当前任务睡1秒。运行这个程序之后，它首先打印“hello”，然后1秒时间什么都不做，然后打印“world”。这是因为执行一个任务是完全顺序的，如果我们有一些并发，那么在这1秒期间就可以执行其它的任务，例如打印“world”

## 派生任务（spawning tasks）

async和await是以异步任务运行代码的一种方式。await会将当前任务在等待IO或者其它事件时休眠。当休眠发生时，其它任务可以运行，但是其它任务是如何知道的？就像我们使用`std::thread::spawn`来派生一个新任务，我们可以使用`tokio::spawn`来派生一个新的异步任务。注意`spawn`是运行时Tokio的函数，而不是标准库的，因为任务完全是运行时的概念

```rust
use tokio::{spawn, time::{sleep, Duration}};

async fn say_hello() {
    // Wait for a while before printing to make it a more interesting race.
    sleep(Duration::from_millis(100)).await;
    println!("hello");
}

async fn say_world() {
    sleep(Duration::from_millis(100)).await;
    println!("world!");
}

#[tokio::main]
async fn main() {
    spawn(say_hello());
    spawn(say_world());
    // Wait for a while to give the tasks time to run.
    sleep(Duration::from_millis(1000)).await;
}
```

这一次我们并发运行这两个函数（并行）而不是顺序执行。如果你多运行这个程序几次，你会看到各种打印顺序出现

这段代码中有三个概念在发挥作用：futures、tasks和threads。`spawn`函数会接收一个future（可以由很多小的future构成）然后作为一个新的Tokio的任务来运行。任务是Tokio运行时排布和管理的概念（不只是单一的future）。Tokio（默认配置下）是一个多线程的运行时，当我们派生一个新任务时，这个任务可能会在不同的OS线程上运行，不一定是它被派生出来的那个线程（也可能是在同一个线程，或者开始在一个，之后被移动到另一个）

当一个future作为一个任务被派生，它会并发地和它被派生出来的任务和所有其它任务一起运行。如果被排布到不同线程，它也会和这些任务并行运行

当我们在Rust中写两条相邻语句时，它们会顺序执行（无论是否在异步代码中）。当我们写`await`时，并没有改变顺序语句的并发性。这两种情况下，两种情况下线程可能会交错执行这段代码，在异步函数情况下，任务可能会在等待点交错执行，但是两条语句还是会顺序执行

如果我们使用了`thread::spawn`或者`tokio::spawn`，我们引入了并发性和潜在的并行性。前一种情况是在线程之间，而后一种情况是在任务之间。之后我们可以看到并发执行future，但是不并行

## Joining tasks

如果我们想得到一个被派生任务的执行结果，然后派生任务会等它结束使用这个结果，这叫做等待任务（joining the tasks），“join threads”的同义词，它们对等待的API也类似

当任务被派生，`spawn`函数会返回一个`JoinHandle`。如果你只是想让任务执行它自己，那么你可以丢弃这个值。但是如果你想让派生任务等待这个被派生的任务完成并使用结果，你可以await这个`JoinHandle`

```rust
use tokio::{spawn, time::{sleep, Duration}};

async fn say_hello() {
    // Wait for a while before printing to make it a more interesting race.
    sleep(Duration::from_millis(100)).await;
    println!("hello");
}

async fn say_world() {
    sleep(Duration::from_millis(100)).await;
    println!("world");
}

#[tokio::main]
async fn main() {
    let handle1 = spawn(say_hello());
    let handle2 = spawn(say_world());

    let _ = handle1.await;
    let _ = handle2.await;

    println!("!");
}
```

这里我们保存了返回的`JoinHandle`，之后我们await了它们。因为我们在退出程序前等待它们完成，我们不再需要`sleep`函数

两个被派生的任务仍然是并发运行。如果你多运行程序几次，你会看到不同的打印顺序。然而被等待的JoinHandle是并发的一个限制：最后的"!"总是在最后被打印

如果我们在第一个`spawn`后立即await而不是保存下来之后await，那么我们会生成另一个任务来运行“hello”future。但是派生任务必须要等待它完成才能做其它事。也就是说不可能再出现并发了。你永远不想这样做（因为你直接写顺序代码就可以了）

## `JoinHandle`

实际上我们可以等待`JoinHandle`的原因是，它本身就是一个future。`spawn`不是一个异步函数，它是一个普通函数，返回一个future。它会在返回future（和异步future不同）之前做一些工作（排布任务），这就是为什么我们不需要等待`spawn`。await一个`JoinHandle`就是等待被派生的任务完成，然后返回它的结果。`JoinHandle`是一个泛型类型，它的类型参数是被派生的任务返回的类型

> `spawn`不是一个异步函数，所以调用它会得到它的返回值，它的返回值是一个future（`JoinHandle`），这个future在未来的某一刻会准备好传入的任务（异步future）的结果，await这个future就是等待它返回结果

await一个`JoinHandle`返回的类型是`Result`，之前例子中使用`let _ = ...`是为了避免存在未使用的`Result`引起的警告。如果被派生的任务成功完成，那么任务的结果就是`Ok`变体。如果任务崩溃或者被丢弃（取消的一种形式），那么结果就是`Err`，包含一个`JoinError`说明。如果你在项目中没有通过`abort`来取消，那么`unwrapping`这个结果是合理的方法，因为这会将被派生的任务的崩溃信息传给派生任务
