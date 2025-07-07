# 第十九章 - 模式和匹配

**模式**（*pattern*）是Rust的特殊语法，用来执行类型匹配的结构，有复杂的和简单的。结合`match`表达式和其它结构可以让你更好地控制程序的执行流程。模式包含以下内容的组合

- **字面值**（*Literals*）
- 解构数组、枚举、结构体或者元组
- 变量
- **通配符**（*Wildcards*）
- **占位符**（*Placeholders*）

要应用模式语法，我们需要用值和模式进行匹配，如果匹配成功，我们就能在代码中使用值。如果值符合模式的样式，我们可以使用模式中命名的部分，如果不符合，这部分的模式关联的代码就不会执行

## 所有可以使用模式的地方

### `match`分支

`match`表达式的格式是：关键字`match`，后面是待匹配的值，花括号中包含一个或多个分支匹配模式，每个模式有一个匹配成功后执行的表达式

```text
match VALUE {
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
}
```

`match`表达式需要穷尽值的所有可能模式。一种方式是使用*catchall*模式作为最后一个分支，匹配所有的剩余情况。还有一种方式是使用特殊模式`_`匹配所有值，但是值不能绑定到变量上，所以它通常作为最后一个分支。当你想忽略所有不需要的值时，可以使用`_`

### 条件`if let`表达式

`if let`等同于只匹配一种情况的`match`表达式。`if let`也可以有对应的`else`表达式来执行未匹配成功的代码。`if let`也可以和`else if`、`else if let`表达式结合使用，给了我们比`match`更高的灵活度，我们可以用它来表示只有一个值在匹配模式。并且Rust不需要这些条件分支互相有联系

```rust
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        println!("Using your favorite color, {color}, as the background");
    } else if is_tuesday {
        println!("Tuesday is green day!");
    } else if let Ok(age) = age {
        if age > 30 {
            println!("Using purple as the background color");
        } else {
            println!("Using orange as the background color");
        }
    } else {
        println!("Using blue as the background color");
    }
}
```

可以看到，`if let`也可以和match分支一样，引入新变量来覆盖已有的变量，但是我们只能在块中使用这个变量，而不能写`if let Ok(age) = age && age > 30`，新的`age`在新域开始前并不生效。`if let`的缺点是编译器不能检查值所有的模式可能性。如果我们忘记使用`else`块处理一些情况，编译器无法提示我们这个逻辑bug

### `while let`条件循环

`while let`条件循环中，只要值可以一直匹配模式，循环就会一直运行

### `for`循环

`for`循环中，`for`关键字后面就是指定的模式，例如在`for x in y`中，`x`就是模式

### `let`语句

每次你使用`let`语句的时候，你就是在使用模式，`let`语句的格式类似这样

```text
let PATTERN = EXPRESSION;
```

变量名就是一种简单的模式。Rust会比较表达式的值和模式，然后分配给它一个**能找到的名字**。在`let x = 5;`中，`x`是一个模式，表示“将匹配模式的任何内容绑定到变量`x`上”。因为名称`x`就是整个模式，这个模式实际上表示“绑定任何东西到变量`x`上，无论值是什么”

`let (x, y, z) = (1, 2, 3);`我们匹配的是元组模式，你可以将这种元组模式看作是嵌套了三个独立的变量模式。如果模式中的元素数量少于元组中的元素数量，那么整个类型就不会匹配，我们会得到编译错误。要解决这个错误，我们可以使用`_`或`..`来忽略元组中的一个或多个值。如果模式中有过多的变量导致无法匹配，我们要删除一些变量来保证和元组中的元素进行匹配

### 函数参数

函数参数也是模式

```rust
fn foo(x: i32) {
    // code goes here
}
```

`x`是一个模式，就像在`let`中一样，我们可以在函数参数使用元组模式来匹配传入的值

```rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({x}, {y})");
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```

我们也可以在闭包参数中使用模式，因为闭包和函数类似

## 拒绝性：一个模式是否可能会匹配失败

模式有两种形式：**可拒绝的**和**不可拒绝的**。匹配传入值的所有可能值的模式是**不可拒绝的**（*irrefutable*），比如`let x = 5;`中的`x`，因为它匹配任何值，所以不可能匹配失败。值可能匹配失败的模式是**可拒绝的**（*refutable*）。比如`if let Some(x) = a_value`中的`Some(x)`，如果`a_value`的值是`None`而不是`Some`，那么`Some(x)`模式就不会匹配

函数参数、`let`语句和`for`循环只能接受不可拒绝的模式，因为当值不匹配这些模式时，程序做不了什么有意义的事情。`if let`、`while let`和`let...else`语句可以接受可拒绝的和不可拒绝的模式，但是编译器对于使用不可拒绝的模式会警告，因为它们是用来处理可能的匹配失败的情况：条件的功能是根据成功或失败执行不同的代码

通常你不需要考虑可拒绝的和不可拒绝的模式的区别，但是你需要熟悉拒绝性的概念，在错误消息中看到后可以正确处理。`let Some(x) = some_option_value;`如果some_option_value是一个None值，匹配Some(x)就会失败，所以这个模式是一个可拒绝的。然而let语句只能接受一个不可拒绝的模式。在编译期Rust会报告这个错误

```bash
$ cargo run
   Compiling patterns v0.1.0 (file:///projects/patterns)
error[E0005]: refutable pattern in local binding
 --> src/main.rs:3:9
  |
3 |     let Some(x) = some_option_value;
  |         ^^^^^^^ pattern `None` not covered
  |
  = note: `let` bindings require an "irrefutable pattern", like a `struct` or an `enum` with only one variant
  = note: for more information, visit <https://doc.rust-lang.org/book/ch19-02-refutability.html>
  = note: the matched value is of type `Option<i32>`
help: you might want to use `let else` to handle the variant that isn't matched
  |
3 |     let Some(x) = some_option_value else { todo!() };
  |                                     ++++++++++++++++

For more information about this error, try `rustc --explain E0005`.
error: could not compile `patterns` (bin "patterns") due to 1 previous error
```

因为我们没有覆盖（也不可能覆盖）`Some(x)`模式的所有有效值，Rust会生成一个编译错误。如果在需要不可拒绝的模式的地方使用了一个可拒绝的模式，我们可以修改代码来使用这个模式：使用`if let`。如果我们给`if let`一个不可拒绝的模式，编译器会给出警告

```rust
let x = 5 else {
    return;
};
```

```bash
$ cargo run
   Compiling patterns v0.1.0 (file:///projects/patterns)
warning: irrefutable `if let` pattern
 --> src/main.rs:2:8
  |
2 |     if let x = 5 {
  |        ^^^^^^^^^
  |
  = note: this pattern will always match, so the `if let` is useless
  = help: consider replacing the `if let` with a `let`
  = note: `#[warn(irrefutable_let_patterns)]` on by default

warning: `patterns` (bin "patterns") generated 1 warning
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.39s
     Running `target/debug/patterns`
5
```

因为这个原因，`match`分支必须使用可拒绝的模式，除了最后一个分支，它应该使用一个不可拒绝的模式来匹配所有值。Rust允许我们在只有一个分支的match中使用不可拒绝的模式，但是这种语法没什么用，因为可以直接用`let`替代

## 模式语法

### 匹配字面值

```rust
let x = 1;

match x {
    1 => println!("one"),
    2 => println!("two"),
    3 => println!("three"),
    _ => println!("anything"),
}
```

这个语法适用于你希望你的代码在得到某一个具体值时执行动作的情况

### 匹配命名变量

命名变量是一个不可拒绝的模式，它会匹配任何值。在`match`、`if let`或`while let`表达式中使用命名变量时比较复杂。因为这些表达式会开启一个新的域，模式中声明的变量会覆盖外面的同名变量

```rust
let x = Some(5);
let y = 10;

match x {
    Some(50) => println!("Got 50"),
    Some(y) => println!("Matched, y = {y}"),
    _ => println!("Default case, x = {x:?}"),
}

println!("at the end: x = {x:?}, y = {y}");
```

第二个分支中引入一个新的变量`y`，会匹配任何`Some`值。因为我们是在`match`表达式内部的新域中，所以这是一个新的`y`变量。这个`y`会匹配`Some`中的任何值，也就是外部`x`中的值。这个值现在是`5`，所以这个分支的表达式会执行，打印`Matched, y = 5`。如果`x`是`None`，那么前两个分支都不会匹配，会进入最后一个分支，因为这个下划线分支没有新变量，所以表达式中的`x`还是外部的`x`。当`match`表达式结束后，它的域也结束了，所以内部的`y`的域也结束了。最后会打印`at the end: x = Some(5), y = 10`。要创建一个可以比较外部`x`和`y`值的`match`表达式，而不是引入一个新变量来覆盖已有的`y`变量，我们需要使用**匹配守卫条件**（*match guard conditional*）

### 多个模式

可以使用`|`语法来同时匹配多个模式，这是一个**匹配或**（*or*）操作符

```rust
let x = 1;

match x {
    1 | 2 => println!("one or two"),
    3 => println!("three"),
    _ => println!("anything"),
}
```

### 使用`..=`匹配范围值

`..=`语法允许我们匹配值是否在某个范围内部

```rust
let x = 5;

match x {
    1..=5 => println!("one through five"),
    _ => println!("something else"),
}
```

指定一个范围比使用`|`更简短，特别是在匹配`1～1000`的任意数字这种场景中。编译器在编译期会检查范围不能为空，并且因为Rust唯一能判断范围是否为空的类型是`char`和数字值，所以范围模式只能作用于数字或`char`类型的值

```rust
let x = 'c';

match x {
    'a'..='j' => println!("early ASCII letter"),
    'k'..='z' => println!("late ASCII letter"),
    _ => println!("something else"),
}
```

### 解构来分离值

我们可以使用模式来解构结构体、枚举和元组，分别使用这些类型的值的不同部分

#### 解构结构体

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x: a, y: b } = p;
    assert_eq!(0, a);
    assert_eq!(7, b);
}
```

代码创建了变量`a`和`b`，匹配了`p`的`x`和`y`成员。虽然例子中使用了新的变量名，更常见的方式是使用和成员名同名的变量，这样更容易记住新变量来自于哪些成员。因为这很常用，并且需要重复写名字，所以Rust提供了缩写语法，你只需要列出结构体成员名称，通过模式创建的变量就使用这些名字

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x, y } = p;
    assert_eq!(0, x);
    assert_eq!(7, y);
}
```

我们也可以通过字面值作为结构体模式的一部分而不为所有成员创建变量来解构。这样做可以让我们在创建变量来解构其它成员时测试一些成员的特殊值

```rust
fn main() {
    let p = Point { x: 0, y: 7 };

    match p {
        Point { x, y: 0 } => println!("On the x axis at {x}"),
        Point { x: 0, y } => println!("On the y axis at {y}"),
        Point { x, y } => {
            println!("On neither axis: ({x}, {y})");
        }
    }
}
```

要记住`match`表达式如果匹配到某个模式的分支后就不会检查其它分支了，所以例子中，如果`p`是`Point { x: 0, y: 0}`，只会打印`On the x axis at 0`

#### 解构枚举

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let msg = Message::ChangeColor(0, 160, 255);

    match msg {
        Message::Quit => {
            println!("The Quit variant has no data to destructure.");
        }
        Message::Move { x, y } => {
            println!("Move in the x direction {x} and in the y direction {y}");
        }
        Message::Write(text) => {
            println!("Text message: {text}");
        }
        Message::ChangeColor(r, g, b) => {
            println!("Change color to red {r}, green {g}, and blue {b}");
        }
    }
}
```

对于不包含数据的枚举变体，例如`Message::Quit`，我们不能解构出它的值，我们只能匹配它的字面值。对于类结构体的枚举变体，例如`Message::Move`，我们可以使用匹配结构体的模式，在变体名后使用花括号，列出成员的变量。对于类元组的变体，例如`Message::Write`包含一个元素，`Message::ChangeColor`包含三个元素，可以使用匹配元组的模式

#### 解构嵌套结构体和枚举

模式匹配也可以应用于嵌套的项

```rust
enum Color {
    Rgb(i32, i32, i32),
    Hsv(i32, i32, i32),
}

enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(Color),
}

fn main() {
    let msg = Message::ChangeColor(Color::Hsv(0, 160, 255));

    match msg {
        Message::ChangeColor(Color::Rgb(r, g, b)) => {
            println!("Change color to red {r}, green {g}, and blue {b}");
        }
        Message::ChangeColor(Color::Hsv(h, s, v)) => {
            println!("Change color to hue {h}, saturation {s}, value {v}");
        }
        _ => (),
    }
}
```

#### 解构结构体和元组

我们可以以更复杂的方式混合、匹配和嵌套结构模式`let ((feet, inches), Point { x, y }) = ((3, 10), Point { x: 3, y: -10 });`，这行代码让我们将复杂类型划分为单独的组件，我们可以以感兴趣的方式使用这些值

### 在模式中忽略值

#### 使用`_`忽略所有值

我们已经使用下划线作为通配符模式来匹配所有值但是不绑定值。通常在`match`表达式里作为最后一个分支使用。其实我们可以在任何模式中使用它，包括函数参数

```rust
fn foo(_: i32, y: i32) {
    println!("This code only uses the y parameter: {y}");
}

fn main() {
    foo(3, 4);
}
```

大多数情况下，当你不再需要某个函数参数时，你会改变函数签名让它不再包括不需要的参数。但是在另一种情况下，比如你在实现一个特质，你需要它的类型签名但是实现代码中并不需要某个参数，可以通过使用下划线来避免使用参数名称产生的编译器警告

### 使用嵌套的`_`来忽略部分值

我们可以在其它模式中使用`_`来忽略值的一部分，例如，当我们只想测试值的一部分

```rust
let mut setting_value = Some(5);
let new_setting_value = Some(10);

match (setting_value, new_setting_value) {
    (Some(_), Some(_)) => {
        println!("Can't overwrite an existing customized value");
    }
    _ => {
        setting_value = new_setting_value;
    }
}

println!("setting is {setting_value:?}");
```

在第一个匹配分支中，我们不需要匹配`Some`中的值，但是我们需要测试`setting_value`和`new_setting_value`都是`Some`时的情况。我们也可以在一个模式中的多个位置使用下划线来忽略特定的值

```rust
let numbers = (2, 4, 8, 16, 32);

match numbers {
    (first, _, third, _, fifth) => {
        println!("Some numbers: {first}, {third}, {fifth}");
    }
}
```

### 通过在名称前加`_`来标记不使用的变量

如果你创建了一个变量但是没有使用它。Rust会警告，因为未使用变量可能会导致bug。但是有时创建还不使用的变量很有用，例如在项目的原型阶段。这种情况下，你可以告诉Rust不要对此发出警告，通过在未使用变量的名称前加上下划线

```rust
fn main() {
    let _x = 5;
    let y = 10;
}
```

这段代码中，编译器只会对`y`发出警告。注意仅使用`_`和名称前使用`_`的细微区别。`_x`仍然会绑定变量的值，而`_`完全不会绑定值

### 使用`..`来忽略值的剩余部分

对于有很多部分的值，我们可以使用`..`来仅匹配其中一部分，忽略剩余部分而不用列出所有要被忽略的值。`..`模式忽略值中所有我们模式中没有显式匹配的部分

```rust
struct Point {
    x: i32,
    y: i32,
    z: i32,
}

let origin = Point { x: 0, y: 0, z: 0 };

match origin {
    Point { x, .. } => println!("x is {x}"),
}
```

`..`是一个更快捷的方式，替换`y: *`和`z: *`的写法，特别是当我们处理有只需要其中的一个或两个成员的包含很多成员的结构体的情况。`..`会尽可能多地扩展它需要表示的值

```rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (first, .., last) => {
            println!("Some numbers: {first}, {last}");
        }
    }
}
```

第一个和最后一个匹配`first`和`last`，`..`会匹配并忽略中间的任何值。使用`..`的含义必须是清晰的。如果对于要匹配的值和要忽略的值无法确定，Rust会报错

```rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (.., second, ..) => {
            println!("Some numbers: {second}")
        },
    }
}
```

### 带匹配守卫的额外条件

一个`匹配守卫`（*match guard*）是相当于额外的`if`条件，在`match`分支的模式后指定，在这个分支被匹配后还需要判断条件。匹配守卫适合表达比`match`表达式更复杂的逻辑。它只能在`match`表达式中使用，不能在`if let`或`while let`表达式中使用。条件中可以使用模式中声明的变量

```rust
let num = Some(4);

match num {
    Some(x) if x % 2 == 0 => println!("The number {x} is even"),
    Some(x) => println!("The number {x} is odd"),
    None => (),
}
```

因为没有办法在一个模式中表达`if x % 2== 0`条件，所以可以使用匹配守卫表示这个逻辑。这种额外的表达性的缺点是编译器无法检查包含匹配守卫表达式的`match`是否穷尽了所有可能

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(n) if n == y => println!("Matched, n = {n}"),
        _ => println!("Default case, x = {x:?}"),
    }

    println!("at the end: x = {x:?}, y = {y}");
}
```

在第二个匹配分支没有引入新变量`y`，所以我们在匹配守卫中使用的是外部的`y`，我们指定了`Some(n)`，这会创建一个新变量`n`，因为外部没有`n`所以不会覆盖任何值。你也可以在`|`指定多个匹配模式中使用匹配守卫，这个条件会应用到所有匹配的模式上

```rust
let x = 4;
let y = false;

match x {
    4 | 5 | 6 if y => println!("yes"),
    _=> println!("no"),
}
```

### `@`绑定

`@`让我们在对值进行模式匹配的同时**创建一个持有值的变量**。我们想测试`Message::Hello`的`id`成员是否在`3..=7`范围内，我们也想绑定这个值到`id_variable`，这样我们就可以在关联代码中使用它，我们也可以命名为`id`，只是例子中声明了不同的名字

```rust
enum Message {
    Hello { id: i32 },
}

let msg = Message::Hello { id: 5 };

match msg {
    Message::Hello {
        id: id_variable @ 3..=7,
    } => println!("Found an id in range: {id_variable}"),
    Message::Hello { id: 10..=12 } => {
        println!("Found an id in another range")
    }
    Message::Hello { id } => println!("Found some other id: {id}"),
}
```

通过在`3..=7`之前指定`id_variable @`，我们可以捕获匹配这个范围的任何值，也测试了这个值是否匹配了范围模式。在第二个分支中，我们只匹配范围，关联代码没有使用绑定该成员实际值的变量，这个模式中不能使用id成员的值。在最后一个分支中，我们指定了一个没有范围的变量

使用`@`可以让我们测试一个值是否匹配一个模式，并且将它保存到变量中

### 总结

Rust的模式适合在不同类型数据之间做区分。在`match`表达式中，Rust会保证你的模式覆盖类型所有的可能值，否则程序无法编译。`let`语句和函数参数中的模式让那些结构体更好用，允许解构成更小的部分，并且将这些小的组件分配给变量，我们可以创建简单或复杂的模式来满足各种需求
