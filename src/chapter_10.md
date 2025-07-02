# 第十章 - 泛型类型、特质和生命周期

每种编程语言都有高效处理**重复概念**的工具。在Rust中这个工具是**泛型**（*generics*）：对具体类型或其它属性的抽象替代。我们可以在编译期和运行期不知道值的具体类型的情况下表达泛型的行为或泛型之间的联系。泛型允许我们使用占位符来替代具体类型，表示多种类型以减少代码的重复

```rust
fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let mut largest = &number_list[0];

    for number in &number_list {
        if number > largest {
            largest = number;
        }
    }

    println!("The largest number is {largest}");
}
```

上面的代码在求一个数组中的最大值，如果我们需要对两个不同的数组求最大值，同样的逻辑就需要重复两遍。写重复代码很啰嗦并且容易出错，而且如果逻辑更新还需要每处都更新。要消除这种重复性，我们可以定义一个接收整数列表作为参数的函数来创建一个抽象

```rust
fn largest(list: &[i32]) -> &i32 {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {result}");

    let number_list = vec![102, 34, 6000, 89, 54, 2, 43, 8];

    let result = largest(&number_list);
    println!("The largest number is {result}");
}
```

转换代码的步骤如下

1. 找到重复的代码
2. 将重复代码放到函数中，指定输入和输出（参数和返回值）
3. 将之前的重复代码修改为函数调用

和函数体中处理的`list`一样，泛型允许代码在抽象类型上操作

## 泛型数据类型

### 在函数定义中

定义使用泛型的函数时，我们在函数签名中参数和返回值的数据类型的位置替换成泛型。这样做给调用者提供了更多的灵活性。要在一个新函数中**参数化**类型，我们需要为类型参数命名。你可以使用任何标识符来命名，根据习惯这里使用`T`，Rust中的类型参数名通常短小，使用一个字母，并且风格是**UpperCamelCase**。当我们在函数体中使用参数时，我们必须在函数签名中声明泛型参数名称，编译器才知道名字的含义。要定义泛型的`largest`函数，我们将类型名称定义放到函数名和参数列表中间的尖括号中

```rust
fn largest<T>(list: &[T]) -> &T {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

// -- code snip --
}
```

上面的代码无法通过编译，因为我们想比较两个`T`类型的值，就必须使用那些可以被排序的类型的值。你可以在类型上实现`std::cmp::PartialOrd`特质来为类型实现排序能力。我们要求`T`是那些实现了此特质的类型`fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {`，这个例子就能通过编译了，因为`i32`和`char`在标准库中都实现了此特质

### 在结构体定义中

我们也可以在定义结构体时，定义泛型类型的成员

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```

首先我们在结构体名称后的尖括号中声明了泛型类型的名称，然后我们在需要指定具体类型的地方使用了泛型类型。注意因为`Point<T>`结构体中的`x`和`y`都是T，所以它们的值的具体类型必须相同，如果使用两个不同类型的值构造`Point<T>`实例就会编译失败

如果希望`Point`结构体中`x`和`y`都是泛型，且它们值的具体类型是不同类型，我们可以使用多个泛型参数

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

fn main() {
    let both_integer = Point { x: 5, y: 10 };
    let both_float = Point { x: 1.0, y: 4.0 };
    let integer_and_float = Point { x: 5, y: 4.0 };
}
```

你可以声明任意多个泛型参数，但是这样会让代码变得难读，如果你发现你需要很多的泛型类型，这表明你的代码需要重构为更小的片段

### 在枚举定义中

我们可以定义包含泛型类型的变体的枚举类型，就像`Option<T>`

```rust
enum Option<T> {
    Some(T),
    None,
}
```

通过使用`Option<T>`枚举，我们可以表达出”可能不存在的值“的抽象概念，并且因为它使用泛型实现，我们可以在任何类型的Option上使用这个抽象

枚举也可以声明多个泛型参数，例如`Result`枚举

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

⚠️当你在代码中发现很多结构体和枚举的定义之间的区别仅在于它们持有的值的类型不同时，你可以通过使用泛型类型来减少这种重复性

### 在方法定义中

我们可以为结构体和枚举实现方法，并且在方法的定义中使用泛型类型

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };

    println!("p.x = {}", p.x());
}
```

我们必须在`impl`后重新声明`T`，才可以使用`T`来表示我们正在`Point<T>`上实现方法。Rust可以识别出尖括号中的类型是泛型类型而不是具体类型。我们可以使用和结构体定义中不同的泛型类型名，但是使用相同的名称更符合习惯。如果你在声明了泛型类型的`impl`块中写了一个方法，那么这个方法会定义在此类型的泛型类型实例上，无论最后的具体类型是什么

我们也可以在实现方法时限制泛型类型。例如为`Point<f32>`的实例实现方法而不是`Point<T>`，此时我们没有在`impl`后声明任何泛型类型

```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

结构体定义中的泛型类型参数名称不总是和你在结构体方法中使用的相同

```rust
struct Point<X1, Y1> {
    x: X1,
    y: Y1,
}

impl<X1, Y1> Point<X1, Y1> {
    fn mixup<X2, Y2>(self, other: Point<X2, Y2>) -> Point<X1, Y2> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10.4 };
    let p2 = Point { x: "Hello", y: 'c' };

    let p3 = p1.mixup(p2);

    println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

这个例子中`X1`和`Y1`在`impl`后声明表示它们是结构体定义的，而`X2`和`Y2`则仅跟方法有关联

### 泛型代码的性能

使用泛型类型不会让你的代码运行得比具体类型更慢。Rust通过在编译期对使用泛型的代码执行单态化来实现这一点。**单态化**（*Monomorphization*）是在编译时将值的具体类型替换掉泛型类型的过程。在这个过程中，编译器执行我们创建泛型函数的反过程：编译器查看所有泛型代码被调用的地方，然后生成对应的具体类型的代码，不会有额外的运行时消耗

```rust
let integer = Some(5);
let float = Some(5.0);
```

当Rust编译这段代码时会执行单态化过程。编译器检查出`Option<T>`实例中使用到的值的类型有两种，分别是`i32`和`f64`。然后它会将`Option<T>`的通用定义扩展成两种具体类型的定义，就像下面代码展示的（编译器可能使用其它名称）

```rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

## 特质：定义共享行为

一个**特质**（*trait*）定义了某个类型自己拥有且能和其它类型共享的功能。我们可以使用特质来以一种抽象的方式定义共享行为。我们可以使用**特质限制**（*trait bounds*）来指定一个泛型类型可以被替换成任何提供这些行为的类型

### 定义一个特质

类型的行为包括我们可以在该类型上调用的方法。特质的定义是一种将方法签名组合在一起的方式，为了实现某个目的而定义一组必需的行为

假设现在有两个结构体来表示两种类型的文本：`NewsArticle`表示新闻故事，`Tweet`表示微博。我们要写一个聚合器库叫做`aggregator`，展示存储在不同类型文本的总结信息，我们通过调用`summarize`方法获取它们的总结

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

方法签名结束的地方需要有一个分号。每个实现此特质的类型必须在方法体中提供自己的单独实现。编译器会要求任何实现`Summary`特质的类型必须实现`summarize`方法。一个特质可以包含多个方法，每个方法一行，以分号结尾

### 在一个类型上实现特质

```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

实现特质和实现普通方法类似，不同点在于`impl`后要写被实现的特质的名称，然后使用`for`，之后指定实现特质的类型名称。在`impl`块中我们将特质中定义的方法签名移入进来，使用花括号创建方法体，在其中实现该类型的行为逻辑

```rust
use aggregator::{Summary, Tweet};

fn main() {
    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    println!("1 new tweet: {}", tweet.summarize());
}
```

### 调用者必须将特质和类型一起引入域中

其它依赖`aggregator`的crate可以将`Summary`引入域中来为自己的类型实现。实现特质有一项限制，我们要为某个类型实现特质，类型和特质需要至少有一个是在当前crate声明的。这个限制是**一致性**（*coherence*）属性的一部分，也叫做**孤儿原则**（*orphan rule*），因为父类型（parent type，指实现特质的类型或者特质本身）不在当前crate。这条规则保证了其它人的代码不会破坏你的代码，反之亦然。如果没有这条规则，两个crate可能为同一个类型实现同一个特质，Rust无法确定应该使用哪个实现

### 默认实现

和我们为某个类型实现特质一样，我们可以保留或者重载每个方法的默认行为

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

为了使用特质方法的默认实现，我们仍然需要指定一个空的`impl`块`impl Summary for NewsArticle {}`。重载默认实现的语法和实现一个没有默认行为的特质方法是一样的。在默认实现中可以调用同一个特质中的其它方法，即使其它方法没有默认实现。依靠这种方式，特质可以提供很多有用的功能，并且只需要实现很少的一部分代码。可以将需要定制化实现的功能作为单独的方法，然后提供一个默认实现的方法，结合这些子方法来实现某些功能，这样会减少那些具体类型需要写的实现代码

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}

impl Summary for Tweet {
    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
}
```

要注意如果已经重载了方法，就没办法再调用默认的行为了

### 特质在参数中

我们要定义一个`notify`函数来调用`item`参数上的`summarize`方法，这个参数是那些实现了`Summary`特质的类型

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

### 特质限制语法

`impl Trait`语法作用方式很直接，但它实际上是**特质限制**（*trait bounds*）的语法糖，原始形式如下

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

我们将特质限制放到泛型类型声明之后，中间加上`:`。`impl Trait`语法很方便，在简单情况下可以让代码更精简，但是在其它情况下使用全限制语法可以表达更复杂的情况

```rust
pub fn notify(item1: &impl Summary, item2: &impl Summary) {
pub fn notify<T: Summary>(item1: &T, item2: &T) {
```

泛型类型`T`也限制了传入函数的两个参数的具体类型必须是一致的，所以在两个参数的具体类型可能不一致时可以使用`impl Trait`语法

#### `impl Trait`语法

使用`+`语法来同时指定多个特质限制

```rust
pub fn notify(item: &(impl Summary + Display)) {
pub fn notify<T: Summary + Display>(item: &T) {
```

#### 使用`where`子句让特质限制更清晰

每个泛型有自己的特质限制，所以如果函数有很多泛型类型参数，就会有很多特质限制。如果都写在函数名和参数列表中间，会让函数签名很难读，Rust提供了`where`子句来改写签名，使得函数名、参数列表和返回值更靠近

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {

fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
```

### 特质在返回值中

我们也可以在返回值类型中使用`impl Trait`语法来返回实现了某些特质的类型

```rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    }
}
```

在闭包和迭代器的用法中指定实现某些特质类型作为返回值很有用。闭包和迭代器创建的类型是只有编译器才知道的类型或者定义很长的类型。`impl Trait`语法可以更精简地指定一个函数返回实现了`Iterator`特质的类型，并不需要写那些很长的类型定义。只有返回同一种实现了某些特质的类型才能使用`impl Trait`语法，返回两种不同的实现了同种特质的类型由于语法在编译器的实现方式是不被允许的

### 使用特质限制有条件地实现方法

在使用了泛型类型参数的结构体的`impl`块中添加特质限制，可以为实现了这些特质的类型有条件地实现方法。可以为这些类型扩展能力

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

我们也可以有条件地为那些实现了其它特质的类型实现某个特质。在任何满足特质限制的类型上实现特质叫做**覆盖式实现**（*blanket implementations*），这种技术在标准库中广泛使用。例如标准库为所有实现了`Display`特质的类型上实现`ToString`特质。覆盖实现出现在特质文档的“implementors”章节

```rust
impl<T: Display> ToString for T {
    // --snip--
}
```

特质和特质限制使得我们可以使用泛型类型参数来减少重复性，也告诉编译器我们希望泛型类型拥有特殊的行为。编译器使用特质限制的信息来检查代码中使用的具体类型是否满足了特质限制。在动态类型语言中，如果我们在没有实现方法的类型上调用该方法会得到运行时错误。但是Rust将这些错误提前到了编译期

## 使用生命周期来验证引用

生命周期是用来保证引用在程序运行过程中总是有效的一种方式。Rust中所有的引用都有一个**生命周期**（*lifetime*），这是引用有效的域，大部分情况下生命周期是隐式的和自动推断的。当引用的生命周期以不同的方式使用时我们必须显式注解它。Rust需要我们使用泛型生命周期参数来注解这些关系，保证运行时中引用总是有效的

### 使用生命周期来防止悬垂引用

生命周期的主要目的是防止**悬垂引用**（*dangling references*），这会导致程序引用到不希望访问的数据

```rust
fn main() {
    let r;

    {
        let x = 5;
        r = &x;
    } // x 已经被清理，r是一个悬垂引用

    println!("r: {r}");
}
```

外部的域声明的`r`没有初始值，内部域中声明了x并且初始化为`5`。我们将`r`赋值为`x`的引用，然后内部域结束，我们尝试打印`r`的值，因为`r`引用的值在我们使用时已经离开了域，所以无法通过编译。`r`可能会引用`x`出域时被释放的内存

### 借用检查器

Rust编译器中有**借用检查器**（*borrow checker*），它的作用是检查所有的域来判断是否所有的借用都是有效的。上面的代码中，`r`的生命周期是`'a`，`x`的生命周期是`'b`。`'b`的作用域范围比`'a`小，Rust会比较两个生命周期的大小并且看到`r`有`'a`的生命周期但是引用了`'b`生命周期的内存。因为`'b`比`'a`短，所以程序无法通过编译，下面的代码中`x`有更长的生命周期，`r`的生命周期更短，所以下面的代码可以通过编译

```rust
fn main() {
    let x = 5;            // ----------+-- 'b
                          //           |
    let r = &x;           // --+-- 'a  |
                          //   |       |
    println!("r: {r}");   //   |       |
                          // --+       |
}                         // ----------+
```

### 函数中的泛型生命周期

我们需要写一个函数`longest`，返回两个字符串切片中更长的那个

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

上面的代码无法通过编译，报错信息显示返回类型需要一个泛型生命周期参数，因为Rust无法确定返回的的引用属于`x`还是`y`。事实上我们也不知道。借用检查器无法判断`x`和`y`的生命周期和返回值的生命周期是如何关联的

### 生命周期注解语法

生命周期注解并没有改变引用的存活时间。它们只是描述了**多个引用之间生命周期的关系**。函数可以接受通过泛型生命周期参数指定的任何生命周期的引用。生命周期注解的语法是以`'`开头，通常是小写字母并且很短，类似泛型类型。我们将生命周期参数注解放到`&`之后，然后使用空格来和引用的类型分隔开

```text
&i32        // a reference
&'a i32     // a reference with an explicit lifetime
&'a mut i32 // a mutable reference with an explicit lifetime
```

#### 函数签名中的生命周期注解

要在函数签名中使用生命周期参数，我们需要在函数名和参数列表之间的尖括号中声明**泛型生命周期**（*generic lifetime*）参数。我们想在签名中表达这些限制：返回的引用和两个引用参数的有效时间相同

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

函数签名告诉Rust对于生命周期`'a`，函数接收两个参数，它们的类型都是字符串切片，存活时间相同。返回的字符串切片存活时间**至少**会和`'a`一样长。实际上返回的引用的生命周期和函数参数引用的值的生命周期中更小的那个相同。注意`longest`函数不需要知道`x`和`y`具体存活多长时间，只要那些域可以替换成`'a`并满足这个签名的要求即可。函数的生命周期注解，只存在于函数签名中，而不在函数体中。生命周期注解变为函数约定的一部分，和参数类型一样。通过指定生命周期，编译器显示的错误信息可以更精确地指向我们出问题的代码。当我们将具体的引用传入函数后，它的生命周期就会替换`'a`，这个例子中是`x`和`y`的生命周期重合的部分，即两个域中更小的那个，因为我们为返回值也指定了`'a`，所以返回值的生命周期和这个更小的域一样

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {result}");
}
```

上面的代码无法通过编译，因为函数签名要求返回值的生命周期是两个引用参数中更小的那个，所以当`string2`离开域时，返回值的引用就已经失效了

### 从生命周期的角度考虑

如何指定生命周期的取决于你的函数在做什么

```rust
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

因为`y`和`x`以及返回值没有关系，所以不需要指定它的生命周期。如果返回值和参数没有关系，那么它一定引用了函数体中的内容，这样指定生命周期也无济于事，因为当函数返回后，返回值就变成了悬垂引用，这种情况最好返回自有数据类型。生命周期语法是**关联函数的参数和返回值的生命周期关系**的。一旦它们建立了关系，Rust就有足够多的信息来检查影响内存安全的操作，不允许创建悬垂指针或者其他破坏内存安全的操作

### 结构体定义中的生命周期注解

我们可以定义成员类型为引用的结构体，但是我们需要在结构体定义中加上每个引用的生命周期注解

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().unwrap();
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

在结构体名称后的尖括号中我们声明了泛型生命周期参数，所以在结构体定义中我们可以使用这个生命周期参数，表示`ImportantExcerpt`实例存活时间不能超过part成员的有效时间

### 生命周期省略

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

在Rust的早期版本（1.0之前），上面的代码无法通过编译，因为每个引用都需要显式标注生命周期，类似于`fn first_word<'a>(s: &'a str) -> &'a str {`。之后Rust团队发现Rust程序员在一些特定场景下会重复写相同的生命周期，这些场景都是可预测的并遵循一些确定的模式。开发者们将这些模式的处理写入编译器，借用检查器可以在这些场景下推断出生命周期。随着这种模式越来越多，以后需要显式标注生命周期的情况也会越来越少

这些模式叫做**生命周期省略规则**（*lifetime elision rules*），这些是编译器考虑的特定场景，如果你的代码符合这些场景，那么就不需要标注生命周期。函数或方法参数上的生命周期叫做**输入生命周期**（*input lifetimes*），返回值上的生命周期叫做**输出生命周期**（*output lifetimes*）。当它们没有显式标注时，编译器使用三条规则来判断引用的生命周期。第一条应用于输入生命周期，后两条应用于输出生命周期。如果三条规则验证后还发现存在无法确定生命周期的引用就会报错。这些规则应用于`fn`定义和`impl`块上

1. 编译器会给每个引用**参数**分配一个名字不同的生命周期
2. 如果只有一个输入生命周期参数，这个生命周期参数会分配给所有引用类型返回值
3. 如果有多个输入生命周期参数，但是其中包括`&self`或者`&mut self`（因为是方法），那么`self`的生命周期会分配给所有引用类型返回值

### 方法定义中的生命周期注解

结构体成员中的生命周期名称需要在`impl`关键字后声明，因为这些生命周期参数也是结构体类型的一部分。在方法签名中出现的引用参数可能和结构体成员的引用生命周期绑定，也可能是独立的

```rust
impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
}
```

在`self`上不需要加生命周期的注解，这是因为第一条规则，编译器帮你自动添加了生命周期参数

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {announcement}");
        self.part
    }
}
```

因为第一个参数是`&self`，所以返回值应用了`self`的生命周期，这是因为第一条和第三条规则，这样输入和输出的生命周期都已知

### 静态生命周期

`'static`表示引用可以在程序运行期间保持有效，所有的字符串字面值都是`'static`生命周期。在为一个引用指定`'static`生命周期时，要考虑这个引用是否确实会在程序运行期间保持有效。大部分时候编译器提示你使用`'static`生命周期的，都是因为你在尝试创建一个悬垂引用或者指定的生命周期不合理，这些情况应该尝试去解决真正的问题而不是直接指定`'static`生命周期

### 结合泛型类型参数，特质限制和生命周期

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("Announcement! {ann}");
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
