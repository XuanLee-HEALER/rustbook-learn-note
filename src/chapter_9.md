# 第九章 - 错误处理

多数情况下，Rust需要你知道一个错误的可能值并且在编译前采取行动处理它。这个要求会让你的程序变得更加健壮，因为它保证你在将代码发布到生产环境前能发现错误并正确地处理它们

Rust中错误分为两大类：*recoverable*和*unrecoverable*。对于前者（例如file not found）我们大概只需要报告问题给用户，然后重试，但是后者几乎是bug的同义词，例如访问数组越界，所以我们希望程序立即停止

Rust中没有异常的概念，对于可恢复错误提供了`Result<T, E>`类型，对于不可恢复错误时有`panic!`宏来停止运行程序

## 不可恢复错误与`panic!`

当你的代码出错，并且你什么都做不了的时候，Rust提供了`panic!`宏。有两种方式会导致程序崩溃：

1. 执行会导致程序崩溃的动作（例如访问数组越界）
2. 直接调用panic!宏

程序崩溃后默认会打印错误信息、展开栈并清理栈，然后退出程序。通过设置环境变量，你可以让Rust在崩溃时打印调用栈来更容易地追踪崩溃的源头

### 展开调用栈或者终止程序来响应崩溃

默认情况下，程序崩溃时会开始unwinding，这意味着Rust会回溯调用栈然后清理它碰到的每个函数的数据，但是这是一项内容量很大的工作。所以Rust允许你选择立即终止程序，即不清理数据的情况下结束程序，程序之前使用的内存将会被操作系统清理，如果你的项目需要尽可能小，你可以将unwinding策略修改为终止策略（在崩溃发生的时候），通过在*Cargo.toml*文件的`[profile.release]`章节下中添加内容`panic = 'abort'`（示例为release模式）

手动调用`panic!`可以看到两行错误信息。第一行显示了崩溃信息和它在源码中的位置，`src/main.rs:2:5`表示在`src/main.rs`文件中第2行第5个字符。这种情况下显示的内容是我们的代码，其它情况下`panic!`可能存在于我们调用的代码中，文件名和行号会是那些代码中调用`panic!`的位置

```rust
fn main() {
    let v = vec![1, 2, 3];

    v[99];
}
```

这行代码执行后，Rust程序会崩溃。在C中尝试索引超过数据结构大小之后的数据是**未定义行为**。你可能会得到那块内存上的任何内容，即使这部分内存不属于这个结构，这叫做*buffer overread*，如果允许攻击者以这种方式操作索引来读他们不被允许访问的在该结构体之外数据，就会导致安全漏洞

`note:`告诉我们设置`RUST_BACKTRACE`环境变量可以打印导致这个错误的所有记录。*backtrace*是所有到达程序当前位置已经调用的函数列表，阅读这些内容的关键点是从上往下读，直到看到你写的代码文件出现。那是问题出现的地方，它上方的代码都是你的代码调用的代码，下面的代码是调用你的代码的代码。这些上下文可能会包含核心的Rust语言代码、标准库代码，或者你使用的其它crate。要查看这些记录信息，*debug symbol*必须开启，调试信号在非`--release`的`cargo build`和`cargo run`情况下是默认开启的

## 可恢复的错误与Result

很多错误并没有严重到需要程序完全停止，例如打开一个不存在的文件，你可能想创建一个新文件而不是直接终止程序

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`T`表示成功情况下，`Ok`变体中包含值的类型，`E`表示失败情况下在`Err`变体中包含错误的类型

```rust
use std::fs::File;

fn main() {
    let greeting_file_result = File::open("hello.txt");
}
```

`File::open`的返回类型是`Result<T, E>`，`T`的类型是`std::fs::File`，这是一个文件句柄。`E`的类型是`std::io::Error`。这个返回值表示，如果函数调用成功会返回一个文件句柄，我们可以使用它对文件进行读写。函数调用也可能失败，例如没有操作权限

```rust
use std::fs::File;

fn main() {
    let greeting_file_result = File::open("hello.txt");

    let greeting_file = match greeting_file_result {
        Ok(file) => file,
        Err(error) => panic!("Problem opening the file: {error:?}"),
    };
}
```

和`Option`类似，`Result`和它的变体也在预包含内容中，可以直接使用`Ok`和`Err`变体

### 在不同的错误上匹配

上面的代码中无论`File::open`失败的原因是什么程序都会崩溃退出。我们需要根据不同的错误原因来执行不同的操作，所以我们可以使用`match`表达式

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let greeting_file_result = File::open("hello.txt");

    let greeting_file = match greeting_file_result {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {e:?}"),
            },
            other_error => {
                panic!("Problem opening the file: {other_error:?}");
            }
        },
    };
}
```

因为`Err`变体内的值的类型是`io::Error`，它是标准库提供的类型。它有一个`kind`方法，调用可以得到`io::ErrorKind`类型的值。这个枚举是标准库提供并且有表示不同类型的错误的变体。其中我们使用了`ErrorKind::NotFound`

### 其它处理`Result<T, E>`类型的方法

`Result<T, E>`类型定义了很多使用闭包作为参数的方法，这些方法的用法比`match`更精简。例如`unwrap_or_else(F)`，`F`接收一个闭包参数，如果`Result`是`Err`时，内部的错误值作为闭包参数，传入的闭包可以处理和上面代码相同的逻辑

### Panic的快捷方式：`unwrap`和`expect`

`unwrap`方法是一种之前`match`表达式写法的快捷方式，如果`Result`的值是`Ok`，这个方法会返回`Ok`中的值，如果是`Err`，它会调用`panic!`。`expect`方法可以让我们修改`panic!`的错误提示信息

```rust
use std::fs::File;

fn main() {
    let greeting_file = File::open("hello.txt")
        .expect("hello.txt should be included in this project");
}
```

### 传递错误

在一个函数实现中，当调用了一些可能会出错的代码，并且不想在函数中处理这些错误，而是直接返回给调用者让它来决定如何处理，这叫做**扩散**（*propagating*）错误，并且让调用方有更多控制权，调用者有比函数当前上下文更多的信息或者更详细的逻辑来判断错误该如何处理

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let username_file_result = File::open("hello.txt");

    let mut username_file = match username_file_result {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut username = String::new();

    match username_file.read_to_string(&mut username) {
        Ok(_) => Ok(username),
        Err(e) => Err(e),
    }
}
```

函数的返回值类型是`Result<String, io::Error>`，如果函数调用成功就会返回一个包含字符串的`Ok`变体，如果失败会得到一个包含`io::Error`的`Err`变体。之所以选择`io::Error`类型是因为函数中所有可能出现的错误类型都是这个。最后一个`match`表达式的返回值就是整个函数的返回值，所以不需要使用`return`。调用这个函数的代码会得到包含用户名的`Ok`变体或者包括`io::Error`的`Err`变体。由调用方来决定遇到错误时该如何处理，例如停止程序或者使用一个默认值，这个函数本身无法判断应该如何处理这些错误，所以我们直接将成功或者失败的信息向上传递

#### 传递错误的快捷方式：`?`操作符

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut username_file = File::open("hello.txt")?;
    let mut username = String::new();
    username_file.read_to_string(&mut username)?;
    Ok(username)
}
```

`Result`值后面的`?`和之前的`match`表达式作用几乎相同。如果`Result`是`Ok`，那么返回里面的值，否则就返回`Err`作为整个函数的返回值。`match`表达式和`?`操作符的区别在于，使用了`?`操作符的错误值会调用`from`函数，这个函数定义在标准库的`From`特质中，用来表示将一种类型转换为另一种类型。当`?`在调用`from`函数时，接收到的错误类型被转换为当前函数定义的返回值的错误类型。所以定义一种可以表示函数所有可能失败的情况的错误类型非常有用，即使错误原因有很多。例如这个函数返回的错误类型是`OurError`，如果我们定义了`impl From<io::Error> for OurError`来通过`io::Error`构建一个`OutError`的实例，那么`?`操作符就会调用`from`来转换错误类型。`?`操作符减少了很多冗余代码，让函数实现更简单，我们也可以使用链式调用让代码更精简

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();

    File::open("hello.txt")?.read_to_string(&mut username)?;

    Ok(username)
}
```

代码中的函数，标准库提供了API`fs::read_to_string("hello.txt")`

##### `?`能在哪使用

`?`操作符只能在那些返回值类型兼容`?`生效的值的函数中使用，因为`?`操作符会执行提前返回函数的操作。`?`也可以作用在`Option<T>`值上，可以在返回`Option`类型的函数中使用，行为和`Result`类似

```rust
fn last_char_of_first_line(text: &str) -> Option<char> {
    text.lines().next()?.chars().last()
}
```

##### 在中间使用`?`操作符可以以更精简的方式展示逻辑

`?`不能自动在`Result`和`Option`之间转换，可以使用`Result`上的`ok`方法和`Option`上的`ok_or`方法来显式转换

`main`函数因为是入口点和终止点，所以对它的返回值类型有一些限制。`main`也可以返回`Result<(), E>`

```rust
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let greeting_file = File::open("hello.txt")?;

    Ok(())
}
```

`Box<dyn Error>`类型是一个特质对象，现在可以将其简单理解为表示所有类型的错误。当`main`函数的返回类型是`Result<(), E>`，如果返回`Ok(())`，那么程序的退出码就是`0`，如果返回的是`Err`值，就会以非`0`值退出。`main`函数可以返回任何实现了`std::process::Termination`特质的类型，它包含了一个`report`函数，返回一个ExitCode

## To panic! or Not to panic

如果无论什么错误都调用`panic!`，那么调用代码就是无法恢复的。如果返回`Result`值，相当于给了调用代码选择权，调用方可以根据不同的返回值进行恢复，或者根据`Err`值来判断这是不可恢复的情况，然后它再调用`panic!`将你返回的可恢复错误转换成不可恢复错误。因此返回`Result`是定义的可能会失败的函数时更好的选择

对于写例子、原型代码和测试，直接panic要比返回`Result`更好

### 例子、原型代码和测试

当你要写一个例子来展示概念时，健壮的错误处理会让代码变得不清晰。在例子中，调用`unwrap`这种会使程序崩溃的方式是将其作为占位符来表示你之后会去处理的错误，根据后续的代码选择不同的处理方式

原型代码中`unwrap`和`expect`方法也很好用，在你决定如何处理错误之前，它们留下了清晰的标记，之后你可以使用错误处理代码替代它们让程序更加健壮

在测试中如果一个方法失败，那么通常你想让整个测试失败，即使那个方法不是测试的方法。因为`panic!`会影响测试是否被标记为失败

### 当你有比编译器更多的信息

当你有一些其它的逻辑来保证`Result`一定是`Ok`时，也适合调用`unwrap`或者`expect`，编译器并不理解这些逻辑。并且即使逻辑上调用这些方法不会失败，你仍然有机会来处理这个`Result`值。如果你可以自己保证代码永远不会得到`Err`值，那么可以调用`unwrap`，最好使用`expect`来指明它之所以永远不会`Err`的理由

```rust
use std::net::IpAddr;

let home: IpAddr = "127.0.0.1"
    .parse()
    .expect("Hardcoded IP address should be valid");
```

### 错误处理的指引

当你的程序可能会以错误状态结束时，最好直接让你的代码panic。*bad state*是一些假设、保证和协议以及不变值已经被破坏的状态，例如当无效值、矛盾值或者缺失值被传入你的代码，并且伴随一种或多种以下状况

- 坏状态是那些意料之外的事情，不是偶尔发生的情况，例如用户输入了错误格式的数据
- 后面的代码需要依赖非坏状态执行，而不是每一步都检查所有问题
- 没有很好的方式在你使用的类型中编码这些信息（18章）

如果有人调用你的代码，传入了无效值，如果可以的话，最好返回一个错误，这样库的用户可以根据错误的值选择处理方式。然而如果继续执行程序是不安全或者有害的，最好直接调用`panic!`，警告库的用户这是他们代码中的bug，让他们在开发阶段来修改。如果你在调用外部代码，当它返回一个你无法处理的无效状态是最好也调用`panic!`

如果**失败是可以预见的**，那么最好返回一个`Result`。例如**转换器**（*parser*）碰到了异常数据或者HTTP请求达到了速度限制，这些情况返回`Result`表示失败本身是一种可能的行为，调用代码必须决定如何处理这些情况。当你的代码在被传入无效值后执行的操作会让用户处于风险之中，你的代码首先应该验证传入的值是否合法，如果值不合法就应该panic。这主要是由于安全原因：尝试在无效数据上做操作会让你的代码暴露给攻击者，这也是为什么数组访问越界标准库会调用`panic!`

函数通常有**协议**（*contracts*）：它们的行为只有当输入符合特定要求时才会被保证合法。当协议被违反时panic非常重要，因为违反协议通常表示这是调用者的bug，这并不是你想让调用方显式处理的那一类错误。事实上这种错误也不需要调用方来处理，调用方的程序员需要修复他的代码。函数的协议如果违反会导致panic，这些内容应该在函数的API文档中写清楚

写太多错误检查会导致代码冗余。可以使用Rust的类型系统来让编译器做很多检查。如果你的函数以特定类型作为参数，你可以认为编译器会保证你肯定可以得到一个该类型的有效值。例如只要你的函数参数不是一个`Option`，那么你可以认为这个值一定是某个值而不是什么都不是

### 创建自定义类型来验证

以第2章的猜数游戏为例，我们并没有检查用户的输入数字在合理范围，将用户导向有效的猜测，并且在用户猜错之后告诉他们应该如何处理是对程序的加强

```rust
loop {
    // --snip--

    let guess: i32 = match guess.trim().parse() {
        Ok(num) => num,
        Err(_) => continue,
    };

    if guess < 1 || guess > 100 {
        println!("The secret number will be between 1 and 100.");
        continue;
    }

    match guess.cmp(&secret_number) {
        // --snip--
}
```

这种方式并不理想，如果程序必须使用1-100之间的数字，那么在很多函数中都需要做这种数据的判断，这样很啰嗦。我们可以创建一个新类型，将验证逻辑放到结构体的`new`函数中

```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {value}.");
        }

        Guess { value }
    }

    pub fn value(&self) -> i32 {
        self.value
    }
}
```

如果`value`未通过检查，我们直接调用`panic!`，警告调用方他们需要修复这个bug，因为创建`Guess`所需要的`value`必须在某个范围内。`value`方法只是一个`getter`。因为`value`属性是私有的，模块外代码不能直接去设置`value`的值，只能通过这个方法来获取值
