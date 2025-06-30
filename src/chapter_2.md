# 第二章 - 猜数游戏

本章内容是使用Rust实现一个猜数字游戏：程序会从1～100生成一个随机数，然后让参与者输入一个猜想的数字，程序接收到输入后，判断这个数对比生成的随机数是高了还是低了，如果和随机数相同，程序会打印祝贺信息然后退出，否则告诉参与者结果，让参与者继续猜数

## 创建一个新项目

使用Cargo创建项目：`cargo new guessing_game`。首先处理输入，检查输入是否为需要的形式

```rust
// 处理输入和输出
use std::io;

// fn声明一个新函数，()表示没有参数，{开始函数体
fn main() {
    // println!宏用来打印欢迎语
    println!("Guess the number!");

    println!("Please input your guess.");

    // 创建一个变量来保存用户输入
    // = 告诉Rust我们想绑定一些东西到变量上
    // String::new 的调用结果就是被绑定的值，这个函数返回String的一个实例，String是标准库提供的字符串类型，它是可增长的、UTF-8编码文本位
    let mut guess = String::new();

    io::stdin()
        .read_line(&mut guess)
        .expect("Failed to read line");

    println!("You guessed: {}", guess);
}
```

Rust有一些预定义在标准库中的项会被自动引入所有程序的所有域中。这个集合叫做*prelude*。如果你想使用的类型不在这个集合中，你必须使用`use`语句来将其显式引入到域中

### 使用变量存储值

Rust中变量默认是**不可变**的，即一旦我们给了变量一个值，这个变量的值就不能改变。要让变量可变，我们需要在变量名前加上`mut`

`::`语法表示`new`是`String`的**关联函数**（*associated function*）。这种函数是实现在类型上的，本例中就对应`String`。`new`函数创建一个新的、值为空的字符串

### 接收用户输入

`stdin`函数返回一个`std::io::Stdin`的实例，这个类型表示终端标准输入的handle（句柄），`.read_line(&mut guess)`是获取用户从标准输入中键入的内容并追加到字符串中，所以我们要把字符串作为参数传进去。字符串参数需要是可变的，这个方法才能修改它的内容。`&`表示参数是一个**引用**（*reference*），引用可以让我们在代码的很多地方访问一份数据，并且不需要在内存中将数据拷贝多次。目前只需要知道引用和变量一样，默认也是不可变的

### 使用`Result`处理可能的失败

当你使用`.method_name()`的方式调用方法时，用新行和额外的空格来将长的一行分为多行通常是好的方式。`read_line`返回的是`Result`值。`Result`是一个**枚举**（*enum*），这是一种可以有多个可能状态的类型，我们称每种可能状态为variant（变体）

`Result`变体有`Ok`和`Err`。前者表示操作成功，它包含成功情况下返回的值，后者表示操作失败，包含操作失败的原因和过程信息

`Result`类型的实例有`expect`方法，如果这个实例包含`Err`值，那么这个方法会导致程序退出然后显示你传入方法的字符串。如果它包含`Ok`值，这个方法会返回给你这个值

### 使用`println!`的占位符来打印值

`println!("You guessed: {}", guess);`中的`{}`是**占位符**（*placeholder*）。当打印一个变量的值时，变量名可以直接写在花括号中。当打印的是表达式的计算结果时，在格式化字符串中放好占位符，然后在后面写逗号分隔的要打印的表达式，顺序和占位符的出现顺序相同

## 生成一个秘密数字

### 使用Crate来获取更多的功能

一个crate是一组Rust源代码文件。我们正在构建的是binary crate，它可以编译为可执行程序，rand crate是一个library crate，它包含让其它程序使用的代码，本身不能编译为可执行程序

我们需要修改*Cargo.toml*来将rand crate作为依赖包含到项目中。在`[dependencies]`章节下添加`rand = "0.8.5"`。Cargo使用*Semantic Versioning*(SemVer)的版本号标准，实际上`0.8.5`是`^0.8.5`的缩写，表示至少为`0.8.5`并且不超过`0.9.0`。任何超过`0.9.0`版本的代码不保证存在跟旧版本相同的API

当我们添加新的外部依赖后，Cargo会从registry拉取依赖所需要的所有其他依赖的最新版本，registry是Crates.io的数据拷贝，这个网站是Rust生态中的开发者将他们的开源Rust项目发布给其他人使用的地方

### 使用*Cargo.lock*文件来保证可重用的构建内容

Cargo有机制来保证所有人每次能构建出相同的内容，Cargo只会使用指定的依赖版本。当你第一次运行`cargo build`时，Cargo会创建一个*Cargo.lock*文件，Cargo会找到所有满足条件的依赖的版本，将它们写入*Cargo.lock*文件，再次构建项目时，如果这个文件存在，就会使用里面指定的版本而不是重新查询依赖的版本。通常此文件也会被加入VCS中

### 更新Crate的版本

当你真的要更新一个crate的版本时，Cargo提供了`update`命令，它会忽略*Cargo.lock*文件，而是按照*Cargo.toml*文件的内容重新查找依赖的最新版本，然后Cargo会将这些新版本重新写入*Cargo.lock*文件中

## 生成一个随机数

```rust
use rand::Rng;

fn main() {
    // ...

    let secret_number = rand::thread_rng().gen_range(1..=100);

    // ...
}
```

`Rng`**特性**（*trait*）定义了随机数生成器需要实现的方法，要使用这些方法必须将这个trait引入域中。`rand::thread_rng`函数会返回一个在当前执行的线程中的随机数生成器，，并且操作系统会提供一个种子。调用`gen_range`方法，它接收一个range表达式作为参数，然后生成一个给定范围内的随机数

运行`cargo doc --open`会在本地构建当前项目以及所有依赖的文档，然后在默认浏览器中打开

## 比较两个数字

```rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    // --snip--

    println!("You guessed: {guess}");

    match guess.cmp(&secret_number) {
        Ordering::Less => println!("Too small!"),
        Ordering::Greater => println!("Too big!"),
        Ordering::Equal => println!("You win!"),
    }
}
```

`std::cmp::Ordering`类型也是枚举，`Less`、`Greater`、`Equal`表示比较两个值时的三种结果。`cmp`方法可以在任何能被比较的值上调用。它接受要比较的内容的引用作为参数，返回一个`Ordering`枚举值

一个`match`表达式由arms（分支）组成。每个分支包括要匹配的**模式**（*pattern*）和匹配成功后要运行的代码。Rust会将这个值**按顺序**和每个分支的模式比较。成功匹配到第一个模式后就不再匹配其它模式

上方的代码无法通过编译，因为mismatched types错误。Rust有一个强大的静态类型系统。同时它也有类型推断。`let mut guess = String::new()`会将`guess`推断为`String`。除非指定其它类型，Rust默认会为数字字面值指定`i32`类型

Rust允许我们使用新值来覆盖之前`guess`的值。**覆盖**（*shadowing*）可以让我们重新使用这个变量名，目前你只要知道这个特性对于转换一个变量的类型很有用`let guess: u32 = guess.trim().parse().expect("Please type a number!");`，字符串的`parse`方法将字符串转换为其它类型。我们需要使用`:`在变量后指定类型来告诉Rust变量的类型是什么，此外`u32`的注解以及与`secret_number`比较的行为会让Rust将`secret_number`也推断为`u32`类型

### 使用循环来允许多次猜测

```rust
    // --snip--

    println!("The secret number is: {secret_number}");

    loop {
        println!("Please input your guess.");

        // --snip--

        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => println!("You win!"),
        }
    }
}
```

`loop`关键词创建了一个无限循环。用户现在无法正常退出游戏

### 当猜想正确时退出游戏

```rust
        // --snip--

        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}
```

### 处理无效输入

```rust
        // --snip--

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("You guessed: {guess}");

        // --snip--
```

我们使用`match`表达式将碰到错误退出程序改为处理错误。`_`是一个通配符，可以匹配所有值，本例中是匹配所有`Err`所包含的任何值。这样可以限制用户必须填写正确的数字
