# tracing

`tracing`是一个框架，用于检测Rust程序，收集结构化、基于事件的诊断信息。核心API由`spans`、`events`和`subscribers`组成

## `Spans`

为了记录程序执行的流程，`tracing`引入了`spans`的概念。与代表某个时间点的日志行不同，跨度代表一个有开始和结束的**时间段**。当程序开始在某个上下文中执行或执行某个工作单元时，它会进入该上下文的跨度；当它停止在该上下文中执行时，它会退出该跨度。线程当前正在执行的`span`被称为该线程的**当前跨度**（*current span*）

```rust
use tracing::{span, Level};
let span = span!(Level::TRACE, "my_span");
// `enter` returns a RAII guard which, when dropped, exits the span. this
// indicates that we are in the span for the current lexical scope.
let_enter = span.enter();
// perform some work in the context of `my_span`...
```

> 在使用async/await语法的异步代码中。如果返回的drop guard跨等待点被持有，`Span::enter`可能会生成错误的trace

## `Events`

一个事件（Event）代表时间上的一个瞬间。它表示在追踪记录期间发生了某件事情。事件与非结构化日志代码发出的日志记录相似，但与典型的日志行不同，一个事件可以发生在`span`的上下文中

```rust
use tracing::{event, span, Level};

// records an event outside of any span context:
event!(Level::INFO, "something happened");

let span = span!(Level::INFO, "my_span");
let_guard = span.enter();

// records an event within "my_span".
event!(Level::DEBUG, "something happened inside my_span");
```

通常，事件用来表示`span`中的一个时间点，例如一个请求以指定状态码返回、从队列中取出了N个新的项等等

## `Subscribers`

As Spans and Events occur, they are recorded or aggregated by implementations of the Subscriber trait. Subscribers are notified when an Event takes place and when a Span is entered or exited. These notifications are represented by the following Subscriber trait methods:

当span和事件发生时，它们被记录和集成到`Subscriber`特质的实现上。当事件发生和一个`span`进入或退出时，`Subscriber`会被通知。这些通知通过以下方法来表示

* `event`，事件发生时调用
* `enter`，进入`span`时调用
* `exit`，退出`span`时调用

此外`subscriber`可以实现`enabled`函数，以根据描述每个`span`或事件的元数据（metadata）来过滤它们接收到的通知。如果对`Subscriber::enabled`的调用对于给定的元数据集合返回`false`，则该`subscriber`将不会收到相应的跨度或事件的通知。出于性能原因，如果没有当前活跃的订阅者通过返回`true`来表示对给定元数据集合的兴趣，那么相应的`span`或事件将永远不会被构造
