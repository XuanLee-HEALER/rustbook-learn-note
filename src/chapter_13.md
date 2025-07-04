# 第十三章 - 函数式语言特性：迭代器和闭包

函数式编程的风格通常包括将函数作为值在参数中传递，从函数中返回函数，将它们赋值给变量供后续使用

## 闭包：捕获它们所处上下文的匿名函数

Rust的闭包是你可以保存到变量中或者作为参数传递给其它函数的匿名函数。你可以在一个地方创建闭包，然后在不同的上下文中调用。闭包可以在定义它们的域中捕获值

### 使用闭包来捕获环境

以这个场景为例：有一个T恤公司，准备向那些在邮件列表的用户赠送限定款T恤。邮件列表上的用户可以添加他们最喜欢的颜色，如果T恤有他们设定的颜色，就赠送这个颜色的T恤，否则赠送公司现在存量最多的颜色

```rust
#[derive(Debug, PartialEq, Copy, Clone)]
enum ShirtColor {
    Red,
    Blue,
}

struct Inventory {
    shirts: Vec<ShirtColor>,
}

impl Inventory {
    fn giveaway(&self, user_preference: Option<ShirtColor>) -> ShirtColor {
        user_preference.unwrap_or_else(|| self.most_stocked())
    }

    fn most_stocked(&self) -> ShirtColor {
        let mut num_red = 0;
        let mut num_blue = 0;

        for color in &self.shirts {
            match color {
                ShirtColor::Red => num_red += 1,
                ShirtColor::Blue => num_blue += 1,
            }
        }
        if num_red > num_blue {
            ShirtColor::Red
        } else {
            ShirtColor::Blue
        }
    }
}

fn main() {
    let store = Inventory {
        shirts: vec![ShirtColor::Blue, ShirtColor::Red, ShirtColor::Blue],
    };

    let user_pref1 = Some(ShirtColor::Red);
    let giveaway1 = store.giveaway(user_pref1);
    println!(
        "The user with preference {:?} gets {:?}",
        user_pref1, giveaway1
    );

    let user_pref2 = None;
    let giveaway2 = store.giveaway(user_pref2);
    println!(
        "The user with preference {:?} gets {:?}",
        user_pref2, giveaway2
    );
}
```

在`giveaway`方法中使用了闭包，我们在`Option<ShirtColor>`类型的值上调用了`unwrap_or_else`方法，它接受一个没有参数、返回值类型为`T`的闭包作为参数，`T`和`Option<T>`中`Some`包含的值类型一样，如果是`None`就返回闭包调用后的返回值。我们在函数参数中定义了这个闭包，如果结果需要时会被计算

例子中的关键点在于，我们传入一个调用`Inventory`实例上的`self.most_stocked()`方法的闭包。标准库不知道我们定义的`Inventory`或者`ShirtColor`类型，或者我们想执行的逻辑。闭包捕获了不可变引用`self/Inventory`，并且将它传入我们在`unwrap_or_else`方法中指定的代码。函数不能以这种方式捕获定义它们环境中的值

### 闭包类型推断和注解

函数和闭包有很多区别

- 闭包通常不需要你注解参数和返回值的类型，但是`fn`函数需要，因为它们作为函数接口的一部分会暴露给用户，而闭包不是作为对外暴露接口来使用的，它们被存储在变量中，不需要用函数名称来调用它们
- 闭包通常很短，并且关联更小的上下文范围。在这些受限的上下文中，编译器可以推断闭包参数和返回值的类型，就像推断变量的类型一样

如果我们想增加代码的清晰度和整洁度，我们可以使用比严格要求更完整的方式添加类型注解。对于闭包的定义，编译器会为每个参数和返回值推断一个具体类型，确定之后就无法修改

```rust
let expensive_closure = |num: u32| -> u32 {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    num
};
```

### 捕获引用或者移动所有权

闭包可以以三种方式从定义它们的环境中捕获值，这三种方式直接对应函数接受参数的三种方式：不可变借用、可变借用、获取所有权。闭包基于它处理捕获的值的方式来决定使用哪种方式

```rust
fn main() {
    let list = vec![1, 2, 3];
    println!("Before defining closure: {list:?}");

    let only_borrows = || println!("From closure: {list:?}");

    println!("Before calling closure: {list:?}");
    only_borrows();
    println!("After calling closure: {list:?}");
}
```

因为闭包捕获的是外部变量的不可变引用，所以在调用前后都可以访问这个vector。并且也可以看到用变量绑定的闭包可以通过`()`调用，好像变量名就是函数名

```rust
fn main() {
    let mut list = vec![1, 2, 3];
    println!("Before defining closure: {list:?}");

    let mut borrows_mutably = || list.push(7);

    borrows_mutably();
    println!("After calling closure: {list:?}");
}
```

在定义闭包和调用闭包中间不能使用`println!`宏打印vector，因为不能在创建可变引用且可变引用生效时去创建值的不可变引用

如果你要强制在闭包中获取外部变量的所有权，可以在闭包的参数列表前使用`move`关键字

```rust
use std::thread;

fn main() {
    let list = vec![1, 2, 3];
    println!("Before defining closure: {list:?}");

    thread::spawn(move || println!("From thread: {list:?}"))
        .join()
        .unwrap();
}
```

即便线程中只需要变量的不可变引用，因为主线程和新线程的结束顺序不确定，如果主线程先退出，那么`list`的引用就会失效，所以子线程必须获取`list`的所有权来保证引用总是有效

### 将捕获的值移出闭包和`Fn`特质

一旦闭包从定义它的环境中捕获了一个值的引用或者所有权（是否有值被移入了闭包），闭包中的代码就决定了在稍后计算的时候如何处理引用或者值（是否有内容被移出了闭包）。闭包存在下列行为

- 将捕获的值移出闭包
- 修改捕获的值
- 既不移动也不修改值
- 不从环境中捕获值

闭包捕获和处理环境中的值的方式会影响它们的类型所实现的特质，这些特质限制了函数和结构体能接受什么类型的闭包。闭包会**自动实现**`Fn`特质的一种、两种或者全部三种，取决于闭包是如何处理捕获的值

1. `FnOnce`表示那些**能被调用一次**的闭包。所有闭包至少实现了这个特质，因为所有闭包都能被调用。一个将捕获的值移出的闭包只会实现这个特质，因为它只能被调用一次
2. `FnMut`表示那些不会将捕获的值移出，但是有可能会修改捕获的值的闭包，这些闭包可以被调用超过一次
3. `Fn`表示那些不会将捕获的值移出，不修改捕获的值和不捕获值的闭包。这些闭包可以在不修改环境的情况下被调用多次，这个特性在需要并发多次调用的闭包中很重要

```rust
impl<T> Option<T> {
    pub fn unwrap_or_else<F>(self, f: F) -> T
    where
        F: FnOnce() -> T
    {
        match self {
            Some(x) => x,
            None => f(),
        }
    }
}
```

`F`的特质限制是`FnOnce() -> T`，表示`F`必须能被调用一次。这个限制表达了`unwrap_or_else`最多只会调用`f`一次，所以实现上面三种特质的任何闭包都可以被接收，非常灵活。如果我们不需要从环境中捕获值，我们可以使用函数名称而不是闭包。编译器自动实现对该函数应用恰当的`Fn`特质

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let mut list = [
        Rectangle { width: 10, height: 1 },
        Rectangle { width: 3, height: 5 },
        Rectangle { width: 7, height: 12 },
    ];

    list.sort_by_key(|r| r.width);
    println!("{list:#?}");
}
```

`sort_by_key`使用`FnMut`特质的闭包是因为它需要获取当前列表中的项的引用，然后参数是可以排序的类型`K`。因为切片中的每一项在排序时都要调用，所以闭包会调用多次。这个闭包没有捕获、修改、移出任何值，所以它满足限制要求

## 使用迭代器来处理一系列项

迭代器模式允许你按顺序在项的序列上执行一些任务。迭代器负责遍历每个项并执行对应的逻辑，并决定什么时候这个序列遍历完成。当你使用迭代器时，不需要自己重新实现这些逻辑。Rust中，迭代器是**懒的**（*lazy*），直到你调用消费迭代器的方法之前是没有性能消耗的。当我们创建一个迭代器后，有多种方式来使用它

```rust
let v1 = vec![1, 2, 3];

let v1_iter = v1.iter();

for val in v1_iter {
    println!("Got: {val}");
}
```

迭代器处理遍历序列的逻辑，减少重复代码。迭代器提供了更好的灵活性，可以对多种序列应用相同的逻辑，不只是可以索引的数据结构

### `Iterator`特质和`next`方法

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // methods with default implementations elided
}
```

`type Item`和`Self::Item`定义了这个特质的**关联类型**（*associated type*）。可以将其简单理解为要实现`Iterator`特质，你需要定义一个`Item`类型，这个类型是`next`方法中的返回值类型。`next`方法每次返回一个包裹在`Some`中的迭代器元素的值，当迭代器遍历完成后，返回`None`。我们可以直接调用`next`方法，这样做需要将迭代器变量声明为可变的，调用`next`方法会改变迭代器的内部状态，这个状态是迭代器用来跟踪序列中的遍历位置。每次调用`next`会从迭代器消费一个项。当我们使用`for`循环时不需要将它声明为可变的，因为循环会获取这个迭代器变量的所有权，然后在后台让它成为可变的

注意我们从`next`中得到的是迭代器中的值的不可变引用。`iter`方法会在不可变引用上创建迭代器。如果我们要获取迭代器中值的所有权，可以使用`into_iter`，如果我们需要可变引用，可以使用`iter_mut`

### 消费迭代器的方法

`Iterator`特质有一些标准库默认实现的方法。这些方法会调用`next`方法，所以你只需要手动实现这个方法。调用`next`的方法叫做**消费适配器**（*consuming adapters*），因为调用这些方法会消费迭代器。例如`sum`方法，它会获取迭代器的所有权，然后遍历它的项最后返回累加和

### 产生其它迭代器的方法

**迭代器适配器**（*Iterator adapters*）是定义在`Iterator`特质下的不消费迭代器的方法，它们通过改变原始迭代器的一部分来产生一个不同的迭代器。例如`map`方法，它接收一个闭包，迭代每个项会调用传入的闭包，使用返回值构建一个新迭代器，包含修改后的项。迭代器适配器也是懒的。因为`map`接收一个闭包，我们可以指定想在项上执行的任何操作。你可以使用链式调用迭代器适配器来以可读性更好的方式执行复杂操作。因为所有的迭代器都是懒的，你必须调用一个消费适配器来得到最终结果

### 使用捕获环境的闭包

很多迭代器适配器接收闭包为参数，通常这些闭包会捕获它们的环境

```rust
#[derive(PartialEq, Debug)]
struct Shoe {
    size: u32,
    style: String,
}

fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}
```

闭包从环境中捕获了`shoe_size`参数

## 改善我们的I/O项目

### 使用迭代器来移除`clone`

```rust
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

如果`Config::build`接收迭代器的所有权，我们可以从迭代器中将字符串移入`Config`，而不需要调用`clone`方法执行新的分配

### 直接使用返回的迭代器

`env::args`函数返回的是`std::env::Args`，这个类型实现了`Iterator`特质，遍历返回`String`值。我们要更新参数的类型为`impl Iterator<Item = String>`。因为我们会获取它的所有权，并且会遍历上面的值，所以可以声明为`mut`，使用`Iterator`特质方法而不是索引，使用`next`方法来获取值，使用`match`来处理不同的情况。使用迭代器适配器来让代码更简洁

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }

    results
}

pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents
        .lines()
        .filter(|line| line.contains(query))
        .collect()
}
```

## 比较性能：循环 vs 迭代器

迭代器尽管是更高层次的抽象，但是编译后的代码和手写的低级代码是一样的。迭代器是Rust的**零成本抽象**之一，表示我们在使用这个抽象时没有引入额外的运行时负载

> "What you don't use, you don't pay for" -- Bjarne Stroustrup
>
> -> What you do use, you couldn't hand code any better

```rust
let buffer: &mut [i32];
let coefficients: [i64; 12];
let qlp_shift: i16;

for i in 12..buffer.len() {
    let prediction = coefficients.iter()
                                 .zip(&buffer[i - 12..i])
                                 .map(|(&c, &s)| c * s as i64)
                                 .sum::<i64>() >> qlp_shift;
    let delta = buffer[i];
    buffer[i] = prediction as i32 + delta;
}
```

这里编译后的代码并没有遍历`coefficients`的值，因为Rust知道这里有12次循环，所以它会展开循环。*Unrolling*是移除额外的循环的优化。你可以随意使用迭代器和闭包，它们让代码看起来层次更高级、可读性更好，并且没有付出运行时性能代价
