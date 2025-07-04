# 第十七章 - 异步编程基础：Async、Await、Futures和Streams

现代计算机对于同时执行超过一个任务提供了两种技术：并行和并发。当我们开始写并行或者并发程序时，很快就会碰到从**异步编程**（*asynchronous programming*）中遇到的问题：操作不会按照它们的开始顺序完成。本章建立在上一章使用线程执行并行和并发任务的内容，介绍了另一种异步编程的方式：Rust的Futures、Streams，以及支持它们`async`和`await`语法，还有在异步操作中管理和组织它们的工具

先考虑两个例子，一个是视频导出，它会消耗CPU和GPU的能力，如果你只有一个CPU核心并且你的操作系统在完成任务前不会暂停它，这就是一个**串行**（*synchronously*）任务，在任务运行期间你什么都干不了。另一个是视频下载，它不会消耗很多CPU资源，CPU必须要等数据通过网络到达本地后执行I/O操作，尽管在数据到达时就可以读取，也需要花费一些时间让所有数据可用。视频导出就是**重CPU任务**（*CPU-bound*, compute-bound），它受限于计算机的数据处理速度，由CPU和GPU决定，决定操作的处理速度。视频下载是一个**重IO任务**（*IO-bound*），因为它受限于计算机的**输入输出速度限制**（*input and output*），它的执行速度只有当网络足够快时才会更快

在这两个例子中，操作系统都会不可见地中断任务来提供并发性。这个并发性体现在以整个程序为单位。很多情况下，我们可以在更细粒度下考虑我们的程序，得到比操作系统的调度更好的结果，因为我们可以看到操作系统看不到的可以并发执行任务的机会。例如，如果我们构建了一个管理文件下载的工具，我们应该在下载任务开始后不锁住UI，用户可以同时下载多个文件。很多操作系统系统对于网络的交互API是阻塞式的，直到它们完全处理好之前都会阻塞程序。大部分函数都是这样工作的，阻塞式通常用来描述那些和文件、网络以及计算机其它资源交互的函数调用，因为这些情况都是程序可以受益于非阻塞式操作的地方

我们可以通过创建新线程执行任务来避免阻塞主线程，但是大量的线程最终也会出问题。有一种更好的方式是任务开始执行时不阻塞，就像我们写阻塞代码一样

```rust
let data = fetch_data_from(url).await;
println!("{data}");
```

这就是Rust提供的异步抽象

- 如何使用Rust的`async`和`await`语法
- 如何使用异步模型来解决多线程章节中我们遇到的一些挑战
- 多线程和异步如何提供互补的解决方案，很多情况下你可以结合两者使用

考虑一个团队在一个软件项目上可以使用不同的工作方式。要么给单一成员分配多个任务，要么每个成员一个任务，或者混合两种方式。当一个人处理多项不同的任务，且没有一个已经完成，这就是**并发**（*concurrency*）。你只是一个人，你不能同一时间做多个任务，但是你仍然可以做多个任务，每次做一个任务，然后在多个任务之间切换。当任务分给团队中的每个人，每个人独立完成自己的任务，这就是**并行**（*parallelism*），每个人在同一时间都可以做任务。在两种工作流中，你可能会在不同的任务之间协作。有些工作是有依赖性的，即一个人的要完成某个任务，必须依赖另一个人的某个任务完成，这个任务的执行过程实际上是**串行**（*serial*）的。并行和并发可以是互相交叉的，如果你有部分工作需要你的同事等待你完成后，它才能继续任务，那么你就必须先完成这个工作让你的同事不再等待，此时并行性被破坏了，对于你自己来说，并发性也被破坏了，因为你不能在多任务之间切换

在学习Rust的异步编程时，我们总是在处理并发操作，根据硬件、操作系统和我们使用的异步运行时，这里的并发性内部可能也会出现并行性

## Futures和异步语法

Rust中异步编程核心是`futures`和Rust的`async`以及`await`关键字。`future`表示目前可能还没有准备好的、在未来某个时间点可能会准备好的值。Rust提供了`Future`特质作为构建块，使那些不同的数据结构实现的异步操作有相同的公共接口。Rust中`futures`是那些实现了`Future`特质的类型。每个`future`包含的信息是关于已经当前任务进行的进度以及*ready*意味着什么

你可以用`async`关键字来描述语句块和函数，指定这些项的执行可以被中断和继续。在一个异步块或者异步函数中，你可以使用`await`关键字来*await a future*，在任意一个你等待`future`返回的地方，都是异步块或异步函数暂停和继续的地方，检查`future`确定它的值是否准备好的过程叫做*polling*

在写异步Rust的时候，多数时候我们使用`async`和`await`关键字，Rust会将它们编译为使用`Future`特质的代码，就像`for`循环会被编译为使用`Iterator`特质的代码。因为Rust提供了`Future`特质，你也可以为自己的数据结构实现它

### 第一个异步程序

这个项目需要使用提前准备好的`trpl`crate，它重新导出了所有我们需要的类型、特质和函数，主要来自`futures`和`tokio`这两个crate，`futures`是Rust的异步代码实验室，也是`Future`特质最初被设计的地方。Tokio是如今Rust最常见的异步运行时

#### 定义`page_title`函数

首先写一个函数，接受一个URL作为参数，然后发送一个请求，返回页面中题目元素的文本

```rust
use trpl::Html;

async fn page_title(url: &str) -> Option<String> {
    let response = trpl::get(url).await;
    let response_text = response.text().await;
    Html::parse(&response_text)
        .select_first("title")
        .map(|title_element| title_element.inner_html())
}
```

对于`get`函数，我们必须等待服务器返回这个URL的响应的第一部分，包括HTTP头、coockies等，这部分是可以和响应体分开传输的。如果响应体很大，则需要等一些时间。因为我们要接收到完整的响应内容，所以`text`方法也是异步的。我们必须显式等待这些`futures`，因为Rust中`futures`是懒的，在使用`await`关键字等待之前它们什么都不会做

#### 我们传入其它线程的闭包会立即运行

Rust的`await`关键字放在你等待的表达式后面。他是个**后缀**（*postfix*）关键字，这样在Rust中可以更好地实现链式调用，例如上面的`response_text`可以这样写`let response_text = trpl::get(url).await.text().await;`

当Rust看到标记`async`关键字的代码块，它会将其编译为一个唯一的异步数据类型，实现`Future`特质。当Rust看到一个函数标记为`async`，它会将其编译为非`async`函数，并将内部代码作为一个异步块。一个异步函数的返回值的类型是编译器为这个异步块创建的匿名数据类型的类型。写`async fn`等同于写一个函数，返回一个`future`，上面的函数编译后类似于下面的代码

```rust
use std::future::Future;
use trpl::Html;

fn page_title(url: &str) -> impl Future<Output = Option<String>> {
    async move {
        let text = trpl::get(url).await.text().await;
        Html::parse(&text)
            .select_first("title")
            .map(|title| title.inner_html())
    }
}
```

新函数体是一个`async move`块，因为这是它使用url参数的方式

#### 判断一个单页的标题

```rust
async fn main() {
    let args: Vec<String> = std::env::args().collect();
    let url = &args[1];
    match page_title(url).await {
        Some(title) => println!("The title for {url} was {title}"),
        None => println!("{url} had no title"),
    }
}
```

`await`只能在`async`块中使用，但是`main`函数不能被标记为`async`，因为运行异步代码需要一个**运行时**（*runtime*），它是一个crate，用来处理执行异步代码的细节。一个程序的`main`函数可以**初始化**（*initialize*）一个运行时，但是不能是运行时本身。每一个运行异步代码的Rust程序至少有一个地方来设置运行时并执行`futures`。很多语言有内置的支持异步的运行时，但是Rust不提供，因为有很多不同的异步运行时实现，每一个都根据它们的目标平台做了不同的取舍。本章的其余部分，我们会使用`trpl`crate的`run`函数，它接收一个`future`作为参数，执行它直至完成。调用`run`会设置一个执行传入的`future`的运行时，当`future`完成后，`run`会返回这个`future`的返回值

```rust
fn main() {
    let args: Vec<String> = std::env::args().collect();

    trpl::run(async {
        let url = &args[1];
        match page_title(url).await {
            Some(title) => println!("The title for {url} was {title}"),
            None => println!("{url} had no title"),
        }
    })
}
```

每个**等待点**（*await point*），即每个使用`await`关键字的地方，表示这是一个将控制权还给运行时的地方。Rust需要追踪在异步块中的状态，这样运行时可以决定终止一些工作，之后当`future`的值准备好时再回来继续执行。这是一个隐式的状态机，就像使用下面定义的这个枚举在每个等待点保存当前状态

```rust
enum PageTitleFuture<'a> {
    Initial { url: &'a str },
    GetAwaitPoint { url: &'a str },
    TextAwaitPoint { response: trpl::Response },
}
```

手写代码去转换各种状态很繁琐且容易出错，特别是当你需要添加一个功能或更多的状态时。Rust会对异步代码自动创建管理相关数据结构的状态机。所有权和借用规则仍然适用，编译器也会检查这些规则。最终有些东西需要执行这个状态机，这个东西就是运行时（**执行器**（*executor*）：运行时的一部分，负责执行异步代码）。如果`main`函数是一个异步函数，那么就需要有一个运行时来管理它的状态，在`main`自身的`future`返回时被处理，但是`main`是程序的开始点，所以`main`自身不能作为运行时。有些运行时提供了你可以直接写`async main`函数的宏，那些宏会生成和普通的`fn main`相同的代码

#### 竞争条件下爬取两个URL

```rust
use trpl::{Either, Html};

fn main() {
    let args: Vec<String> = std::env::args().collect();

    trpl::run(async {
        let title_fut_1 = page_title(&args[1]);
        let title_fut_2 = page_title(&args[2]);

        let (url, maybe_title) =
            match trpl::race(title_fut_1, title_fut_2).await {
                Either::Left(left) => left,
                Either::Right(right) => right,
            };

        println!("{url} returned first");
        match maybe_title {
            Some(title) => println!("Its page title is: '{title}'"),
            None => println!("Its title could not be parsed."),
        }
    })
}

async fn page_title(url: &str) -> (&str, Option<String>) {
    let text = trpl::get(url).await.text().await;
    let title = Html::parse(&text)
        .select_first("title")
        .map(|title| title.inner_html());
    (url, title)
}
```

在定义`title_fut_1`和`title_fut_2`时什么事情都没有发生，因为`future`是懒的，我们还没有等待它们。然后我们将`future`传入`trpl::race`，返回值表示哪个`future`先结束。`race`是建立在`select`上的，`select`函数可以做很多`race`函数做不到的事，但是它的结构更加复杂。`race`的返回值类型是`trpl::Either`，它和`Result`类似，有两种情况，使用`Left`和`Right`来表示“这个或者那个”

## 在并发编程中使用异步技术

大多数情况下，使用异步方式完成并发任务的API和那些使用线程的类似。即使API看起来差不多，它们通常有不同的行为，而且它们在性能上几乎总是不同

### 使用`spawn_task`创建一个新任务

`trpl`提供了`spawn_task`函数，类似`thread::spawn`函数，其中还使用了异步版本的`sleep`函数

```rust
use std::time::Duration;

fn main() {
    trpl::run(async {
        trpl::spawn_task(async {
            for i in 1..10 {
                println!("hi number {i} from the first task!");
                trpl::sleep(Duration::from_millis(500)).await;
            }
        });

        for i in 1..5 {
            println!("hi number {i} from the second task!");
            trpl::sleep(Duration::from_millis(500)).await;
        }
    });
}
```

当主异步块的`for`循环结束，整个程序就会结束，因为`spawn_task`开启的新任务在`main`函数退出后也会结束。如果你想让子任务结束后再终止程序，你需要使用`join`来等待第一个任务结束。在线程中我们使用`join`方法来阻塞当前线程直到它完成。在这个例子中，可以使用`await`做相同的事情，因为任务返回的`handle`是一个`future`，它的输出类型是一个`Output<Result<()>>`

```rust
let handle = trpl::spawn_task(async {
    for i in 1..10 {
        println!("hi number {i} from the first task!");
        trpl::sleep(Duration::from_millis(500)).await;
    }
});

for i in 1..5 {
    println!("hi number {i} from the second task!");
    trpl::sleep(Duration::from_millis(500)).await;
}

handle.await.unwrap();
```

这个例子和线程最大的区别是，我们没有开启一个新的操作系统线程。其实我们都不需要开启一个新任务。因为异步块会编译成匿名`future`数据结构，我们可以将循环放到异步块中，让运行时使用`trpl::join`运行它们直到双方都完成。当你给`trpl::join`函数传入两个`future`，会产生一个新的`future`，他的输出类型是包含传入的`future`都完成后的结果的元组

```rust
let fut1 = async {
    for i in 1..10 {
        println!("hi number {i} from the first task!");
        trpl::sleep(Duration::from_millis(500)).await;
    }
};

let fut2 = async {
    for i in 1..5 {
        println!("hi number {i} from the second task!");
        trpl::sleep(Duration::from_millis(500)).await;
    }
};

trpl::join(fut1, fut2).await;
```

现在，你会发现输出结果每次都差不多，跟使用线程的结果不一样。这是因为`trpl：：join`函数是**公平的**（*fair*）即它会公平检查每个`future`，在它们之间切换，不会出现另一个任务准备好了还让当前任务继续执行。线程是操作系统决定哪个线程执行多久。对于异步Rust来说，由运行时来决定处理哪个任务（实际上，如果异步运行时底层使用操作系统的线程来处理任务的并发性时情况会变得复杂，所以保证任务的公平性对于运行时要做很多工作--但是这仍然是可以实现的），运行时不需要对任何操作都保证公平性，它们通常会提供不同的API让你选择是否需要公平性

可以对于上面的代码做额外的实验

- 移除任一或两个循环外围的异步块移除 -- `join`方法报错
- 每个异步块定义后，立即对其执行`await`操作 -- 先执行完一个`future`，再执行另一个`future`，`join`方法报错，因为两个`future`都已经被执行完成，对应的实例已经被清理
- 只用异步块包裹第一个循环，然后在第二个循环体结束后等待其产生的`future` -- 先执行第二个循环体，然后再执行第一个循环体

### 使用消息传递在两个任务上计数

```rust
let (tx, mut rx) = trpl::channel();

let val = String::from("hi");
tx.send(val).unwrap();

let received = rx.recv().await.unwrap();
println!("Got: {received}");
```

`trpl::channel`是一个异步版本的多生产者-单消费者管道API，和基于线程版本的实现相比：它使用可变的接收端`rx`，`recv`方法会返回一个`future`，我们需要等待而不是直接接收一个值。同步版本的`Receiver::recv`方法会在接收到消息前阻塞，异步版本不会，它会将控制权返还给运行时，直到消息到来或者发送端被关闭。相反我们不会等待`send`，因为它也不会阻塞，它也不需要阻塞，因为我们使用的管道是**无界的**。因为在异步块中的异步代码都在`trpl::run`中执行，所有方法调用都可以避免阻塞。然而`run`函数之外的代码仍然会阻塞。`trpl::run`函数的重点是：它可以让你选择阻塞异步代码的地方，那里也是同步和异步代码转换的地方。在很多异步运行时中，`run`函数其实被命名为`block_on`就是这个原因

```rust
let (tx, mut rx) = trpl::channel();

let vals = vec![
    String::from("hi"),
    String::from("from"),
    String::from("the"),
    String::from("future"),
];

for val in vals {
    tx.send(val).unwrap();
    trpl::sleep(Duration::from_millis(500)).await;
}

while let Some(value) = rx.recv().await {
    println!("received '{value}'");
}
```

`rx.recv`调用创建了一个`future`，我们要等待它。运行时会在等待点暂停这个`future`，直到它准备好再执行。一旦有消息到来，`future`会处理为`Some(message)`，有多少消息就处理多少次。当管道关闭后，无论是否有消息到达，`future`都会被处理为`None`，表示管道已经没有更多的值

这个程序现在有两个问题，一是它是一次性打印所有消息而不是以500ms间隔打印，二是这个程序不会退出。对于第一个问题，因为在异步块中，`await`关键字出现在代码中的顺序就是它们在程序执行时的顺序。在代码片段中只有一个异步块，所以所有事情都是线性运行的。我们需要将`tx`和`rx`都操作放在不同的异步块中，然后可以使用`trpl::join`来同时运行它们，然后我们等待返回的新`future`

```rust
let tx_fut = async {
    let vals = vec![
        String::from("hi"),
        String::from("from"),
        String::from("the"),
        String::from("future"),
    ];

    for val in vals {
        tx.send(val).unwrap();
        trpl::sleep(Duration::from_millis(500)).await;
    }
};

let rx_fut = async {
    while let Some(value) = rx.recv().await {
        println!("received '{value}'");
    }
};

trpl::join(tx_fut, rx_fut).await;
```

但是这个程序依旧不会退出

- `trpl::join`返回的`future`，只有当传入的两个`future`都完成后才会返回
- `tx`发送完最后一条信息，并且休眠完就会退出
- `rx_fut`在`while let`结束前不会完成
- `while let`循环在等待`rx.recv`返回`None`之前不会结束
- 只有当管道另一端被关闭，`rx.recv`才会返回`None`
- 管道只有当我们调用`rx.close`或`tx.close`时才会关闭
- 我们没有调用`rx.close`，`tx`在传入`trpl::run`的异步块结束前也不会被清理
- 而异步块不会结束，因为它阻塞到了`trpl::join`位置-> 返回列表头部

我们发送消息只是借用了`tx`，如果我们可以将`tx`移动进异步块，当块结束时就它的值会被自动清理。`move`关键字对异步块的作用机制和闭包相同

```rust
let (tx, mut rx) = trpl::channel();

let tx_fut = async move {
    let vals = vec![
        String::from("hi"),
        String::from("from"),
        String::from("the"),
        String::from("future"),
    ];

    for val in vals {
        tx.send(val).unwrap();
        trpl::sleep(Duration::from_millis(500)).await;
    }
};

let rx_fut = async {
    while let Some(value) = rx.recv().await {
        println!("received '{value}'");
    }
};

trpl::join(tx_fut, rx_fut).await;
```

异步版本的管道也是一个多生产者管道，所以调用`tx`上的`clone`方法可以让我们在多个异步块中发送消息

```rust
let (tx, mut rx) = trpl::channel();

let tx1 = tx.clone();
let tx1_fut = async move {
    let vals = vec![
        String::from("hi"),
        String::from("from"),
        String::from("the"),
        String::from("future"),
    ];

    for val in vals {
        tx1.send(val).unwrap();
        trpl::sleep(Duration::from_millis(500)).await;
    }
};

let rx_fut = async {
    while let Some(value) = rx.recv().await {
        println!("received '{value}'");
    }
};

let tx_fut = async move {
    let vals = vec![
        String::from("more"),
        String::from("messages"),
        String::from("for"),
        String::from("you"),
    ];

    for val in vals {
        tx.send(val).unwrap();
        trpl::sleep(Duration::from_millis(1500)).await;
    }
};

trpl::join3(tx1_fut, tx_fut, rx_fut).await;
```

这里我们将新异步块放到了接收消息之后，但是它也能放前面。future被等待的顺序是核心，而不是它们创建的顺序

## 处理任意数量的Futures

我们有一种宏形式的`join`，可以传入任意数量的参数，它会等待传入的`future`完成，之前的代码可以改成`trpl::join!(tx1_fut, tx_fut, rx_fut);`。这种宏的形式只能在我们提前知道有多少个`future`时使用。然而在实际的Rust代码里，将`future`放到集合中，然后等待其中一些或者全部`future`完成才是更常见的模式。`trpl::join_all`函数接受任何实现了`Iterator`特质的类型，所以我们可以将`join!`的代码换成这样

```rust
let futures = vec![tx1_fut, rx_fut, tx_fut];

trpl::join_all(futures).await;
```

这段代码无法编译，虽然所有的异步块都没有返回任何内容，每个异步块的返回值都是`Future<Output = ()>`，记住`Future`是一个特质，编译器实际上为每个异步块构建了一个唯一的枚举类型，你不能将两个不同的手写结构体放到一个`Vec`中，相同的规则也应用于编译器生成的不同的枚举类型。要实现这样的效果我们需要使用特质对象。特质对象允许我们将每个`future`当作相同的类型。

> 之前讨论过另一种`Vec`包含多种类型的方式：使用一个枚举将所有可能的类型定义好。但在这里我们不能这样做，首先我们没有办法为枚举声明不同的类型名称，因为它们都是匿名的。其次我们使用`vector`和`join_all`就是为了在`future`的动态集合上做处理，我们只关心它们的输出值

```rust
let futures =
    vec![Box::new(tx1_fut), Box::new(rx_fut), Box::new(tx_fut)];

trpl::join_all(futures).await;
```

上面的代码依旧不能编译，我们会得到同样的错误，还有关于`Unpin`特质的新错误。首先修复`Box::new`调用需要显式声明类型的错误

```rust
let futures: Vec<Box<dyn Future<Output = ()>>> =
    vec![Box::new(tx1_fut), Box::new(rx_fut), Box::new(tx_fut)];
```

1. `Future<Output = ()>`表示这个`future`的输出是`()`类型
2. 使用`dyn`来标记特质对象
3. 将整个特质对象类型包裹到`Box`中
4. 最后我们显式声明`futures`是一个`Vec<Box<dyn Future<Output = ()>>>`

根据编译器之后的报错我们将代码修改如下才能正确运行

```rust
use std::pin::Pin;

// -- snip --

let futures: Vec<Pin<Box<dyn Future<Output = ()>>>> =
    vec![Box::pin(tx1_fut), Box::pin(rx_fut), Box::pin(tx_fut)];
```

首先，使用`Pin<Box<T>>`处理这些堆上保存的用`Box`包裹起来的`futures`时会增加一些开销，但我们只能通过这种方式让**类型对齐**。实际上我们不需要在堆上分配内存，这些`future`对于函数来说都是**局部的**（*local*）。`Pin`本身是一个**包装类型**，我们通过它让`Vec`中的类型统一--开始我们想用Box--并且不需要在堆上分配空间。我们可以通过调用`std::pin::pin`宏直接为每个`future`生成`Pin`。我们必须要知道`pinned`引用的类型，Rust仍然不知道需要将这些类型解释为动态特质对象，那才是我们要放到Vec中的内容

```rust
use std::pin::{Pin, pin};

// -- snip --

let tx1_fut = pin!(async move {
    // --snip--
});

let rx_fut = pin!(async {
    // --snip--
});

let tx_fut = pin!(async move {
    // --snip--
});

let futures: Vec<Pin<&mut dyn Future<Output = ()>>> =
    vec![tx1_fut, rx_fut, tx_fut];
```

目前我们忽略了`future`可能返回不同类型的返回值的情况。我们可以使用`trpl::join!`来等待这些`future`，因为它允许我们传入多个返回值类型不同的`future`，返回一个包含不同返回值类型的元组。这种情况下我们不能使用`trpl::join_all`，因为它需要传入的`future`都是相同的类型。这就是一个基础的取舍：只要类型相同，我们可以使用`join_all`来处理任意多个`future`，但我们只能利用`join`和`join!`来处理可数个类型不同的`future`

### 竞争Futures

当我们*join*所有的`futures`时，只有它们都完成才能程序才能继续。但有时我们只需要部分`future`完成，例如让一个`future`和另一个`future`竞争

```rust
let slow = async {
    println!("'slow' started.");
    trpl::sleep(Duration::from_millis(100)).await;
    println!("'slow' finished.");
};

let fast = async {
    println!("'fast' started.");
    trpl::sleep(Duration::from_millis(50)).await;
    println!("'fast' finished.");
};

trpl::race(slow, fast).await;
```

上面的代码中总是`fast`先完成。如果你交换传入`future`的参数顺序，那么消息的打印顺序就会改变，即使还是`fast`先完成，这是因为`race`函数的实现不是公平的，他总是按照传入的参数顺序来决定执行顺序。一些其它的实现可能是公平的，随机选择`future`来先运行。Rust会在等待中的`future`没有准备好时暂停任务切换到另一个。反过来也一样，Rust只有在等待点才会暂停异步块然后将控制权返回运行时，在等待点之间的代码都是同步执行的。这意味着如果你在一个没有等待点的异步块中写很多逻辑，这个`future`就会阻塞住其它的`future`。有时这种现象叫做一个`future`使其它`future`**饥饿**（*starving*）。有时这不是大问题。但是如果你在做一些高代价的启动操作或执行长时间运行的任务，或者是持续做一些工作，你就需要考虑何时何地将控制权返回给运行时。同理，如果你有需要长时间运行的阻塞操作，可以将它包装成一个异步函数，异步提供了一种程序之间不同部分相互关联的方式

### 将控制权返回给运行时

```rust
fn slow(name: &str, ms: u64) {
    thread::sleep(Duration::from_millis(ms));
    println!("'{name}' ran for {ms}ms");
}

let a = async {
    println!("'a' started.");
    slow("a", 30);
    slow("a", 10);
    slow("a", 20);
    trpl::sleep(Duration::from_millis(50)).await;
    println!("'a' finished.");
};

let b = async {
    println!("'b' started.");
    slow("b", 75);
    slow("b", 10);
    slow("b", 15);
    slow("b", 350);
    trpl::sleep(Duration::from_millis(50)).await;
    println!("'b' finished.");
};

trpl::race(a, b).await;
```

要让两个`future`可以切换运行，我们需要在任务中间加入等待点将控制权返回给运行时，例如使用`trpl::sleep`。虽然在每个slow调用后都加上了等待点，但是我们可以以任意顺序划分任务。但是我们并不想真正去休眠，我们想让执行速度更快，只需要将控制权返回给运行时。使用`yield_now`函数就可以这样做，使用这个函数可以更清晰地表达意图，也比`sleep`更快，因为`sleep`使用的定时器通常在粒度上有限制，即使我们传1纳秒的参数

```rust
let one_ns = Duration::from_nanos(1);
let start = Instant::now();
async {
    for_ in 1..1000 {
        trpl::sleep(one_ns).await;
    }
}
.await;
let time = Instant::now() - start;
println!(
    "'sleep' version finished after {} seconds.",
    time.as_secs_f32()
);

let start = Instant::now();
async {
    for _in 1..1000 {
        trpl::yield_now().await;
    }
}
.await;
let time = Instant::now() - start;
println!(
    "'yield' version finished after {} seconds.",
    time.as_secs_f32()
);
```

经过简单的测试，结果是调用`yield_now`更快。这意味着异步对于重计算的任务也是有用的，如何使用仍然取决于你的程序要执行的任务，因为它提供了一种在程序的不同部分之间构建一种关系的工具。这是一种**合作多任务**（*cooperative multitasking*）的模式，每个`future`都可以决定什么时候通过等待点返回控制权。每个`future`各自负责避免阻塞时间过长，在一些基于Rust的嵌入式操作系统中，这是唯一多任务模式。实际代码中，你不会在每一行都用等待点交还控制权。因为即使这种任务切换的开销不是很大，但仍然是有代价的。多数情况下，尝试切分重计算任务会让它的执行速度更慢，所以有时利用所有的性能让阻塞操作时间更快结束会更好。要经过测试来确定代码的性能瓶颈在哪。同时底层机制也需要牢记在心，尤其是当你看到很多任务后，你希望可以并发执行这些任务，但是它们却在串行执行

## 构建我们自己的异步抽象

我们可以将`future`组合到一起来创建新模式。例如构建一个`timeout`函数，结合已有的异步构造块。生成的结果可以作为创建其它异步抽象的异步构建块

```rust
let slow = async {
    trpl::sleep(Duration::from_millis(100)).await;
    "I finished!"
};

match timeout(slow, Duration::from_millis(10)).await {
    Ok(message) => println!("Succeeded with '{message}'"),
    Err(duration) => {
        println!("Failed after {} seconds", duration.as_secs())
    }
}
```

考虑一下`timeout`API要包含的内容

- 它本身是一个异步函数，我们可以等待它
- 它的第一个参数应该是实际运行的`future`。我们可以使用泛型来接收任意`future`
- 它的第二个参数是最长超时时间，我们使用`Duration`，因为可以直接传入`trpl::sleep`函数
- 它应该返回一个`Result`，如果`future`在超时时间内成功完成，`Result`返回`Ok`和`future`返回的值，如果执行超时，`Result`会是`Err`和超时等待的时间

```rust
async fn timeout<F: Future>(
    future_to_try: F,
    max_time: Duration,
) -> Result<F::Output, Duration> {
    // Here is where our implementation will go!
}
```

我们需要实现的逻辑：让传入的`future`和超时时间竞争。我们可以使用`trpl::sleep`制作一个定时器`future`，使用`trpl::race`来运行这个定时器和传入的`future`。同时由于`race`是不公平的，为了让`max_time`很小情况下`future_to_try`也可能完成，需要将这个`future`放到`trpl::race`的第一个参数

```rust
use trpl::Either;

// --snip--

fn main() {
    trpl::run(async {
        let slow = async {
            trpl::sleep(Duration::from_secs(5)).await;
            "Finally finished"
        };

        match timeout(slow, Duration::from_secs(2)).await {
            Ok(message) => println!("Succeeded with '{message}'"),
            Err(duration) => {
                println!("Failed after {} seconds", duration.as_secs())
            }
        }
    });
}

async fn timeout<F: Future>(
    future_to_try: F,
    max_time: Duration,
) -> Result<F::Output, Duration> {
    match trpl::race(future_to_try, trpl::sleep(max_time)).await {
        Either::Left(output) => Ok(output),
        Either::Right(_) => Err(max_time),
    }
```

## Streams：一组有序的Futures

异步版本的`recv`方法会在一段时间中生成一个序列。这是一种被称为`stream`的模式。迭代器和异步管道接收者有两个区别：首先是时间，迭代器是同步的，而管道接收者是异步的。其次是API，在处理`Iterator`时，我们调用同步的`next`方法，对于`trpl::Receiver`，我们调用的是异步`recv`方法。一个流就像异步版本的迭代器，尽管`trpl::Receiver`是在等待接收信息，更通用的流API范围更广：以迭代器的方式异步产生下一个项。在Rust中，我们可以基于迭代器创建一个流。然后通过调用它的`next`方法来等待输出

```rust
let values = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
let iter = values.iter().map(|n| n * 2);
let mut stream = trpl::stream_from_iter(iter);

while let Some(value) = stream.next().await {
    println!("The value was: {value}");
}
```

上面这段代码会提示没有`next`方法存在。原因是我们需要将特定特质引入域中。这个特质是`StreamExt`，`Ext`是Rust社区通过新特质来扩展已有特质的常见模式。目前只需要知道`Stream`特质定义了低级接口，高效组合了`Iterator`和`Future`特质。`StreamExt`提供了基于`Stream`的高级的API，`Stream`和`StreamExt`还不是Rust标准库的一部分，但是大多数异步生态中的crate都使用相同的定义

### 组合流

很多概念在流中可以很自然表示：在队列中的项逐渐可用，当总体数据集对于计算机内存而言数量很大时，每次仅从文件系统中拉取部分数据，或者网络上的数据随着时间逐渐到达。因为流是多个`futures`，我们可以使用它们和其它的流进行组合

```rust
use trpl::{ReceiverStream, Stream, StreamExt};

fn main() {
    trpl::run(async {
        let mut messages = get_messages();

        while let Some(message) = messages.next().await {
            println!("{message}");
        }
    });
}

fn get_messages() -> impl Stream<Item = String> {
    let (tx, rx) = trpl::channel();

    let messages = ["a", "b", "c", "d", "e", "f", "g", "h", "i", "j"];
    for message in messages {
        tx.send(format!("Message: '{message}'")).unwrap();
    }

    ReceiverStream::new(rx)
}
```

`get_messages`函数返回的是`impl Stream<Item = String>`类型，其中我们创建了一个异步管道。我们也使用了新类型`ReceiverStream`，将`rx`从`trpl::channel`转为有`next`方法的`Stream`。接下来添加一个特性：流中的每一个元素获取都有超时时间

```rust
use std::{pin::pin, time::Duration};
use trpl::{ReceiverStream, Stream, StreamExt};

fn main() {
    trpl::run(async {
        let mut messages =
            pin!(get_messages().timeout(Duration::from_millis(200)));

        while let Some(result) = messages.next().await {
            match result {
                Ok(message) => println!("{message}"),
                Err(reason) => eprintln!("Problem: {reason:?}"),
            }
        }
    })
}
```

我们使用`timeout`方法为获取流中的项添加超时，它定义于`StreamExt`特质中。然后我们更新`while let`循环，因为流现在返回的是`Result`，`Ok`表示按时收到的信息，`Err`表示消息到来前已经超时。我们`match`这个`Result`值，打印到来的消息和超时信息。我们在添加超时功能后`pin`了消息，因为超时helper产生的流需要被`pin`才能被调度（poll）

```rust
fn get_messages() -> impl Stream<Item = String> {
    let (tx, rx) = trpl::channel();

    trpl::spawn_task(async move {
        let messages = ["a", "b", "c", "d", "e", "f", "g", "h", "i", "j"];
        for (index, message) in messages.into_iter().enumerate() {
            let time_to_sleep = if index % 2 == 0 { 100 } else { 300 };
            trpl::sleep(Duration::from_millis(time_to_sleep)).await;

            tx.send(format!("Message: '{message}'")).unwrap();
        }
    });

    ReceiverStream::new(rx)
}
```

要在`get_messages`函数发送信息中间不阻塞地休眠，需要使用异步的`sleep`函数，然而我们不能让`get_messages`自身作为异步函数，因为那样我们的返回类型就是`Future<Output = Stream<Item = String>>`，调用者就需要等待`get_messages`本身来得到流。要记住：所有的`future`是线性执行的，在`future`之间的代码是并发执行的。等待`get_messages`会让它发送完所有的信息，包括发送每条信息之间的休眠时间。结果是超时时间没有起作用，访问流冲的项没有任何延迟。上面这种调用`spawn_task`的方式是允许的，因为我们设置好了运行时，如果我们没有设置这样的程序会崩溃。其它的实现可能有不同的取舍：它们可能会新建一个运行时来避免崩溃，但是这样做会有额外的开销，或者它们直接在没有运行时引用的前提下不允许直接创建新任务。使用运行时之前要了解它们实现中的取舍。根据结果看，超时不影响所有数据到达，因为我们的管道是**无界的**（*unbounded*），它可以保存相当于内存容量大小的消息，如果消息在超时前没有到来，我们的流处理器会报告它，但是当再次从流中获取数据时，这条消息可能还会接收到

### 合并流

首先我们创建一个流，如果我们让它直接运行，它每毫秒会创建一个项

```rust
fn get_intervals() -> impl Stream<Item = u32> {
    let (tx, rx) = trpl::channel();

    trpl::spawn_task(async move {
        let mut count = 0;
        loop {
            trpl::sleep(Duration::from_millis(1)).await;
            count += 1;
            tx.send(count).unwrap();
        }
    });

    ReceiverStream::new(rx)
}
```

这种无限循环的模式--只有当整个运行时退出时才会结束--在异步Rust中非常常见：很多程序需要一直运行，通过异步的方式，只要在循环的中间至少有一个等待点，它不会阻塞任何调用

```rust
let messages = get_messages().timeout(Duration::from_millis(200));
let intervals = get_intervals();
let merged = messages.merge(intervals);
```

将多个流合并为一个流，只要有流中的任意源有数据就会生成一个项，但是没有确定的顺序。这里`messages`和`intervals`都不需要被`pin`或者声明为可变，因为它们被合并进了`merged`流。目前代码无法通过编译，因为两个流的类型不同，`messages`的类型是`Timeout<impl Stream<Item = String>>`，`Timeout`是实现了`Stream`特质的类型，`intervals`流是`impl Stream<Item = u32>`的类型，要合并这两个流，我们需要将其中一个的类型转换成与另一个相匹配。我们要修改`intervals`流，因为`messages`已经是我们需要的基本形式，我们必须处理超时错误

```rust
let messages = get_messages().timeout(Duration::from_millis(200));
let intervals = get_intervals()
    .map(|count| format!("Interval: {count}"))
    .timeout(Duration::from_secs(10));
let merged = messages.merge(intervals);
let mut stream = pin!(merged);
```

首先使用`map`来将`intervals`转为字符串，然后再匹配`Timeout`类型。因为我们实际上在这个流上并不需要超时，所以我们只要创建一个远大于生成项的时间的超时流即可。将合并流`pin`住保证这些操作的安全性，最后我们要让`stream`声明为可变，`while let`循环的`next`调用才能遍历流。现在代码有两个问题，首先它不会停止，其次英文字母会被淹没在时刻信息中

```rust
let messages = get_messages().timeout(Duration::from_millis(200));
let intervals = get_intervals()
    .map(|count| format!("Interval: {count}"))
    .throttle(Duration::from_millis(100))
    .timeout(Duration::from_secs(10));
let merged = messages.merge(intervals).take(20);
let mut stream = pin!(merged);
```

首先使用`throttle`方法让`intervals`流的内容不会淹没`messages`流的项。**节流**（*throttling*）是限制函数被调用的速率的一种方式，在这个例子中，就是流以什么速率被**调度**（*poll*）。代码中是每100ms被调用一次，这和消息到达的速率差不多。为了限制我们从流中接收的项的数量，我们在`merged`流上调用了`take`方法，因为我们要限制最终的输出项的个数，而不是某个流的所有内容。`throttle`调用会创建一个新的包裹原始流的流，所以原始流只会以节流的速率被调度，而不是它自身的执行速率。也不会生成大量未经处理的时刻消息，因为我们根本没有创造过它们，这是Rust的`future`的工作机制内在的“懒特性”在起作用，让我们可以实现不错的性能。最后我们要处理错误，两个基于管道的流，`send`调用在另一端被关闭时会失败，是由于运行时执行`futures`构成的流的方式产生的问题。之前我们使用`unwrap`来忽略它，但是在那些更好的App中，应该主动处理错误，至少是不再尝试发送任何消息

```rust
fn get_messages() -> impl Stream<Item = String> {
    let (tx, rx) = trpl::channel();

    trpl::spawn_task(async move {
        let messages = ["a", "b", "c", "d", "e", "f", "g", "h", "i", "j"];

        for (index, message) in messages.into_iter().enumerate() {
            let time_to_sleep = if index % 2 == 0 { 100 } else { 300 };
            trpl::sleep(Duration::from_millis(time_to_sleep)).await;

            if let Err(send_error) = tx.send(format!("Message: '{message}'")) {
                eprintln!("Cannot send message '{message}': {send_error}");
                break;
            }
        }
    });

    ReceiverStream::new(rx)
}

fn get_intervals() -> impl Stream<Item = u32> {
    let (tx, rx) = trpl::channel();

    trpl::spawn_task(async move {
        let mut count = 0;
        loop {
            trpl::sleep(Duration::from_millis(1)).await;
            count += 1;

            if let Err(send_error) = tx.send(count) {
                eprintln!("Could not send interval {count}: {send_error}");
                break;
            };
        }
    });

    ReceiverStream::new(rx)
}
```

## 再探异步特质

### `Future`特质

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

`Future`的关联类型`Output`表示`future`会产生的值的类型。这和`Iterator`的`Item`关联类型类似。`Future`还定义了一个`poll`方法，接受一个特殊的对`self`参数的可变引用的`Pin`类型和一个`Context`类型的可变引用

```rust
enum Poll<T> {
    Ready(T),
    Pending,
}
```

`Poll`类型和`Option`相似，有两个变体，包含值的`Ready(T)`，还有不包含值的`Pending`。`Pending`变体表示`future`还有工作要做，调用者需要稍后检查状态，`Ready`表示`future`已经完成工作，`T`的值已经可用。对大多数`future`来说，调用者在`future`已经返回`Ready`值之后就不应该再次调用`poll`。很多`future`在`ready`之后如果再次调用`poll`就会崩溃。如果可以安全地再次调用`poll`的情况应该会在文档中说明。这个行为和`Iterator::next`类似

Rust会将使用await的代码其编译为下面调用`poll`的代码

```rust
match page_title(url).poll() {
    Ready(page_title) => match page_title {
        Some(title) => println!("The title for {url} was {title}"),
        None => println!("{url} had no title"),
    }
    Pending => {
        // But what goes here?
    }
}
```

如果`future`还在`Pending`状态，我们使用一些方式来重复检查，直到`future`最终准备好。如果Rust将代码编译成这样，那么每个await都会阻塞。Rust会保证这里的循环过程可以放弃控制权给其它任务，可以暂停当前`future`上的工作去执行其它`future`的工作，之后再来检查这个`future`，保证这个机制的就是异步运行时，这里的**排期和协调**（*scheduling and coordination*）就是它的主要工作。对于之前的`rx.recv`，`recv`调用返回一个`future`，等待这个`future`就是对它调用`poll`。运行时会暂停这个`future`，直到它以`Some(message)`或`None`准备好。运行时在`poll`返回`Poll::Pending`时知道`future`还没有准备好，当返回`Poll::Ready(Some(message))`或`Poll::Ready(None)`时知道该`future`已经准备好。`future`的基本机制是：运行时调用它负责的每个`future`，当`future`没有准备好时会将`future`重新置入休眠状态

### `Pin`和`Unpin`特质

```bash
error[E0277]: `{async block@src/main.rs:10:23: 10:33}` cannot be unpinned
  --> src/main.rs:48:33
   |
48 |         trpl::join_all(futures).await;
   |                                 ^^^^^ the trait `Unpin` is not implemented for `{async block@src/main.rs:10:23: 10:33}`
   |
   = note: consider using the `pin!` macro
           consider using `Box::pin` if you need to access the pinned value outside of the current scope
   = note: required for `Box<{async block@src/main.rs:10:23: 10:33}>` to implement `Future`
note: required by a bound in `futures_util::future::join_all::JoinAll`
  --> file:///home/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/futures-util-0.3.30/src/future/join_all.rs:29:8
   |
27 | pub struct JoinAll<F>
   |            ------- required by a bound in this struct
28 | where
29 |     F: Future,
   |        ^^^^^^ required by this bound in `JoinAll`
```

`trpl::join_all`返回一个叫做`JoinAll`的结构体，这个结构体包含一个`F`泛型类型，它需要实现`Future`特质，直接使用`await`等待一个`future`会隐式`pin`这个`future`，所以并不需要在等待任何`future`时都使用`pin!`。这里因为我们没有直接等待`future`，而是通过将一些`future`传入`join_all`函数构建了一个新的`future`。`join_all`函数签名的参数要求传入值的类型都实现`Future`特质，`Box<T>`只有当`T`实现了`Unpin`特质才会实现`Future`

在`Future`特质的`poll`方法定义中，`cx`参数和`Context`类型是运行时获取何时检查那些还处于懒状态的`future`的关键，它具体的工作机制只有在写自定义类型的`Future`特质实现时才需要考虑。这里要注意的是`self`参数，有类型注解的`self`和其它的函数参数的类型注解工作方式类似，核心区别是

- 它告诉Rust，`self`在方法被调用时的类型必须是什么
- 它不能是任何类型，它限制了类型可能是需要实现哪些特质、是某个类型的引用或者智能指针，或者包裹这个类型引用的`Pin`类型

这里只需要知道，如果我们想`poll`一个`future`，检查它是`Pending`还是`Ready(Output)`，我们需要一个`Pin`包裹的该类型的可变引用。`Pin`是一个包裹器，对一些类指针类型，例如`&`、`&mut`、`Box`和`Rc`（Pin通常和那些实现了`Deref`或者`DerefMut`特质的类型一起工作，但这实际上就等同于指针类型）`Pin`本身**不是一个指针**，没有类似于`Rc`和`Arc`的引用计数功能，它只是一个工具，让编译器用来强制应用指针用法的限制。`future`中的一系列等待点会被编译为状态机，编译器会保证这些状态机遵循Rust的安全规则，包括所有权和借用规则。要实现这一点，Rust会查看在一个等待点和下一个等待点或者这个异步块结束位置之间需要哪些数据，然后在编译好的状态机中创建对应的变体，每个变体获得它在这部分代码中使用的数据所需要的访问权，无论是通过所有权还是可变/不可变引用

目前为止都很理想，如果我们在一个异步块中对于所有权和引用的使用方式有哪些错误，借用检查会告诉我们。当我们想将与该代码块结合的`future`移动到别处，例如将其放入`Vec`然后传入`join_all`，事情就会变得复杂。当我们移动一个`future`，无论是将其放入一个数据结构然后在`join_all`中使用还是从函数返回，实际上移动了Rust为我们创建的状态机。和Rust的大多数其它类型不同，Rust为异步块创建的`future`可以在变体的成员中引用它们自己

默认情况下，任何包含自身引用的对象的移动操作都是不安全的，因为引用总是指向它们实际引用的值的内存地址。如果你移动了数据结构，那些内部引用还指向老位置。然而内存上的位置已经是无效的了。首先，当你对数据结构做改变时，数据结构不会更新，更重要的是，操作系统现在已经可以自由使用那块内存，你可能会读到完全不相关的数据。理论上，Rust编译器可以在移动时尝试更新对象的所有引用，但是这样就会增加大量性能开销，特别是如果存在大量的引用需要更新。如果我们可以保证数据结构**不会在内存中被移动**，那么我们就不需要更新任何引用。这就是Rust的借用检查器所需要的：在安全代码中，它不让你移动任何存在存活引用的项。`Pin`就是基于这个前提所构建的，它给予编译器所需要的保证，当我们将一个指针包裹进`Pin`来`pin`一个值，就表示它不会被移动，因此，如果你有一个`Pin<Box<SomeType>>`，实际上就是`pin`了`SomeType`的值，而不是Box指针。实际上，`Box`指针仍然可以自由移动。记住：我们关心的是保证**被引用的数据始终不被移动**，如果一个指针移动了，但是它指向的数据还在原位，那么不会发生问题。通过查看`std::pin`模块和`Pin`类型的文档可以了解如何通过`Pin`包裹`Box`来完成这件事的

因为大部分类型的移动都是安全的，无论它们是否通过`Pin`封装。我们只要注意那些包含自引用的类型。基本类型是安全的，因为它们没有任何自引用，平常用到的大多数类型也是这样。你可以移动`Vec`。如果你有一个`Pin<Vec<String>>`，即使`Vec<String>`在没有其它引用指向它时可以安全地移动，你也只能通过安全但是有限的`Pin`提供的API做一些事情，我们需要一种方式来告诉编译器在这种情况下移动它们是完全没问题的，这就是`Unpin`发挥的作用。`Unpin`是一个标记特质，类似`Send`和`Sync`，它本身没有定义方法。标记特质的存在只是为了告诉编译器在一个特定的上下文中使用实现了此特质的类型是安全的。`Unpin`告诉编译器这个类型不需要遵守任何关于它的值是否可以安全移动的保证。和`Send`和`Sync`一样，编译器自动为所有可以证明可以安全移动的类型实现了`Unpin`。有一个特例，就是`impl !Unpin for SomeType`，这里的`SomeType`是需要那些额外保证才能安全移动的类型名称，尤其是指向该类型的指针被包裹在`Pin`中使用时

`Pin`和`Unpin`的关系中有两点需要注意，首先`Unpin`是**常态**（*"normal" case*），`!Unpin`是特例，其次一个类型实现`Unpin`或`!Unpin`，只有当你使用一个类型的`pinned`指针时才重要，例如`Pin<&mut SomeType>`。为了让这个概念具体一些，考虑`String`类型，它包含长度信息和Unicode字符，我们可以将`String`放到`Pin`中，然而`String`自动实现了`Unpin`，所以我们可以做那些如果`String`实现了`!Unpin`就不合法的事情，例如将相同内存上字符串替换成其它的内容，这没有违反`Pin`的协议，因为`String`没有自引用，但这会导致它的移动是不安全的。这就是为什么默认实现`Unpin`而不是`!Unpin`

现在可以理解`join_all`调用返回的错误了，我们开始将异步块构建的`future`直接放到`Vec<Box<dyn Future<Output = ()>>>`中，但是这些`future`是包含自引用的类型，所以它们没有实现`Unpin`，它们需要被`pinned`，所以我们需要传入`Pin`类型，即`future`中的底层数据是不会被移动的

`Pin`和`Unpin`在构建低级库或者在构建自定义的异步运行时非常重要，但是普通的Rust代码基本不需要。`Pin`和`Unpin`的组合使得在Rust中安全实现复杂类型成为可能，这是一项具有挑战性的工作，因为它们包含自引用。需要`Pin`的类型现在主要出现在异步Rust代码中，但是以后在其它情况下也可能看到它。在`std::pin`的文档中有关于`Pin`和`Unpin`如何工作的规则，还有使用它们需要遵循的规则

### `Stream`特质

流类似于异步迭代器。与`Iterator`、`Future`不同，`Stream`特质在标准库中没有定义，但是`futures`crate中包含整个异步生态使用的非常统一的定义。在`Iterator`中，我们如此考虑序列，它的`next`方法返回`Option<Self::Item>`类型的值，从`Future`中，我们需要随着时间读取，它的`Poll<Self::Output>`提供了`poll`方法，为了表示随时间产生项的序列，我们定义了结合两者的`Stream`特质

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

trait Stream {
    type Item;

    fn poll_next(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;
}
```

`Item`是流产生的项的数据类型。和`Iterator`一样，可能会有0个或多个项，但和`Future`不同，它总是有一个`Output`，即使值的类型是`()`。`Stream`也定义了方法来获取这些项。我们叫它`poll_next`，它使用和`Future::poll`相同的方式`poll`，和`Iterator::next`相同的方式生成项。它的返回类型组合了`Poll`和`Option`。外部类型是`Poll`，因为必须用它来检查`future`是否准备好。内部类型是`Option`，因为它需要一个信号来标识是否还有更多的项生成，就像迭代器那样。这些定义很可能会成为Rust标准库的一部分，同时它已经是大多数运行时工具箱的一部分，所以你可以放心依赖它们。我们可以直接使用`poll_next`API写我们自己的`Stream`状态机，就像我们直接调用它们的`poll`方法来处理`future`。但是使用`await`更好，`StreamExt`特质提供的`next`方法也是这样

```rust
trait StreamExt: Stream {
    async fn next(&mut self) -> Option<Self::Item>
    where
        Self: Unpin;

    // other methods...
}
```

我们之前使用的`next`函数签名可能和上面代码中定义的不太一样，因为它使用的Rust版本还不支持在特质中定义异步方法，它的定义形式可能是`fn next(&mut self) -> Next<'*, Self> where Self: Unpin;`，`Next`类型是一个实现了`Future`的结构体，允许我们给`&mut self`定义生命周期为`Next<'*, Self>`，让`await`可以在这个方法上使用。`StreamExt`自动在所有实现了`Stream`特质的类型上实现这个特质，这些单独定义的特质允许社区在不影响基础特质的情况下扩展方便的API

## 合到一起：Futures，Tasks和Threads

线程是一种实现并发的方式，使用异步的`future`和`stream`是另一种方式。如何选择需要根据实际情况进行判断。在很多情况下，结果不是二选一而是两者都用，因为很多操作系统在几十年前就提供了基于线程的并发模型，所以很多编程语言支持它们，即使这些模型也有各自的取舍。在很多操作系统中，它们会为每个线程分配一些内存，启动和停止线程都会产生开销，而且线程只有在你的操作系统和硬件都支持的时候才能用。一些嵌入式系统可能连OS都没有，所以它们也不支持线程。异步模型不一样，它和基于线程的并发模型是完全互补的关系。在异步模型中，并发操作不一定需要自己的线程。它们可以运行在**任务**上，就像`stream`章节中我们使用`trpl::spawn_task`从同步函数中启动任务。一个任务就像一个线程，但不是通过操作系统管理，而是通过库级别的代码管理，即异步运行时

下面是用线程实现的一个生成流的函数

```rust
fn get_intervals() -> impl Stream<Item = u32> {
    let (tx, rx) = trpl::channel();

    // This is *not* `trpl::spawn` but `std::thread::spawn`!
    thread::spawn(move || {
        let mut count = 0;
        loop {
            // Likewise, this is *not* `trpl::sleep` but `std::thread::sleep`!
            thread::sleep(Duration::from_millis(1));
            count += 1;

            if let Err(send_error) = tx.send(count) {
                eprintln!("Could not send interval {count}: {send_error}");
                break;
            };
        }
    });

    ReceiverStream::new(rx)
}
```

这段代码创建的流和`stream`章节中使用`trpl::spawn_task`和`trpl::sleep`的方式创建的流完全一样，虽然一个是运行时中的异步任务，另一个是操作系统的线程，但最终产生的流是没有区别的。实际上，两种模型的区别在于，我们可以在任何一台现代个人计算机上启动数百万个异步任务，但是对于线程来说，这样做会耗尽内存

它们的API非常相似是有原因的。线程就像一些同步操作的边界，并发情况可能是在线程之间的。任务就像一些异步操作的边界，并发情况可能在任务之间或者任务内部出现，因为一个任务可以在future之间来回切换。`future`是Rust中最细粒度的并发，每个`future`可以表示成包含其它`future`的树结构。运行时的执行器来管理任务，然后任务管理`future`。从这个角度看，任务就像是轻量级的、由运行时管理的线程，并且具有运行时而不是操作系统管理的优势。这不代表异步模型总是比线程更好的选择。线程实现的并发有时是更简单的并发方案，这是优点也是缺陷。线程被称为*fire and forget*，它们没有等同于`future`的概念，所以它们只要不被操作系统中断，就会持续运行直到结束。也就是说它们没有`future`支持的**任务内并发**（*intratask concurrency*）。Rust中的线程没有取消机制，然而对于`future`，当它结束时状态可以被正确地清理。这种特点也让线程更难组合。比如很难使用线程来构建`timeout`或者`throttle`等helper。`future`是更丰富的数据结构，也意味着它们可以更自然地结合到一起

任务让我们对`future`有一些额外的控制力，例如选择在哪和如何组织它们。线程和任务可以一起工作，因为任务（在一些运行时中）可以在线程中移动。实际上，我们使用的运行时（包括`spawn_blocking`和`spawn_task`函数）默认是多线程的。很多运行时使用叫做**偷工时**（*work stealing*）的方式来透明地在线程中间移动任务，基于当前线程的使用情况，来改善系统的性能。这种方法需要线程和任务，当然还有`future`

在考虑该使用哪种模型时，首先要考虑以下问题

- 如果工作是更并行化的（parallelizable），例如处理一批数据，每一部分可以并行处理，那么线程是更好的选择
- 如果工作时更并发化的（concurrent），例如从不同的源处理可能以不同间隔或者不同速率传来的消息，异步模型是更好的选择

两者一起使用的例子如下

```rust
use std::{thread, time::Duration};

fn main() {
    let (tx, mut rx) = trpl::channel();

    thread::spawn(move || {
        for i in 1..11 {
            tx.send(i).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    trpl::run(async {
        while let Some(message) = rx.recv().await {
            println!("{message}");
        }
    });
}
```

回到本章最开始的问题，视频编码任务应该使用单独的线程，因为视频编码是重计算的，但是通知UI哪些操作已经做完可以使用异步模型中的管道
