# 第十二章 - I/O项目：一个命令行程序

本章的任务是构建一个命令行搜索工具**grep**（*globally search a regular expression and print*），接受文件路径和一个字符串作为参数，读取文件内容，找到文件中包含这个字符串的所有行，然后打印这些行

## 接收命令行参数

项目命名为minigrep，使用`cargo new`创建项目。第一件事是接受两个命令行参数：文件名和要搜索的字符串。`$ cargo run -- searchstring example-filename.txt`

### 读取参数的值

`std::env::args`函数会返回传入程序的参数构成的迭代器，目前对于迭代器只需要了解：迭代器会创建一些值，我们可以在迭代器上调用`collect`方法来将其转化为包含所有迭代器中的元素的集合，例如vector

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    dbg!(args);
}
```

`args`函数嵌套在两层模块中，我们将这个函数的父模块引入域中。这样可以使用`std::env`中其它的函数，如果直接将这个函数名引入域中，很容易会和定义在当前模块中的同名函数弄混。如果提供的参数中包含无效的Unicode，这个函数会崩溃。如果你的程序需要接收可能包含无效的Unicode参数，可以使用`std::env::args_os`。这个函数返回的是产生`OsString`的值的迭代器。`collect`是一个你经常需要注解返回值类型的函数，因为Rust无法推断你想要的集合类型

### 将参数值存入变量

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    let query = &args[1];
    let file_path = &args[2];

    println!("Searching for {query}");
    println!("In file {file_path}");
}
```

### 读取一个文件

在项目顶层目录创建一个文本文件`poem.txt`。然后在代码中读取该文件

```rust
use std::env;
use std::fs;

fn main() {
    // --snip--
    println!("In file {file_path}");

    let contents = fs::read_to_string(file_path)
        .expect("Should have been able to read the file");

    println!("With text:\n{contents}");
}
```

`fs::read_to_string`接收`file_path`参数，打开这个文件，返回`std::io::Result<String>`类型的值，`Ok`中会包含文件内容

### 重构来改善模块化和错误处理

现在尝试改善我们的程序，修复当前的程序结构产生的四个问题和它们可能导致的潜在错误

1. `main`函数现在执行两个任务，转换参数和读取文件。随着程序变长，`main`函数中的任务数量会增加。随着函数的责任越来越多，它更难去分析、测试，在不改变其它部分的条件下进行调试。最好分离其中的功能使每个函数负责一个任务
2. `query`和`file_path`是我们程序的参数，`contents`这种变量是用来执行程序的逻辑。`main`函数随着我们需要引入域的变量增多，这些变量更难分辨出它们的目的。最好将配置变量放到一个结构体中让它们的目的更清晰
3. 如果读取文件失败，我们使用`expect`来打印错误信息，但是提供的错误信息很笼统。读取文件失败的原因可能有很多，我们打印这条信息对于用户没有任何帮助
4. 我们使用`expect`来处理错误，如果用户没有指定足够多的参数，他们会得到*index out of bounds*错误，这条信息没有清楚地解释问题。最好将所有错误处理的代码放在一个地方，未来的维护者如果想修改错误处理的代码，只需要修改这个地方即可

### 对于二进制项目的职责分离

Rust社区对于`main`函数增长的二进制crate的职责分离步骤开发了一个导航原则

1. 将程序分为`main.rs`文件和`lib.rs`文件，将逻辑放入后者
2. 如果你的命令行处理逻辑很少，它可以保留在`main.rs`中
3. 当命令行处理逻辑变得复杂，将它移动到`lib.rs`中

经过这三步后，main函数应该保留以下内容

- 根据参数值调用命令行处理逻辑
- 设置其它的配置
- 调用`lib.rs`的`run`函数
- 如果`run`返回错误，处理返回的错误

之前无法直接测试`main`函数，现在这种结构可以测试程序的逻辑了

#### 分离参数处理器

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let (query, file_path) = parse_config(&args);

    // --snip--
}

fn parse_config(args: &[String]) -> (&str, &str) {
    let query = &args[1];
    let file_path = &args[2];

    (query, file_path)
}
```

`main`函数不再负责处理命令行参数和变量的关系，修改完可以重新运行程序来验证参数处理可以正常工作。经常检查你的代码改动可以帮助你在错误发生时尽早找到原因

#### 组织配置的值

目前我们处理参数的函数返回的是元组，但是我们很快就将元组解构为各个部分，这是一个标志，表明我们可能没有做好正确的抽象。我们返回的这两个值是有联系的，它们都是配置的一部分。但是我们目前的数据结构没有传达这一含义。我们要将这两个值放入一个结构体，然后给每个成员一个有意义的名称。这样会让代码未来的维护者了解这些不同的值是如何互相关联的，还有它们的目的是什么

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = parse_config(&args);

    println!("Searching for {}", config.query);
    println!("In file {}", config.file_path);

    let contents = fs::read_to_string(config.file_path)
        .expect("Should have been able to read the file");

    // --snip--
}

struct Config {
    query: String,
    file_path: String,
}

fn parse_config(args: &[String]) -> Config {
    let query = args[1].clone();
    let file_path = args[2].clone();

    Config { query, file_path }
}
```

这里我们使用`clone`方法得到自有的`Config`实例，花费了更多的时间和内存。但是复制数据的动作也让代码表达了更直接的含义，并且不需要再管理引用的生命周期，这种情况下，放弃一些性能换取代码的简洁性是一个好的取舍

> 使用`clone`的取舍
>
> 在很多Rustaceans中存在一个观念，那就是避免使用`clone`来处理所有权问题，因为它有运行时性能损耗。但是在这个示例程序中，这些拷贝只发生一次，并且字符串内容很少。在最开始，拥有一个可以工作的效率不高的程序要比尝试极致优化的程序更好

#### 为Config创建一个构造器

因为`parse_config`的目的是创建一个`Config`实例，我们可以将它从普通函数修改为关联`Config`结构体的`new`方法，这让代码更符合语言习惯

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args);

    // --snip--
}

// --snip--

impl Config {
    fn new(args: &[String]) -> Config {
        let query = args[1].clone();
        let file_path = args[2].clone();

        Config { query, file_path }
    }
}
```

#### 修复错误处理

如果程序的命令行参数少于两个，现在的程序就会报错，但是错误信息并不能向用户说明情况。我们需要修改错误信息

```rust
// --snip--
fn new(args: &[String]) -> Config {
    if args.len() < 3 {
        panic!("not enough arguments");
    }
    // --snip--
```

现在错误输出的内容更好了，但是还有不想让用户看到的的无关信息，例如`thread 'main' panicked at src/main.rs:26:13`。所以调用`panic!`更适用于编程问题而不是使用问题

#### 返回一个`Result`而不是调用`panic!`

我们将`new`改为`build`，因为很多程序员都会期望`new`函数不会失败

```rust
impl Config {
    fn build(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let file_path = args[2].clone();

        Ok(Config { query, file_path })
    }
}
```

错误包含值总是有`'static`生命周期的字符串字面值

#### 调用`Config::build`然后处理错误

我们用以非零值退出程序来替换`panic!`的作用。以非零值退出程序是一个惯例，来通知调用程序的进程程序以错误状态退出

```rust
use std::process;

fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::build(&args).unwrap_or_else(|err| {
        println!("Problem parsing arguments: {err}");
        process::exit(1);
    });

    // --snip--
```

使用`unwrap_or_else`让我们定义非`panic!`错误处理。如果`Result`是`Ok`值，这个方法和`unwrap`一样，如果值是`Err`值，这个方法会调用闭包中的代码，这个函数会向闭包传递`Err`中的值。`process::exit`函数会立即停止程序，返回传入的退出状态码

#### 从`main`中分离逻辑

```rust
fn main() {
    // --snip--

    println!("Searching for {}", config.query);
    println!("In file {}", config.file_path);

    run(config);
}

fn run(config: Config) {
    let contents = fs::read_to_string(config.file_path)
        .expect("Should have been able to read the file");

    println!("With text:\n{contents}");
}

// --snip--
```

#### 从`run`函数返回错误

run函数在发生错误时也应该返回`Result<T, E>`，这可以让我们以对用户友好的方式在`main`函数中处理错误

```rust
use std::error::Error;

// --snip--

fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.file_path)?;

    println!("With text:\n{contents}");

    Ok(())
}
```

现在只要知道`Box<dyn Error>`表示函数可以返回一个实现了`Error`特质的类型

#### 在`main`中处理`run`返回的错误

```rust
fn main() {
    // --snip--

    println!("Searching for {}", config.query);
    println!("In file {}", config.file_path);

    if let Err(e) = run(config) {
        println!("Application error: {e}");
        process::exit(1);
    }
}
```

之所以使用`if let`来处理，是因为不需要使用`run`成功运行的返回值，所以只需要处理错误分支就可以

#### 将代码分离到库crate中

将下面这些不在`main`函数中的代码移动到`src/lib.rs`中

- `run`函数定义
- 相关的`use`语句
- `Config`定义
- `Config::build`函数定义

## 使用TDD模式开发库的功能

使用TDD（测试驱动开发）过程来为程序添加搜索逻辑

1. 先写一个会失败的测试函数，保证它因为你预期的原因测试失败
2. 写/修改一个最小可执行代码让新测试函数通过
3. 重构你新增或修改的代码确保测试仍然能通过
4. 重复步骤1

### 写一个错误的测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn one_result() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.";

        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }
}
```

这个测试函数会失败，目前甚至还无法编译，因为不存在`search`这个函数

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    vec![]
}
```

这里的生命周期参数指定了返回值的vector中字符串切片的生命周期和`contents`是相关的

### 写代码来通过测试

搜索函数的逻辑是

1. 遍历内容的每一行
2. 检查该行是否包含查询的字符串
3. 如果包含就将其添加到返回的切片中
4. 如果不包含就什么都不做
5. 返回匹配列表

## 处理环境变量

我们将为程序添加新功能，用户可以通过**环境变量**来开启大小写不敏感搜索模式，这样用户只要设置一次环境变量，之后在这个终端会话中的所有搜索操作就都是大小写不敏感的了

### 写一个大小写不敏感版本的`search`函数的失败测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn case_sensitive() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Duct tape.";

        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }

    #[test]
    fn case_insensitive() {
        let query = "rUsT";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Trust me.";

        assert_eq!(
            vec!["Rust:", "Trust me."],
            search_case_insensitive(query, contents)
        );
    }
}
```

代码中修改了旧的`contents`，加上了“Duct tape”的内容，这样可以确保我们不会意外破坏大小写敏感的搜索功能

### 实现`search_case_insensitive`函数

```rust
pub fn search_case_insensitive<'a>(
    query: &str,
    contents: &'a str,
) -> Vec<&'a str> {
    let query = query.to_lowercase();
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.to_lowercase().contains(&query) {
            results.push(line);
        }
    }

    results
}
```

这个函数就是将待查询的字符串和查询内容都转换为小写后进行比较，因为`to_lowercase`函数只能处理简单的ascii码，所以真实应用需要处理Unicode的问题。我们要在`Config`中加上大小写敏感的开关，然后在`run`函数进行判断

```rust
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.file_path)?;

    let results = if config.ignore_case {
        search_case_insensitive(&config.query, &contents)
    } else {
        search(&config.query, &contents)
    };

    for line in results {
        println!("{line}");
    }

    Ok(())
}
```

然后我们需要处理环境变量的问题

```rust
use std::env;
// --snip--

impl Config {
    pub fn build(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let file_path = args[2].clone();

        let ignore_case = env::var("IGNORE_CASE").is_ok();

        Ok(Config {
            query,
            file_path,
            ignore_case,
        })
    }
}
```

因为我们不需要这个环境变量的值，所以使用`is_ok`来判断这个环境变量是否存在

## 将错误消息写入到标准错误而不是标准输出

大多数终端有两种输出，标准输出和标准错误，这个区别可以让用户选择将程序的成功输出转向一个文件，并且仍然能在终端打印错误信息

### 检查错误写入到哪里

我们目前的程序会看到错误信息也会被输出到文件中（如果我们把程序的结果转向一个文件），`>`语法告诉shell将标准输出的内容写入到`output.txt`而不是屏幕

### 将错误打印到标准错误

标准库提供了`eprintln!`宏来打印信息到标准错误
