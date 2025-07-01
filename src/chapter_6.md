# 第六章 - 枚举和模式匹配

**枚举**（*enumerations*），也叫做*enums*。枚举允许你定义一个类型，枚举出它所有可能的**变体**（*variants*）

## 定义一个枚举

枚举是一种表明某个此类型的值一定属于一组可能值的方式

现在考虑一个例子，我们需要处理IP地址。目前IP地址的两种主要标准：v6和v4。目前我们程序只会碰到这两种地址。我们可以**枚举**（*enumerate*）所有可能的变体。每个IP地址只可能属于v4或者v6，但**不能同时属于两种**。IP地址的特性非常适合枚举结构，因为每个枚举类型的值只能是其中的一个变体

```rust
enum IpAddrKind {
    V4,
    V6,
}
```

### 枚举值

我们可以按照以下代码的方式创建两个变体的实例

```rust
let four = IpAddrKind::V4;
let six = IpAddrKind::V6;
```

枚举的变体在它的标识符（枚举名称）的命名空间下，使用`::`来划分。目前我们没有办法保存IP地址的数据，只知道类型，我们可以使用结构体来解决这个问题

```rust
enum IpAddrKind {
    V4,
    V6,
}

struct IpAddr {
    kind: IpAddrKind,
    address: String,
}

let home = IpAddr {
    kind: IpAddrKind::V4,
    address: String::from("127.0.0.1"),
};

let loopback = IpAddr {
    kind: IpAddrKind::V6,
    address: String::from("::1"),
};
```

利用这种方式，我们将枚举类型关联上了其它值。然而只使用一个枚举值来表示相同的概念更简洁，不需要将枚举类型放到结构体中，我们可以直接把数据放到每个枚举变体中

```rust
enum IpAddr {
    V4(String),
    V6(String),
}

let home = IpAddr::V4(String::from("127.0.0.1"));
let loopback = IpAddr::V6(String::from("::1"));
```

我们直接将数据放入了枚举的每个变体中，不再需要定义额外的结构体。每个变体的名字也变成了一个函数，用来构造一个变体的实例。我们通过定义枚举自动获得了这个构造器函数。使用枚举的另一个优势是每个变体都可以有不同的具体类型和关联数据，v4的IP地址总是有4个数字部分（0～255），如果我们想用4个`u8`存储v4地址，而用字符串存储v6地址，结构体是做不到的，而枚举可以

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

let home = IpAddr::V4(127, 0, 0, 1);
let loopback = IpAddr::V6(String::from("::1"));
```

标准库中的`IpAddr`实现也使用了枚举，但是它是把v4地址和v6地址作为结构体放到了变体中

```rust
struct Ipv4Addr {
    // --snip--
}

struct Ipv6Addr {
    // --snip--
}

enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr),
}
```

你可以将任何类型的数据放到枚举变体中。也可以包含其它的枚举，标准库中的类型定义通常不比你能想到的更复杂。如果我们不使用枚举而使用不同的结构体，每个结构体都属于不同的类型，我们不能简单定义一个接收任意类型的函数，但是枚举可以。我们也可以在枚举上定义方法

### `Option`枚举和它相对于Null值的优势

`Option`类型是非常常见的场景的一种编码实现：一个值可能是一些东西或者它什么都不是。例如当你从非空列表中请求第一个项，你会得到一个值，而如果从空列表中请求，你什么都无法得到。在类型系统中表示这个概念就是编译器可以检查你是否处理了所有你应该处理的情况

Rust没有其它语言的`null`特性。Null是表示没有值在那里的值。在有`null`的语言中，变量总是有两种状态：`null`或者非`null`。null值的问题是，当你使用`null`值作为非`null`值使用，你会得到错误。因为`null`或非`null`的属性非常广泛，所以非常容易出错。然而`null`尝试表达的概念仍然是有用的：`null`表示一个现在无效的值或者因为一些原因缺失的值

Rust用一个枚举来表示一个值存在不存在的概念，这就是`Option<T>`

```rust
enum Option<T> {
    None,
    Some(T),
}
```

因为这个枚举很重要，所以它在预包含的内容中，不需要显式将它引入域，它的变体也在预包含中，所以可以不加`Option::`直接使用两个变体，`<T>`使得`Some`变体可以持有任何类型的数据，每个不同的具体类型会让整个`Option<T>`变为不同的类型。对于`Some`变体的值，Rust可以推断类型，但是对于`None`变体的值，需要显式注解整个`Option`的类型才行

对一个`Some`值，我们知道具体的值在`Some`中，对一个`None`值，它表示类似`null`的含义。为什么`Option<T>`比`null`更好？简单来说，`Option<T>`和`T`是两种类型，编译器不会让我们像使用有效值`T`一样使用`Option<T>`。为了拥有一个可能为`null`的值，你必须将这个值的类型变为`Option<T>`，然后当你使用这个值时，你需要显式处理这个值为`null`的情况。任何地方都有值的类型一定不会是`Option<T>`，你可以安全地认为这个值不会为`null`

## `match`控制流语句

`match`控制流结构可以让你用值和一些模式进行匹配，然后基于匹配到的模式来执行对应代码。模式可以由字面值、变量名、通配符和很多其他东西组成。`match`的威力来自于模式的**表达性**（*expressiveness*）和编译器可以保证所有可能的case都会被处理。待匹配的值尝试匹配`match`的每一个模式，在第一个匹配成功的地方，会执行对应的代码块。这里用硬币举一个例子

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

首先`match`关键字后面是一个表达式，和`if`类似，但是和`if`不同的是，这个表达式的类型可以是任意的（`if`后面必须是布尔值）。然后是分支，每个分支有两部分：一个模式和一些代码。`=>`用来区分模式和运行的代码，每个分支之间使用`,`分割。当`match`表达式执行时，它**按顺序**用表达式的值和每个分支的模式比较，如果模式匹配了值，该模式的关联代码会执行，如果模式没有匹配值，会继续下一个分支。每个**分支的关联代码是一个表达式**。分支中表达式的计算值会作为整个`match`表达式的值返回。如果分支代码很短，我们通常不使用花括号，如果你需要运行多条语句，那么必须使用花括号，但是花括号后的逗号不是必须的

### 在模式中绑定值

分支另一个有用的特性是它们可以在匹配模式中绑定值的部分，这里的例子是从1999到2008年，美国铸造了正面印有50个州的25美分硬币，且只有25美分硬币有此设计。我们将这个信息添加到上面声明的枚举中

```rust
#[derive(Debug)] // so we can inspect the state in a minute
enum UsState {
    Alabama,
    Alaska,
    // --snip--
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),
}
```

想象一下有朋友正在收集50个州的25美分硬币，当我们通过硬币类型来整理零钱的时候，我们也会将每个25美分对应的州显示出来，如果我们的朋友没有，它们可以拿走收藏起来

```rust
fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {state:?}!");
            25
        }
    }
}
```

当我们调用`value_in_cents(Coin::Quarter(UsState::Alaska))`时，`coin`就是`Coin::Quarter(UsState::Alaska)`。当匹配到对应的分支，`state`就会绑定`UsState::Alaska`的值，我们就能在代码块中使用这个绑定的值

### 使用`Option<T>`匹配

当我们想获得`Options<T>`的`Some`变体内部的值，也可以使用`match`来处理

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}

let five = Some(5);
let six = plus_one(five);
let none = plus_one(None);
```

组合使用`match`和枚举在多数情况下很有用。Rust代码中这种模式经常出现：`match`一个枚举值，绑定变体内部的值到某个变量上，然后基于此执行代码

### 匹配是有穷尽的

`match`的分支模式必须覆盖所有的可能性，下面是一个无法通过编译的例子

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        Some(i) => Some(i + 1),
    }
}
```

Rust知道我们没有覆盖所有可能的情况。甚至知道我们忘记了哪个模式。Rust中的匹配是**有限的**（*exhaustive*）

### 匹配所有的模式和`_`占位符

使用枚举，我们可以对特定值执行特定动作，但是对所有剩余其它值执行一个默认动作。假如我们在实现一个游戏，如果roll出3，玩家不动，得到一个帽子，如果roll出7，玩家失去帽子，roll出其它值玩家在棋盘上移动

```rust
let dice_roll = 9;
match dice_roll {
    3 => add_fancy_hat(),
    7 => remove_fancy_hat(),
    other => move_player(other),
}

fn add_fancy_hat() {}
fn remove_fancy_hat() {}
fn move_player(num_spaces: u8) {}
```

对于最后一个分支，我们使用`other`覆盖所有其它可能的值。这种catch-all模式满足了`match`分支必须有限的要求。要注意我们必须**将catch-all分支放到最后**，因为模式是按照顺序匹配的。当我们想有一个catch-all分支，但是不需要使用模式匹配到的值，我们可以使用`_`，这是一个特殊的模式，告诉Rust我们不使用值，所以Rust也不会告警

### `if let`和`let else`的简洁控制流

在你需要处理值只匹配一个模式而忽略其它的情况下可以使用`if let`语法，下面是使用`match`的写法，我们只需要在值为Some时处理，而None的情况什么都不做，必须加上`_ => ()`这种模板代码

```rust
let config_max = Some(3u8);
match config_max {
    Some(max) => println!("The maximum is configured to be {max}"),
    _ => (),
}
```

`if let`语法接受一个模式和一个表达式，并且用`=`分隔。它和`match`的工作方式一样，表达式会传递给它匹配模式。在`if let`块中可以使用max变量，并且关联块中的代码只有在值匹配模式的时候执行。使用`if let`让你失去了`match`提供的检查其它可能性的能力。`match`和`if let`的选择取决于简洁性和检查完整性的权衡

```rust
let config_max = Some(3u8);
if let Some(max) = config_max {
    println!("The maximum is configured to be {max}");
}
```

`if let`可以和`else`结合使用，含义就类似match表达式中`_`分支

### 使用`let else`让代码始终顺利运行

一种常见的模式是当某些值出现执行一些计算，否则返回一个默认值。看下面的例子

```rust
impl UsState {
    fn existed_in(&self, year: u16) -> bool {
        match self {
            UsState::Alabama => year >= 1819,
            UsState::Alaska => year >= 1959,
            // -- snip --
        }
    }
}
```

我们可能会这样实现

```rust
fn describe_state_quarter(coin: Coin) -> Option<String> {
    if let Coin::Quarter(state) = coin {
        if state.existed_in(1900) {
            Some(format!("{state:?} is pretty old, for America!"))
        } else {
            Some(format!("{state:?} is relatively new."))
        }
    } else {
        None
    }
}
```

这样做实现了目标。但是它把所有的逻辑都放到了`if let`语句中实现，如果工作更复杂，就很难持续跟踪最上层关联的分支。我们可以利用`if let`表达式提前返回这个值

```rust
fn describe_state_quarter(coin: Coin) -> Option<String> {
    let state = if let Coin::Quarter(state) = coin {
        state
    } else {
        return None;
    };

    if state.existed_in(1900) {
        Some(format!("{state:?} is pretty old, for America!"))
    } else {
        Some(format!("{state:?} is relatively new."))
    }
}
```

但是这仍然不理想，`if let`的一个分支产生了一个值，另一个分支直接从函数返回了。要让这个常见的模式更好地表达，Rust有`let-else`语法，这个语法左边是一个模式，右边是表达式，和`if let`类似，但是它没有`if`分支，只有`else`分支，如果分支匹配，它会将值绑定到**外面域的模式**中，如果模式没有匹配，程序会进入`else`分支

```rust
fn describe_state_quarter(coin: Coin) -> Option<String> {
    let Coin::Quarter(state) = coin else {
        return None;
    };

    if state.existed_in(1900) {
        Some(format!("{state:?} is pretty old, for America!"))
    } else {
        Some(format!("{state:?} is relatively new."))
    }
}
```

使用`let-else`语法可以让函数的主流程一直顺利往下走，不会出现两个的控制流（像使用`if`那样）
