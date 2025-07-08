# 第十六章 - 无畏并发

**并发编程**（*concurrent programming*）是程序的不同部分独立运行，**并行编程**（*parallel programming*）是程序的不同部分同时运行，随着计算机的多核能力更强，这两种编程方式也变得更加重要

所有权和类型系统是一个可以帮助管理内存安全和并发问题的强大工具集。利用这两部分内容，很多并发错误在Rust中会变成编译错误，Rust称这些内容为*fearless concurrency*，让你编写出bug更少的、在不引入新bug的情况下易于重构的代码

对于高级编程语言来说，仅支持某种解决方案中的一部分是合理的，因为高级语言通过放弃实现一些内容来获得更好的抽象。而低级语言希望提供在任何情况下都能获得最佳性能的解决方案，并且对硬件做尽可能少的抽象

## 使用线程来同时运行代码

目前在多数操作系统中，执行程序的代码运行在进程中，操作系统同时管理多个进程。在一个程序中，你也可以同时运行那些独立的部分代码。这种特性叫做**线程**（*threads*），例如一个webserver会使用多个线程，所以它能同时响应多个请求。多个线程可以同时运行，但是不会保证不同线程中代码的运行顺序，这会导致以下问题

- **竞争条件**（*race condition*），线程会以随机的顺序来访问数据或资源
- **死锁**（*dead lock*），如果两个线程互相等待对方，那么两个线程都无法继续执行
- 只有特定情况下会出现的bug，很难通过复现来修复

编程语言会以不同的方式实现线程，很多线程相关的操作都提供了语言可以调用的API。Rust标准库使用了**1:1模型**的线程实现，即一个语言中的线程直接对应一个操作系统线程。也有其它crate实现了线程的其它模型

### 使用`spawn`创建一个新线程

要创建一个新线程，调用`thread::spawn`并传入一个闭包，包含我们要在线程中运行的代码

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {i} from the spawned thread!");
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {i} from the main thread!");
        thread::sleep(Duration::from_millis(1));
    }
}
```

注意，当Rust程序的主线程结束后，所有创建的子线程都会被终止，无论它们是否完成。`thread::sleep`强制当前线程在一个时间范围内停止运行，让另一个线程运行。线程可能会交替执行，但不保证执行顺序，取决于操作系统如何排布线程

### 使用`join`处理来等待所有线程运行完成

我们可以通过保存`thread::spawn`的返回值到一个变量中，来修复新建线程尚未运行或被提前终止的问题。返回值的类型是`JoinHandler<T>`，它是一个自有值，当我们调用它的`join`方法时，就会等待这个线程完成

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {i} from the spawned thread!");
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {i} from the main thread!");
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap();
}
```

`join`会阻塞当前正在运行的线程，直到它所代表的线程执行结束。**阻塞**（*Blocking*）一个线程表示该线程不能继续执行代码或者退出。如果把`join`放到主线程的`for`之前调用，代码结果就是子线程执行完成后再执行主线程

### 在线程中使用`move`闭包

我们会经常使用带有`move`关键字的闭包传入`thread::spawn`，因为闭包会获取从环境中的值的所有权，因此需要将值的所有权从一个线程移动到另一个

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("Here's a vector: {v:?}");
    });

    handle.join().unwrap();
}
```

上面的代码中，Rust推断（infer）闭包捕获`v`的方式，因为`println!`只需要`v`的引用，所以闭包尝试借用`v`，但是因为编译器无法确定新线程会运行多长时间，所以它不知道v在使用的时候是否仍然有效。如果在`spawn`之后调用`drop(v)`，当子线程开始执行时，`v`已经不再有效，所以它的引用同样是无效的。通过在闭包前添加`move`关键字，我们强制闭包捕获它使用的值的所有权

## 使用消息传递的方式在线程之间发送数据

用**消息传递**（*message passing*）的方式保证并发安全是一个越来越受欢迎的方式，其中线程或者**行动者**（*actor*）通过互相发送包含数据的消息来进行通信。Rust标准库提供了**管道**（*channel*）的实现。管道是通用的编程概念，数据通过它从一个线程被发送到另一个线程。管道分为两部分，一个发送者端和一个接收者端。发送者端位于上游，就像你把橡皮鸭子放到河里的上游位置，接收者端就是橡皮鸭子在下游的停止位置。你代码的一部分在发送端调用方法，发送数据，另一部分检查接收端获取数据。发送端或者接收端杯清理后，这个管道被认为已经关闭

```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
}
```

使用`mpsc::channel`函数创建一个新管道。*mpsc*表示**多生产者，单消费者**（*multiple producer, single consumer*）。Rust标准库实现的管道支持多个发送端，但是只能有一个接收端消费这些值。`mpsc::channel`函数返回一个元组，表示发送端和接收端。`tx`和`rx`是传统地表示*transmitter*和*receiver*的缩写

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });
}
```

新线程需要拥有发送端实例才能通过管道发送消息。发送端的`send`方法的参数是要发送的值，返回类型为`Result<T, E>`，如果接收端已经被清理，由于没有地方可以发送，`send`操作会返回一个错误

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {received}");
}
```

接收端提供两个有用的方法`recv`和`try_recv`，`recv`会阻塞当前线程执行，直到有值从管道传来。接收到值后，`recv`会返回`Result<T, E>`类型的值，当发送端关闭，`recv`会返回一个错误表示没有更多的值会到来。`try_recv`不会阻塞，而是直接返回`Result<T, E>`，`Ok`表示如果接收到值就会包含这个值，如果没有接受到值就会返回`Err`。如果当前线程在接收消息时有其它任务，可以使用`try_recv`，通常我们会使用一个循环调用`try_recv`，如果有消息就处理，没有就做其它的工作

### 管道和所有权传递

在写Rust程序的过程中思考所有权可以避免出现很多并发编程错误

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
        println!("val is {val}");
    });

    let received = rx.recv().unwrap();
    println!("Got: {received}");
}
```

当消息发送给其它线程后，接收端所在的线程可能在我们尝试再次使用这个值之前修改或者清理它。由于其它线程对值的修改可能导致错误或者非预期的结果，出现不一致或者不存在的数据，所以上面的代码无法通过编译

### 发送多个值，接收端等待处理

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {received}");
    }
}
```

我们可以把`rx`当作迭代器来使用，当管道关闭后，迭代也结束了

### 通过克隆发送端来创建多个生产者

```rust
let (tx, rx) = mpsc::channel();

let tx1 = tx.clone();
thread::spawn(move || {
    let vals = vec![
        String::from("hi"),
        String::from("from"),
        String::from("the"),
        String::from("thread"),
    ];

    for val in vals {
        tx1.send(val).unwrap();
        thread::sleep(Duration::from_secs(1));
    }
});

thread::spawn(move || {
    let vals = vec![
        String::from("more"),
        String::from("messages"),
        String::from("for"),
        String::from("you"),
    ];

    for val in vals {
        tx.send(val).unwrap();
        thread::sleep(Duration::from_secs(1));
    }
});

for received in rx {
    println!("Got: {received}");
}
```

在创建第一个子线程时，我们调用了发送端的`clone`方法，得到一个新的发送端实例，然后我们将原始的发送端传入第二个子线程中，此时两个线程可以使用不同的发送端发送消息

## 共享状态并发

在所有编程语言中，管道的概念就是单一所有权，一旦你将值传入管道，你就不应该再次使用这个值。使用共享状态执行并发操作像多所有权，多个线程可以同时访问相同的内存空间

### 使用互斥量限制同一时间只有一个线程访问数据

*Mutex*是*mutual exclusion*的缩写，表示在任意时间点只能有一个线程来访问它保护的数据。要在一个互斥量中访问数据，一个线程需要先声明它要获得互斥量的**锁**（*lock*）来访问数据。锁是一种表示互斥量的数据结构，跟踪当前是谁在互斥地访问数据。因此互斥量被描述为**保护**（*guarding*）通过锁系统持有的数据

互斥量很难用好，你必须记住这两条规则

1. 在使用数据前你必须尝试获得锁
2. 当你在互斥量的保护下使用完数据，你必须解锁数据，其它的线程才能获得锁

### `Mutex<T>`的API

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    {
        let mut num = m.lock().unwrap();
        *num = 6;
    }

    println!("m = {m:?}");
}
```

使用`new`来创建一个`Mutex<T>`，我们要使用`lock`方法来获取锁以访问其中的值，这个调用会阻塞当前线程，在获取到锁之前不能做任何工作。如果其它持有锁的线程崩溃，`lock`方法会失败，没有线程能再次获取锁，如果是这种情况，我们要使用`unwrap`来让当前线程崩溃。获取锁后，我们可以处理它的返回值，例子中是`num`，是内部数据的可变引用。`m`的类型是`Mutex<i32>`，我们必须调用`lock`来使用其中的`i32`值

严格地说，`lock`方法返回一个智能指针，叫做`MutexGuard`，它被包裹在`LockResult`中，我们通过`unwrap`获得智能指针的值。`MutexGuard`智能指针实现了`Deref`特质，可以访问内部数据，也实现了`Drop`特质，当它离开域后会自动释放锁。我们不用担心忘记释放锁导致使用互斥量的其它线程阻塞

### 在多个线程之间共享`Mutex<T>`

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handles = vec![];

    for _ in 0..10 {
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

`counter`值在之前的循环中已经被移动，编译器会提示我们不能移动`counter`到多个线程中

### 多所有权结合多线程

```rust
use std::rc::Rc;
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Rc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Rc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

`Rc<T>`在线程之间共享是不安全的，`Rc<T>`管理引用计数时，每次`clone`调用会增加计数，每个克隆被清理时减少计数。但是它没有使用任何并发原语来保证计数操作不会被其它线程中断，这会导致计数错误，从而造成内存泄漏或者值提前被清理的结果

### 使用`Arc<T>`进行原子引用计数

`Arc<T>`是和`Rc<T>`类似的类型，但是保证并发安全。*a*表示*atomic*，是**原子引用计数类型**（*atomically reference-counted*），原子性是另一个并发原语，可以查看`std::sync::atomic`的文档来了解详情

为什么所有的基本类型不直接使用原子性的？为什么标准库类型不默认使用`Arc<T>`实现？原因是为了保证操作的线程安全，代价是运行时产生的性能开销，你只有在真的需要这些特性时才想承担代价。如果你只是在单线程中执行操作，那么没有这些线程安全的保证可以让你的代码执行的更快

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

### `RefCell<T>`/`Rc<T>`和`Mutex<T>`/`Arc<T>`之间的相似性

`Mutex<T>`也提供了内部可变性，和`Cell`家族的功能一样。Rust不会在你使用`Mutex<T>`时保证你不会遇到各种类型的逻辑错误。就像使用`Rc<T>`可能会创造循环引用，导致内存泄漏一样。`Mutex<T>`可能会造成死锁。当一个操作需要等待两个资源上的锁，而两个线程都需要对方的锁时会发生死锁

## 通过`Send`和`Sync`特质来扩展并发性

之前讨论过的并发操作都是标准库的内容，不是语言本身提供的。你处理并发编程的方式不局限于语言或者标准库，你可以编写自己的并发特性或者使用其他人写的

Rust语言中核心的并发概念是`std::marker`中定义的特质，`Send`和`Sync`

### 使用`Send`保证线程之间所有权的传递

`Send`标记特质标识那些实现了`Send`的类型的值的所有权可以在线程之间传递，几乎所有Rust的类型都实现了`Send`，有一些例外，包括`Rc<T>`，它不能实现`Send`，因为如果克隆了`Rc<T>`并且将它的克隆的所有权转移到其它线程，那么两个线程可以同时更新它的引用计数。这个原因导致`Rc<T>`只能在单线程环境下使用

任何全部由实现了`Send`特质的类型组成的类型会被自动标记为实现`Send`，几乎所有的基本类型都实现了`Send`，包括裸指针

### 使用`Sync`允许多线程访问

`Sync`标记特质标识那些实现了`Sync`特质的类型在多个线程中被引用是安全的。换句话说，任何实现了`Sync`的类型`T`，如果`&T`实现了`Send`，那么引用类型可以安全地在线程之间传递。和`Send`类似，基本类型都实现了`Sync`特质，由所有实现了`Sync`特质的类型组成的类型自动标记为实现`Sync`特质

智能指针类型`Rc<T>`也没有实现`Sync`，理由和它没有实现`Send`一样。`RefCell<T>`类型和关联的`Cell<T>`家族类型没有实现`Sync`。`RefCell<T>`在运行时检查借用规则不是线程安全的。`Mutex<T>`实现了`Sync`所以可以在多线程中共享访问

### 手动实现`Send`和`Sync`是不安全的

通常我们不需要手动实现这两个特质，作为标记特质，它们也没有方法用来实现，它们只是用来强制应用和并发有关的**不变量**（*invariants*）。手动实现这些特质就是在实现不安全的Rust代码。构建那些不是由`Send`和`Sync`构成的新的并发类型时，要小心实现各种安全保障
