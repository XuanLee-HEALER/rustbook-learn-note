# 第五章 - 用结构体组织关联数据

## 定义和初始化结构体

结构体和元组的比较

- 相同点：它们都是包含多个关联的值，这些值的类型可以不同
- 不同点：在结构体中必须对每个值进行命名。使用命名来访问值比使用值的声明顺序灵活的多

结构体的名称应该可以说明它组织到一起的数据的具体含义，其中包含的每个数据我们叫做**成员**（*fields*）

```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}
```

通过对每个成员指定具体的值，我们可以创建一个结构体的**实例**（*instance*）。在创建实例时我们可以不按声明时的顺序来指定成员的值

```rust
fn main() {
    let user1 = User {
        active: true,
        username: String::from("someusername123"),
        email: String::from("<someone@example.com>"),
        sign_in_count: 1,
    };
}
```

使用点操作符`.`获取指定的值。如果实例可变，我们可以使用点操作符和赋值来修改指定的成员的值

```rust
fn main() {
    let mut user1 = User {
        active: true,
        username: String::from("someusername123"),
        email: String::from("<someone@example.com>"),
        sign_in_count: 1,
    };

    user1.email = String::from("anotheremail@example.com");
}
```

要注意Rust不允许我们仅标记某些成员是可变的，只允许整个结构体的实例是可变的。和任何表达式一样，在函数的末尾构建一个结构体的实例就会将其作为返回值

```rust
fn build_user(email: String, username: String) -> User {
    User {
        active: true,
        username: username,
        email: email,
        sign_in_count: 1,
    }
}
```

上面代码中将参数名称写得和结构体成员名称相同很重要，原因见下一小节

### 使用成员初始化简写

我们可以使用**成员初始化简写**（*field init shorthand*）语法来重写`build_user`

```rust
fn build_user(name: String, email: String) -> User {
    User {
        active: true,
        name,
        email,
        sign_in_count: 1,
    }
}
```

### 使用结构体更新语法从其它实例创建新实例

**结构体更新语法**（*struct update syntax*），下面例子中我们创建了一个新的`User`实例，除了`email`以外都使用`user1`的值

```rust
fn main() {
    // --snip--

    let user2 = User {
        active: user1.active,
        username: user1.username,
        email: String::from("another@example.com"),
        sign_in_count: user1.sign_in_count,
    };
}
```

使用结构体更新语法我们可以用更少的代码做到同样的事，`..`语法表示结构体剩余成员应该和指定的实例的值相同

```rust
fn main() {
    // --snip--

    let user2 = User {
        email: String::from("another@example.com"),
        ..user1
    };
}
```

`..user1`**必须在最后一行**表示剩下的所有其它成员，其它被指定值的成员仍然可以以任意顺序排列。结构体更新语法使用了`=`，像一次赋值操作，它也移动了数据。在这个例子中，我们在创建了`user2`后不能再以一个整体来使用`user1`，因为`user1`的`username`成员被移动到了`user2`。如果我们仅使用`user1`的`active`和`sign_in_count`，另外两个都新建，那么`user1`在创建`user2`后依然有效。因为`active`和`sign_in_count`都实现了`Copy`特质

### 使用无命名成员的元组结构体创建不同的类型

Rust也支持**类元组的结构体**（*tuple structs*）。结构体有名字而成员没有命名，或者说只有成员的类型。当你想给元组一个名字并且让元组成为和其它相同类型元组不同的类型时可以使用，还有当给普通的结构体的每个成员命名很麻烦时也可以使用

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);
}
```

`black`和`origin`值的类型不一样，因为它们是不同结构体的实例。元组结构体和元组的比较

- 相同点：可以将元组结构体的成员分给各自的部分（模式匹配），可以使用`.`和索引来访问成员
- 不同点：元组结构体解构时需要指定类型的名称`let Point(x, y, z) = point;`

### 没有任何成员的类似Unit元组的结构体

你可以定义没有任何成员的结构体。这叫做*unit-like structs*因为它们类似于`()`。这种结构体当你想在某个类型上实现特质但它本身不包含任何数据时使用，即你仅希望保存类型时很有用

```rust
struct AlwaysEqual;

fn main() {
    let subject = AlwaysEqual;
}
```

这种结构体的**每个实例都完全相等**。我们要为这个类型实现一些行为，但是这些行为完全不需要任何数据

### 结构体数据的所有权

在`User`结构体定义中我们使用`String`而不是`&str`类型。这是因为我们想让每个结构体的实例都拥有它们自己所有的数据，只要结构体有效，成员数据就有效。结构体可以拥有其他位置的数据的引用成员，但是需要使用**生命周期**（*lifetime*）标记

## 使用结构体的一个示例程序

我们先写一个计算矩形面积的程序，开始使用单一变量，然后重构为使用结构体

```rust
fn main() {
    let width1 = 30;
    let height1 = 50;

    println!(
        "The area of the rectangle is {} square pixels.",
        area(width1, height1)
    );
}

fn area(width: u32, height: u32) -> u32 {
    width * height
}
```

通过调用`area`函数可以正确计算矩形面积。但是它有两个参数，并且这两个参数的联系并不清晰。我们需要将宽和高组织起来提高可读性和可管理性

我们先使用元组来重构

```rust
fn main() {
    let rect1 = (30, 50);

    println!(
        "The area of the rectangle is {} square pixels.",
        area(rect1)
    );
}

fn area(dimensions: (u32, u32)) -> u32 {
    dimensions.0 * dimensions.1
}
```

现在函数需要传入一个参数。但是元组不能为它的元素命名，我们只能使用索引来访问元素，这样同样不太清晰。对于计算面积，宽和高的出现位置并不重要，但是如果要绘制一个矩形，那么我们必须记住`width`对应`0`而`height`对应`1`，这对其他使用此代码的人很困难

现在使用结构体来重构：增加更多的含义

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!(
        "The area of the rectangle is {} square pixels.",
        area(&rect1)
    );
}

fn area(rectangle: &Rectangle) -> u32 {
    rectangle.width * rectangle.height
}
```

我们定义了`Rectangle`结构体，有两个成员`width`和`height`，类型都是`u32`。`area`函数现在只有一个参数，类型是`Rectangle`的不可变引用，函数中使用成员名称来访问。现在`area`函数签名展示了宽和高互相关联，并且使用名称来访问数据

### 使用继承特质来添加有用的功能

`println!`宏中的`{}`需要使用那些类型的`Display`特质实现代码中指定的格式，为终端用户输出。那些基本类型默认实现了`Display`特质。结构体默认是没有实现`Display`特质的。在花括号中加上`:?`表示我们想使用`Debug`格式，`Debug`特质允许我们以对开发者有用的格式打印结构体。但是结构体默认也没有实现`Debug`特质

在结构体定义之上添加`#[derive(Debug)]`可以继承`Debug`特质。当我们有更大的结构体时，需要更易读的形式，可以使用`:#?`格式。另一种使用`Debug`格式打印值的方式是`dbg!`宏，它会获取表达式的所有权（`println!`只是使用值的引用），然后打印宏调用的文件和行号和表达式的计算值，然后返回这个值的所有权。调用`dbg!`宏会打印到标准错误控制台流中`stderr`

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let scale = 2;
    let rect1 = Rectangle {
        width: dbg!(30 * scale),
        height: 50,
    };

    dbg!(&rect1);
}
```

我们不想让`dbg!`获取`rect1`的所有权，所以使用引用。使用`dbg!`宏也可以打印调试信息，但它会获取数据的所有权

## 方法语法

**方法**（*Methods*）和函数类似，包括声明方式和定义内容。和函数不同的是，方法是定义在结构体（或者枚举、特质对象中）上下文中的，它的第一个参数总是`self`，表示方法被调用时所在的那个结构体的实例

### 定义方法

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!(
        "The area of the rectangle is {} square pixels.",
        rect1.area()
    );
}
```

要在`Rectangle`上定义方法，首先要使用一个`impl`块。这个块中所有的内容都与`Rectangle`类型相关联。然后我们将`area`函数移进去，将第一个参数改为`self`。在`main`中我们可以使用**方法语法**（*method syntax*）来调用`Rectangle`实例上的`area`方法

在签名中我们使用的是`&self`，`&self`是`self: &Self`的缩写，`impl`块中使用`Self`类型表示所实现的那个类型的别名。方法的第一个参数必须是`Self`类型，如果参数名是self，Rust允许使用缩写（省略类型）。`self`前使用`&`表示这个方法引用了`Self`实例，方法可以使用引用、获取所有权和不可变引用三种方式接受参数

`&mut self`获取`Self`的可变引用。`self`是获取所有权，获取所有权的方法通常会把`self`转化成其它值，并且你不想在这个过程结束后让调用者还能使用原来的实例

使用方法而不是函数的主要原因除了提供方法语法和不重复在每个函数中传入`self`之外，主要是为了**组织代码**（*organization*）。我们将一个类型的实例能做的所有事情都放到`impl`块中，而不是让代码的用户搜索某个类型出现的所有位置

> ⚠️我们可以使用成员名称作为方法名称。通常（并不一定）我们使用和成员名相同的方法名时，这个方法只是简单返回对应成员的值。这种方法叫做*getters*。Rust不会自动为类型实现它们，这种方法很有用，因为你可以让成员私有然后方法公有，通过这种方式来对成员进行只读访问

Rust有一个特性叫做*automatic referencing and dereferencing*。调用方法也应用了此特性。当你调用一个方法`object.something()`，Rust会自动加上`&`、`&mut`或者`*`让调用方法的实例匹配方法的签名。这种自动引用的行为可以实现是因为方法有明确的接收者（`self`的类型），只要有接收者和方法名称，Rust可以找到方法的`Self`的具体类型

### 更多参数的方法

```rust
impl Rectangle {
    fn area(&self) -> u32 {
        self.width* self.height
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}
```

方法中多个参数和函数中多个参数的工作方式相同

### 关联函数

所有定义在`impl`块中的函数都叫做**关联函数**（*associated functions*），因为它们都关联于某个类型。我们可以定义没有self参数的关联函数，它们不需要类型的实例作为参数。我们已经使用过的`String::from`就是这种函数。关联函数通常用作构造器，会返回一个结构体的实例。这些构造器通常叫做`new`，`new`不是一个特殊的名称，语言中也不是关键字

```rust
impl Rectangle {
    fn square(size: u32) -> Self {
        Self {
            width: size,
            height: size,
        }
    }
}
```

返回值和函数体中的Self关键字表示`impl`块所实现类型的别名。要调用关联函数，我们使用`::`语法结合结构体和函数名称。函数被组织在结构体的命名空间下。`::`语法用在关联函数和模块创建的命名空间中

#### 多个`impl`块

每个结构体都可以定义多个`impl`块。虽然没有必要将方法分到多个`impl`块中，但这是允许的。之后讨论泛型类型和特质时会看到多`impl`块的情况
