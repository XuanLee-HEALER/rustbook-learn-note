# 第二十一章 - 构建一个多线程Web服务器

如何构建一个Web服务器

- 学习一些TCP和HTTP的知识
- 在一个socket上监听TCP连接
- 解析一些HTTP请求
- 创建一个正确的HTTP响应
- 使用线程池提高服务器的吞吐量

## 构建一个单线程Web服务器

web服务器中主要的两个协议是**HTTP**（*Hypertext Transfer Protocol*）和**TCP**（*Transmission Control Protocol*），两个协议都是**请求-响应**（*request-response*）协议，模式是一个**客户端**（*client*）初始化一个请求，一个**服务器**（*server*）监听请求然后响应客户端。请求和响应的内容由协议定义。TCP是一个低级协议，描述了信息如何在服务器之间转移，但是并不定义传递的内容。HTTP协议基于TCP构建，定义了请求和响应的内容。技术上HTTP也可以在其他协议上应用，但是多数情况下，HTTP都使用TCP协议发送数据

### 监听TCP连接

标准库提供了`std::net`模块让我们监听TCP连接

```bash
$ cargo new hello
     Created binary (application) `hello` project
$ cd hello
```

在`src/main.rs`中，代码会监听本地地址`127.0.0.1:7878`

```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        println!("Connection established!");
    }
}
```

使用`TcpListener`在`127.0.0.1:7878`地址上监听TCP连接。在地址中，冒号之前是IP地址，`7878`是端口号。选择这个端口有两个原因：HTTP服务器通常不监听这个端口，所以我们的服务器不太可能和其他web服务器冲突，`7878`是rust在电话键盘上输入的结果。`bind`函数就像`new`函数一样，返回一个新的`TcpListener`实例，之所以叫做`bind`是因为在网络中，监听一个端口号也叫“绑定到端口”。`bind`返回`Result<T, E>`，即绑定操作可能会失败。例如绑定`80`端口需要管理员权限（非管理员只能监听`1023`以上的端口）。如果我们运行了程序的两个实例，即两个程序监听同一个端口，其中一个程序会失败。因为这只是一个练习程序，所以现在并不处理这些可能的错误，直接使用`unwrap`来停止程序

`TcpListener`上的`incoming`方法返回一个迭代器，产生的项是流（`TcpStream`类型）。一个`stream`表示一个在客户端和服务端之间打开的连接。一个`connection`是客户端连接服务端进行完整的请求和响应过程的名称，客户端连接服务端，服务端生成响应，然后服务端关闭连接。我们会从`TcpStream`读取数据得知客户端发送了什么内容，然后将我们的响应写入流中发回客户端。这个`for`循环会按顺序处理每个连接。如果获取流出现任何错误，我们就使用`unwrap`来终止程序，如果没有任何错误，程序打印消息。当客户端连接服务端时，`incoming`方法可能出现错误的原因是实际上我们遍历得到的不是连接，而是**连接尝试**（*connection attempts*）。连接可能有很多原因未成功创建，大部分是操作系统层面的问题。例如很多操作系统会限制同时打开的连接数，新创建的连接可能会超过这个限制，在其它打开的连接关闭之前就会报错

运行`cargo run`后，在浏览器输入`127.0.0.1:7878`，浏览器会显示错误，因为服务端目前没有返回任何数据，但是终端会打印消息，有时一个请求可能有多条消息打印，这是因为有些浏览器会同时尝试请求页面和其它资源，例如`favicon.ico`，也可能是因为浏览器因为服务器没有响应任何数据尝试连接服务器多次。`stream`离开域后，它的`drop`方法会关闭连接，浏览器有时会对已关闭连接进行重试。浏览器有时也会向服务器打开多个连接但是不发请求，这样之后如果它们发送请求的速度会更快。这种情况下，我们在服务器可以看到每个连接，无论连接上是否有请求。很多基于Chrome的浏览器都会这样做，你可以使用隐私浏览模式或者使用不同的浏览器来禁用这项优化。记住当你运行完某个版本的程序后需要通过ctrl-c关闭程序，再重启程序才能保证运行的是最新的代码

### 读取请求

为了区分获取连接和处理连接的逻辑，我们创建新函数来处理连接。在`handle_connection`函数中，我们会读取TCP流中的数据，然后打印它，这样我们就能看到浏览器发送的数据

```rust
use std::{
    io::{BufReader, prelude::*},
    net::{TcpListener, TcpStream},
};

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        handle_connection(stream);
    }
}

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    println!("Request: {http_request:#?}");
}
```

我们将`std::io::prelude`和`std::io::BufReader`引入域中来使用读写流有关的类型和特质。在`for`循环中调用`handle_connection`函数，传入`stream`参数。在函数中，我们创建了新的`BufReader`实例，包裹一个`stream`的引用。`BufReader`给我们提供带缓冲功能的`std::io::Read`特性定义的方法。`http_request`变量收集浏览器发送的请求，我们将这些字符串行收集到`Vec<*>`中。`BufReader`实现了`std::io::BufRead`特质，提供了`lines`方法。`lines`方法返回`Result<String, std::io::Error>`类型的迭代器，只要碰到新行字节就划分数据。要获取其中的`String`，我们使用`map`，然后`unwrap`每个`Result`。如果数据不是有效的UTF-8字符串或者从流中读数据出问题都可能返回`Err`。对于生产环境的程序，我们需要更优雅地处理这些错误，这里为了简单直接退出程序

浏览器通过发送两个换行符组成的行来结束请求。所以为了从流中得到一个完整的请求，我们按行读取，直到遇到空字符串，一旦我们将行收集到vector中，就使用debug格式来打印它们。我们可以从输出中看到请求的第一行中`GET`后的路径。如果重复的连接都是请求`/`路径，那么我们就知道浏览器在重复尝试获取`/`，因为它没有从服务器得到响应

## 再探HTTP请求

HTTP是基于文本的协议，请求格式如下

```text
Method Request-URI HTTP-Version CRLF
headers CRLF
message-body
```

第一行是**请求行**（*request line*），表示客户端请求的是什么。第一部分是使用的**方法**（*method*），例如`GET`或`POST`，含义是客户端如何制作这个请求，这里使用`GET`，表示请求读取信息。下一部分是`/`，表示客户端请求的`通用资源标识符`（*uniform resource identifier, URI*）一个URI基本上等同于**通用资源地址符**（*uniform resource locator, URL*）两者有一些区别，HTTP规范中使用的术语是URI。最后一部分是客户端使用的HTTP版本，然后以`CRLF`序列结尾（`CRLF`表示*carriage return*和*line feed*）这是打字机时代的术语，`CRLF`序列可以被写成`\r\n`，`\r`是转到开头，`\n`是新行。这个序列区分了请求行和剩余的请求数据。剩下的行中`Host:`之后的内容是请求头，`GET`请求没有请求体

### 写一个响应

响应的形式如下

```text
HTTP-Version Status-Code Reason-Phrase CRLF
headers CRLF
message-body
```

第一行是**状态行**（*status line*）包含HTTP版本，一个表示请求结果的总结信息的数字状态码，之后是原因部分，提供状态码的文本解释。在CRLF之后是响应头，然后是另一个CRLF，然后是响应体

```text
HTTP/1.1 200 OK\r\n\r\n
```

`200`状态码是标准的成功响应。文本`OK`是简单的HTTP响应。将这个内容写到流中

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    let response = "HTTP/1.1 200 OK\r\n\r\n";

    stream.write_all(response.as_bytes()).unwrap();
}
```

`response`变量保存成功消息的数据，然后调用`as_bytes`转换成字节数组，`write_all`接收`&[u8]`参数，然后将这些字节发送给连接。因为`write_all`操作可能失败，需要使用`unwrap`处理

### 返回真正的HTML

在项目目录的根目录创建`hello.html`，不是在`src`目录中，内容可以是任何合法的HTML

```rust
use std::{
    fs,
    io::{BufReader, prelude::*},
    net::{TcpListener, TcpStream},
};
// --snip--

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("hello.html").unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```

我们将`fs`模块加入到`use`语句中。读取文件内容到字符串的代码之前使用过。然后使用`format!`将文件内容作为成功响应的响应体。要保证这是一个有效的HTTP响应，我们添加了`Content-Length`头，设置我们的响应体的大小。现在我们忽略了`http_request`直接发回HTML文件，意味着请求任何路径都会得到相同的HTML响应，我们需要根据请求来定制响应，只在请求`/`路径时才发送HTML文件

### 验证请求并选择性响应

现在我们要在返回HTML文件前检查浏览器请求的路径是否为`/`，如果请求其它路径就返回错误

```rust
// --snip--

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    if request_line == "GET / HTTP/1.1" {
        let status_line = "HTTP/1.1 200 OK";
        let contents = fs::read_to_string("hello.html").unwrap();
        let length = contents.len();

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
    } else {
        // some other request
    }
}
```

我们只需要检查HTTP请求的第一行，第一个`unwrap`获取`Option`中的值，如果迭代器没有项就停止程序，第二个`unwrap`处理`Result`。然后检查`request_line`中的`GET`请求访问的路径

```rust
    // --snip--
    } else {
        let status_line = "HTTP/1.1 404 NOT FOUND";
        let contents = fs::read_to_string("404.html").unwrap();
        let length = contents.len();

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
    }
```

我们的响应是一个`404`状态码和`NOT FOUND`原因的状态行。响应体是名称为`404.html`的HTML文件，你需要在`hello.html`的同级目录创建一个`404.html`文件。现在访问`127.0.0.1:7878`会返回`hello.html`，访问其它路径会返回404.html的内容

## 重构代码

目前`if`和`else`表达式中有很多重复内容，它们的过程都是读取文件然后将文件内容写到流中。唯一的区别是状态行和HTML文件名不同。我们可以将不同的部分放到if和else中来赋值，从而让代码更精简

```rust
// --snip--

fn handle_connection(mut stream: TcpStream) {
    // --snip--

    let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```

现在`if`和`else`表达式以元组形式返回状态行和正确的文件名，之后我们直接使用这两个变量。重复代码现在已经移到`if`和`else`之外，这样更容易看出两种情况的区别，如果我们想改变读取文件和写入响应的工作，只需要改后面的部分

## 将单线程服务器转换为多线程服务器

### 在当前服务器的实现下模拟一个慢请求

在`/sleep`请求路径模拟慢响应，在响应前等待5秒

```rust
use std::{
    fs,
    io::{BufReader, prelude::*},
    net::{TcpListener, TcpStream},
    thread,
    time::Duration,
};
// --snip--

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        handle_connection(stream);
    }
}

fn handle_connection(mut stream: TcpStream) {
    // --snip--

    let buf_reader = BufReader::new(&stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = match &request_line[..] {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
        "GET /sleep HTTP/1.1" => {
            thread::sleep(Duration::from_secs(5));
            ("HTTP/1.1 200 OK", "hello.html")
        }
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };

    // --snip--

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```

我们需要使用`request_line`切片内容匹配字符串字面值的模式，`match`不会像相等方法那样自动引用和解引用。对于`/sleep`，当请求到达时，服务器会等待5秒，然后渲染正确的HTML页面。使用`cargo run`运行服务器，开启两个浏览器窗口，一个访问`http://127.0.0.1:7878/`，另一个访问`http://127.0.0.1:7878/sleep`。如果先访问后者，那么前者会等待`sleep`结束才能看到页面。有很多种技术可以避免请求被阻塞到慢请求之后，包括上一章的异步，我们这里使用线程池

## 使用线程池来提高吞吐量

一个**线程池**（*thread pool*）是一组提前开启的线程，等待并准备好处理到来的任务。当程序接受到一个新任务，它会从池中分配一个线程给任务，然后线程处理任务。在第一个线程在处理任务的时候，剩下的线程仍然在等待其它任务。当线程处理完它的任务后，空闲线程会进入池中，继续等待新任务到来。线程池可以让你并发处理连接，增加服务器的吞吐量。我们会限制线程池中线程的数量，来防止DoS攻击，如果我们对每个请求都创建一个新线程，那么如果有人构造了1000万个请求，就会耗尽所有服务器资源，终止服务器运行。我们的池中会有固定数量的线程。到来的请求会发送给池中的线程来处理。线程池会维护一个接收到来请求的队列。池中的每个线程会从队列中取出一个请求处理，执行完后再处理队列中的其它请求。这个设计中，我们最多能并发处理N个请求，N是线程数量。如果每个线程都响应一个慢请求，那么后续超出N个的请求仍然会阻塞到队列，但是在到达瓶颈点之前我们仍然增加了处理慢请求的数量

线程池技术是改善web服务器吞吐量的多种方式的一种。此外还有*fork/join*模型，单线程异步I/O模型和多线程异步I/O模型。当你在设计代码时，可以先写一个客户端接口来帮助引导设计。使用API，以你想调用它们的方式进行组织，然后再在结构中实现功能，而不是先实现功能，再设计公共API。和TDD类似，我们使用**编译器驱动**开发。我们先写调用我们想写的函数的代码，然后我们通过编译器的错误信息来确定我们要如何修改代码来让它工作

### 对每个请求开启一个线程

首先我们要尝试为所有连接创建一个新线程

```rust
use std::{
    fs,
    io::{BufReader, prelude::*},
    net::{TcpListener, TcpStream},
    thread,
    time::Duration,
};

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        thread::spawn(|| {
            handle_connection(stream);
        });
    }
}

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = match &request_line[..] {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
        "GET /sleep HTTP/1.1" => {
            thread::sleep(Duration::from_secs(5));
            ("HTTP/1.1 200 OK", "hello.html")
        }
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```

修改完代码后，你会发现对`/`路径对请求不会再等待`/sleep`请求完成。但像之前提到过的，这样做最终会消耗完我们系统的资源

### 创建有限个线程

我们想让线程池的工作方式和线程类似，这样的话从线程切换到线程池，代码不需要有很大的修改

```rust
use std::{
    fs,
    io::{BufReader, prelude::*},
    net::{TcpListener, TcpStream},
    thread,
    time::Duration,
};

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }
}

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = match &request_line[..] {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
        "GET /sleep HTTP/1.1" => {
            thread::sleep(Duration::from_secs(5));
            ("HTTP/1.1 200 OK", "hello.html")
        }
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```

我们使用`ThreadPool::new`来创建一个新线程池，参数是线程数量。在`for`循环中，`pool.execute`和`thread::spawn`的接口类似，接收一个处理请求的闭包

### 使用编译器驱动开发构建`ThreadPool`

使用`cargo check`查看编译器报错。错误信息表示我们需要有`ThreadPool`类型或模块。因为我们的`ThreadPool`实现和我们的web服务器是相互独立的。所以要从二进制crate转到库crate来实现`ThreadPool`。在库crate中实现线程池可以单独使用线程池做任何任务，而不只限于服务web请求。在`src/lib.rs`中声明结构体

```rust
pub struct ThreadPool;
```

在`src/main.rs`中引入结构体后编译器还是会报错。提示我们需要创建`new`关联函数

```rust
pub struct ThreadPool;

impl ThreadPool {
    pub fn new(size: usize) -> ThreadPool {
        ThreadPool
    }
}
```

我们使用`usize`作为`size`参数的类型，因为它不可能是负数。错误提示`ThreadPool`中没有`execute`方法。我们的`execute`函数应该接收一个闭包，并且使用池中的空闲线程来处理这个闭包。因为我们想和`thread::spawn`的接口保持一致，所以可以先查看`thread::spawn`的函数签名

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
    where
        F: FnOnce() -> T,
        F: Send + 'static,
        T: Send + 'static,
```

`F`是我们关心的参数，`T`和我们不关心的返回值有关。`spawn`使用`FnOnce`作为`F`的特质限制。这也是我们需要的，因为最终我们会把`execute`得到的参数传入`spawn`，并且线程只会运行一次处理请求的闭包，这符合`FnOnce`中的`Once`。`F`还有特质限制`Send`和生命周期绑定`'static`，我们同样需要这两部分：`Send`是将闭包从一个线程传入另一个线程，`'static`是因为我们不知道线程会运行多久

```rust
pub struct ThreadPool;

impl ThreadPool {
    // --snip--
    pub fn new(size: usize) -> ThreadPool {
        ThreadPool
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
    }
}
```

这里在`FnOnce`后使用了`()`，因为`FnOnce`表示一个不接收参数、返回单元类型为`()`的闭包。和函数定义类似，返回值可以在签名中忽略，但是即使没有参数，也需要使用括号。目前编译可以通过，但是运行程序之后，发送请求会报错，因为我们的库没有调用传入`execute`的闭包。*if the code compiles, it works*这句话不完全对，我们的项目通过了编译，但是它实际上什么都没做。如果我们在构建一个真实的、完整的项目，现在就是写单元测试的好时机，来检查代码是否可以通过编译并且具有符合我们预期的行为

### 在`new`中验证线程的数量

虽然负数个数的线程数量没有意义，但是`0`个线程的线程池同样没有意义，但是`0`是合法的`usize`值。我们保证`size`大于`0`

```rust
pub struct ThreadPool;

impl ThreadPool {
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        ThreadPool
    }

    // --snip--
    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
    }
}
```

我们添加了一些`ThreadPool`的文档注释。添加一个章节来声明我们函数会崩溃的情况是好的文档实践。运行`cargo doc --open`然后点击`ThreadPool`结构来查看为`new`函数生成的文档。除了增加`assert!`宏，我们也可以将`new`改为`build`，返回一个`Result`，就像之前在I/O项目中那样。但是在当前项目中我们认为创建包含`0`个线程的线程池是不可恢复的错误

```rust
pub fn build(size: usize) -> Result<ThreadPool, PoolCreationError> {
```

### 创建空间来存储线程

在我们返回`ThreadPool`实例之前，我们可以创建好线程并保存进结构体中。`spawn`函数的返回类型是`JoinHandler<T>`，其中`T`是闭包返回的类型。我们传入线程池的闭包不会返回任何东西，所以`T`是单元类型

```rust
use std::thread;

pub struct ThreadPool {
    threads: Vec<thread::JoinHandle<()>>,
}

impl ThreadPool {
    // --snip--
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let mut threads = Vec::with_capacity(size);

        for _ in 0..size {
            // create some threads and store them in the vector
        }

        ThreadPool { threads }
    }
    // --snip--

    pub fn execute<F>(&self, f: F)
        where
        F: FnOnce() + Send + 'static,
        {
        }
}
```

`with_capacity`函数的功能和`Vec::new`相同，区别是前者会为vector预分配空间

### `Worker`负责将代码从`ThreadPool`发送给线程

标准库提供了`thread::spawn`作为创建线程的方式，接受线程创建后执行的代码块为参数。然而我们只想创建线程并让它们等待我们发送给它的任务。标准库的线程实现没有提供这种创建线程的方式。我们需要在`ThreadPool`和线程之间引入一个新的数据结构来做这件事，我们称这个结构体为`Worker`，是一个在线程池的实现中常见的术语。`Worker`选择要运行的代码然后在线程中运行。我们会保存一个`Worker`的实例，每个`Worker`会保存一个`JoinHandler<()>`实例。我们会给每个`Worker`一个`id`，在记录日志和debug时可以区分不同的Worker实例。以下是创建`ThreadPool`时的过程

1. 定义一个`Worker`结构体，保存`id`和`JoinHandler<()>`
2. 修改`ThreadPool`来保存`Worker`实例
3. 定义一个`Worker::new`函数接收`id`为参数，返回一个`Worker`实例，保存`id`和一个空闭包启动的线程
4. 在`ThreadPool::new`中，使用`for`来生成`id`，创建新的`Worker`实例，将`worker`保存到vector中

```rust
use std::thread;

pub struct ThreadPool {
    workers: Vec<Worker>,
}

impl ThreadPool {
    // --snip--
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id));
        }

        ThreadPool { workers }
    }
    // --snip--

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
    }
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize) -> Worker {
        let thread = thread::spawn(|| {});

        Worker { id, thread }
    }
}
```

外部代码不需要知道`ThreadPool`中`Worker`结构体的实现细节，所以我们让`Worker`结构体和它的`new`函数保持私有。`Worker::new`函数使用我们给定的`id`，存储`JoinHandle<()>`实例。如果操作系统因为系统资源不够无法创建线程，那么`thread::spawn`会崩溃，这会导致我们整个服务器崩溃。学习示例中可以这样写，但是在生产环境的线程池实现中，你应该使用`std::thread::Builder`的`spawn`方法，它会返回`Result`让你做进一步的错误处理

### 将请求通过管道发送给线程

我们想让创建的`Worker`结构体从`ThreadPool`队列获取代码然后将其发送到自己的线程运行。我们之前学习的管道是一种简单的线程间通信方式，很适合当前的情况。我们使用一个管道来做工作队列，`execute`会从`ThreadPool`发送任务到`Worker`实例

1. `ThreadPool`会创建一个管道，并且持有发送者
2. 每个`Worker`会在接收者阻塞
3. 我们会创建一个新的`Job`结构体，包括我们想发送给管道的闭包
4. `execute`方法会通过发送者发送它想执行的工作
5. 在线程中，`Worker`会遍历接收者然后执行它接收到的任何任务的闭包

```rust
use std::{sync::mpsc, thread};

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

struct Job;

impl ThreadPool {
    // --snip--
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id));
        }

        ThreadPool { workers, sender }
    }
    // --snip--

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
    }
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize) -> Worker {
        let thread = thread::spawn(|| {});

        Worker { id, thread }
    }
}
```

在线程池创建管道的时候，将管道的接收者传入每个`Worker`。我们要在`Worker`实例启动的线程中使用接收者，所以我们需要在闭包中引用`receiver`

```rust
use std::{sync::mpsc, thread};

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

struct Job;

impl ThreadPool {
    // --snip--
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, receiver));
        }

        ThreadPool { workers, sender }
    }
    // --snip--

    pub fn execute<F>(&self, f: F)
        where
        F: FnOnce() + Send + 'static,
        {
        }
}

// --snip--

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: mpsc::Receiver<Job>) -> Worker {
        let thread = thread::spawn(|| {
            receiver;
        });

        Worker { id, thread }
    }
}
```

Rust提供的管道模式是多生产者单消费者，所以我们不能克隆管道的消费者。我们也不想向多个消费者发送多次任务，我们需要一个任务列表，多个`Worker`实例可以每次处理一条消息。此外将一个任务从队列中取出会修改`receiver`，所以线程需要以安全的方式来共享和修改`receiver`，否则会产生竞争条件。要在多个线程中共享所有权，并且让线程修改值，我们需要使用`Arc<Mutex<T>>`。`Arc`类型让多个`Worker`实例拥有接收者，`Mutex`可以保证一次只有一个`Worker`可以从接收者获取工作

```rust
use std::{
    sync::{Arc, Mutex, mpsc},
    thread,
};
// --snip--

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

struct Job;

impl ThreadPool {
    // --snip--
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    // --snip--

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
    }
}

// --snip--

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        // --snip--
        let thread = thread::spawn(|| {
            receiver;
        });

        Worker { id, thread }
    }
}
```

在`ThreadPool::new`的签名中，我们将接收者放到`Arc`和`Mutex`中。对每个`Worker`，我们克隆`Arc`来增加引用计数，`Worker`实例可以共享接收者的所有权

### 实现`execute`方法

我们将`Job`变成类型别名，对应一个特质对象，表示`execute`接收的闭包的类型

```rust
use std::{
    sync::{Arc, Mutex, mpsc},
    thread,
};

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

// --snip--

type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    // --snip--
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(job).unwrap();
    }
}

// --snip--

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(|| {
            receiver;
        });

        Worker { id, thread }
    }
}
```

从`execute`接收的闭包创建新`Job`，将任务发送到管道的发送者。调用`send`的`unwrap`处理发送失败的情况。当我们停止了所有执行的线程，即接收者停止接收新消息后，这种情况可能发生。但现在我们不能停止线程：只要池还存在我们的线程就会执行，所以我们可以确定这里不会失败，但是编译器不知道。我们需要`Worker`中的`thread::spawn`的闭包永远循环，让管道的接收者在任务到达时处理任务

```rust
use std::{
    sync::{Arc, Mutex, mpsc},
    thread,
};

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(job).unwrap();
    }
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

// --snip--

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || {
            loop {
                let job = receiver.lock().unwrap().recv().unwrap();

                println!("Worker {id} got a job; executing.");

                job();
            }
        });

        Worker { id, thread }
    }
}
```

我们调用`receiver`的`lock`得到互斥量，然后调用`unwrap`，出现任何错误直接退出程序。如果互斥量处于有毒（`poisoned`）状态，获取锁会失败，如果其它线程在持有锁时崩溃且没有正常释放锁就会发生这种情况，此时调用`unwrap`让线程崩溃是正确的动作。获取锁后，调用`recv`接收`Job`，最后使用`unwrap`直接处理错误，如果持有发送者的线程关闭，发送者被清理，这里的调用就会报错，和接收端关闭后`send`方法返回`Err`是一样的。`recv`的调用是阻塞的，所以如果没有任务，当前线程会等待任务。`Mutex<T>`可以保证同时只有一个`Worker`线程获取任务

这里不使用`while-let`的原因是，这样做依然会导致慢请求阻塞其它的请求，原因是`Mutex`没有提供`unlock`方法，而锁的生命周期是基于`LockResult<MutexGuard<T>>`的`MutexGuard<T>`的生命周期。在编译期借用检查器会强制应用规则，`Mutex`可以保证除非我们持有锁，否则不被允许访问资源，如果我们不注意`MutexGuard<T>`的生命周期，这个保证可能会让我们持有更长时间的锁，`let job = recevier.lock().unwrap().recv().unwrap()`对于let，当语句结束，右边表达式的值会立马被丢弃。然而`while let`（`if let`和`match`）在关联语句块结束前都不会释放临时值

## 优雅关闭和清理工作

当我们使用ctrl-c的方式关闭进程时，所有其它线程会立即停止，即使它们还在服务状态。之后我们会通过实现`Drop`特质，让线程池的每个线程上调用`join`，使它们在程序结束前可以完成正在处理的任务。我们会实现一种方式告诉线程它们应该停止处理新的任务然后关闭

### 为`ThreadPool`实现`Drop`特质

当线程池池被删除时，我们的线程应该调用`join`保证它们已经完成当前工作

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            worker.thread.join().unwrap();
        }
    }
}
```

使用`&mut`是因为我们需要使用`worker`可变引用做后续操作。这段代码无法通过编译，因为`join`需要获取参数的所有权。为了解决这个问题，我们需要将线程移出`Worker`实例，一种方法是让Worker持有`Option<thread:JoinHandler<()>>`,然后调用`Option`的`take`方法来获取值的所有权，但是这种方式只在`Worker`被清理的时候有用，我们还需要处理所有使用`worker.thread`的地方

⚠️当你发现为了规避某个问题需要将你知道永远存在的东西放在`Option`中的时候，你需要使用其它的方法

本例中，更好的方式是使用`Vec::drain`方法。它接收一个`range`参数来指定哪些值被移出`Vec`，然后返回这些项的迭代器

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        for worker in self.workers.drain(..) {
            println!("Shutting down worker {}", worker.id);

            worker.thread.join().unwrap();
        }
    }
}
```

### 通知线程停止监听工作

现在代码还不能以我们期望的方式运行。虽然调用了`join`，因为线程中有`loop`循环，所以线程不会被关闭，主线程会永远阻塞。首先我们要修改`ThreadPool`的`drop`方法，在等待线程停止之前显式清理`sender`。这里我们需要使用`Option::take`将`sender`从`ThreadPool`移出

```rust
pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: Option<mpsc::Sender<Job>>,
}
// --snip--
impl ThreadPool {
    pub fn new(size: usize) -> ThreadPool {
        // --snip--

        ThreadPool {
            workers,
            sender: Some(sender),
        }
    }

    pub fn execute<F>(&self, f: F)
        where
        F: FnOnce() + Send + 'static,
        {
            let job = Box::new(f);

            self.sender.as_ref().unwrap().send(job).unwrap();
        }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        drop(self.sender.take());

        for worker in self.workers.drain(..) {
            println!("Shutting down worker {}", worker.id);

            worker.thread.join().unwrap();
        }
    }
}
```

清理`sender`的操作会关闭管道，没有更多的任务被发送。在`Worker`实例中的无限循环中对`recv`的调用会返回错误

```rust
impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || {
            loop {
                let message = receiver.lock().unwrap().recv();

                match message {
                    Ok(job) => {
                        println!("Worker {id} got a job; executing.");

                        job();
                    }
                    Err(_) => {
                        println!("Worker {id} disconnected; shutting down.");
                        break;
                    }
                }
            }
        });

        Worker { id, thread }
    }
}
```

让`main`中在优雅关闭服务器之前只接收两个请求

```rust
fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming().take(2) {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }

    println!("Shutting down.");
}
```

这里只是为了展示优雅关闭和清理资源的过程。`take`方法定义在`Iterator`特质中，限制最多迭代两个项。`ThreadPool`在`main`结束后离开域，执行`drop`中的代码

```bash
$ cargo run
   Compiling hello v0.1.0 (file:///projects/hello)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.41s
     Running `target/debug/hello`
Worker 0 got a job; executing.
Shutting down.
Shutting down worker 0
Worker 3 got a job; executing.
Worker 1 disconnected; shutting down.
Worker 2 disconnected; shutting down.
Worker 3 disconnected; shutting down.
Worker 0 disconnected; shutting down.
Shutting down worker 1
Shutting down worker 2
Shutting down worker 3
```

`ThreadPool`的`Drop`实现甚至会在`Worker3`开始工作前执行。`Worker`实例在被清理时打印消息，然后线程池调用了`join`等待每个`Worker`线程结束。`ThreadPool`清理了`sender`，在`Worker`接收到错误之前，我们尝试执行`Worker0`的`join`。`Worker0`还没有从`recv`得到任何错误，所以主线程阻塞等待`Worker0`关闭，同时`Worker3`接收到任务，然后所有线程接收到错误。当`Worker0`结束，主线程等待剩余的`Worker`实例结束，最后它们都退出了循环并结束

扩展项目还可以做这些事情

- 为`ThreadPool`和它的公共方法添加更多文档
- 添加库函数的测试
- 将`unwrap`调用修改为更健壮的错误处理代码
- 使用`ThreadPool`来执行一些任务来服务web请求
- 在crates.io上找到一个线程池crate，然后实现一个类似的web服务器。比较它的API和我们实现的线程池的健壮性
