# 第十八章 - 面向对象编程特性

**面向对象**（*Object-oriented programming*，OOP）是对问题建模的一种方式。对象作为编程概念在编程语言*Simula*中于1960s提出，影响了Alan Kay的编程架构，即在对象之间互相传递消息。为了描述这个架构，他在1967年提出了面向对象编程的术语。对于OOP来说，有很多含义互相对立的定义，就满足其中某些定义来说，Rust是面向对象的，对另外一些则不是

## 面向对象语言的特性

对象包含数据和行为

> 四人帮的OOP定义：
>
> 面向对象程序由对象构成。一个对象包括数据和操作这些数据的行为。这些行为被称为**方法**（*methods*/*operations*）
>
> 在这个意义上，Rust是面向对象的：结构体和枚举包括数据，`impl`块中定义它们拥有的方法，即使结构体和枚举不叫对象，但它们提供了一样的功能

### 封装：隐藏实现细节

**封装**（*encapsulation*）的含义是不让使用对象的代码访问对象的实现细节。唯一可以与对象交互的方式是它提供的公共API，使用对象的代码**不应该**访问对象的内部，使其可以直接改变对象的数据或者行为。我们可以使用`pub`关键字来决定哪些模块、类型、函数和方法是公共的，默认情况所有项对于外部都是私有的

```rust
pub struct AveragedCollection {
    list: Vec<i32>,
    average: f64,
}
```

结构体标记为`pub`所以其它代码可以使用它，但是里面的成员仍然是私有的。因为我们想保证无论何时列表中增加或减少了值，它的平均值都被正确更新。我们通过实现`add`、`remove`和`average`来实现

```rust
impl AveragedCollection {
    pub fn add(&mut self, value: i32) {
        self.list.push(value);
        self.update_average();
    }

    pub fn remove(&mut self) -> Option<i32> {
        let result = self.list.pop();
        match result {
            Some(value) => {
                self.update_average();
                Some(value)
            }
            None => None,
        }
    }

    pub fn average(&self) -> f64 {
        self.average
    }

    fn update_average(&mut self) {
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }
}
```

我们让`list`和`average`成员保持私有，外部代码不能直接在`list`中增加或删除项，如果这样`average`不会随着`list`变化而同步。`average`方法返回`average`成员的值，允许外部代码读但不能更改。因为我们将`AveragedCollection`的实现细节封装起来，所有可以很容易地修改它，例如使用的数据结构，只要`add`、`remove`、`average`这些公共方法的签名不变，使用`AveragedCollection`的外部代码就不需要改变

### 继承：作为类型系统和作为代码分享

**继承**（*inheritance*）是一种机制，通过它一个对象可以使用其它的对象定义，使得该对象在不重新定义的前提下获得父对象包含的数据和行为。Rust不支持继承，在不使用宏的前提下，没有办法定义一个继承父结构体的结构体。使用继承通常有两个主要原因。一个是**代码重用**，你可以为一个类型定义特定的行为，利用继承在其它类型上重用这些定义。在Rust代码中，默认特质方法可以起到相似的作用。另一个和类型系统有关：允许子类型在需要父类型的地方使用。这叫做**多态**（*polymorphism*），即如果子类型和父类型共享一些特性，你可以在运行时互相替换不同类型的对象

> **Polymorphism**
>
> 多态是一个通用的概念，表示那些可以同时支持多种类型数据的代码。对于继承来说，那些类型通常是子类
>
> Rust使用泛型在支持不同的可能类型上做抽象，使用**特质限制**（*trait bounds*）在那些类型必须提供的能力上添加限制。这有时叫做**有界参数多态**（*bounded parametric polymorphism*）

继承作为一种编程设计方案，在很多编程语言中已经不再支持了。因为分享超出必要范围的代码是有风险的。子类不应该总是共享所有它们的父类，但是继承允许这样。这会让程序设计变得不灵活，也会导致调用子类上的方法没有意义或者因为方法没有应用在子类上导致错误。有些语言只允许单继承，但这也限制了程序设计的灵活性

## 利用特质对象保存不同类型的值

之前的`SpreadsheetCell`枚举中有整数、浮点数和文本变体，我们可以用它表示每个单元格保存不同类型的数据，还可以使用一个vector来表示一行单元格。当我们的项在编译期的类型是固定一个集合时，这是一个完美的解决方案。然而，有时我们想让库用户可以自己扩展一些在某些情形下有效的类型

以GUI工具为例，它会遍历一系列项，调用它们的`draw`方法来将它们绘制到屏幕。我们会创建一个`gui`crate，它需要追踪不同类型的值，对不同类型的值调用`draw`方法，它不需要知道调用`draw`方法会发生什么，只要知道值有这个方法可以被调用就行。对于提供继承机制的语言，我们可能会定义一个`Component`类，包含`draw`方法。其它的类，例如`Button`、`Image`和`SelectBox`会继承`Component`类，它们会重载`draw`方法来添加自定义逻辑

### 为通用行为定义一个特质

我们要定义一个特质叫做`Draw`，有一个方法`draw`。然后我们定义一个vector来容纳特质对象，一个**特质对象**（*trait object*）包含实现了指定特质的类型的实例和一张用来在运行时查找该类型上的特质方法的表。我们通过指针来创建一个特质对象，例如`&`或者`Box<T>`智能指针，然后是`dyn`关键字，然后指定关联的特质。我们可以在使用泛型或具体类型的地方使用特质对象。无论在哪里使用特质对象，Rust的类型系统都会在编译期确保上下文中使用的任何值都会实现特质对象所要求实现的特质，因此不需要在编译期知道所有可能使用的类型。从它组织数据和行为的意义上来说，特质对象更像其它语言中的对象概念，。特质对象没有在其它语言中的对象那么有用：它们只是允许在通用行为上做抽象

```rust
pub trait Draw {
    fn draw(&self);
}
pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}
```

vector的类型是`Box<dyn Draw>`，这声明了一个特质对象类型，它表示`Box`指向的类型必须实现`Draw`特质

```rust
impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

这种工作方式和使用泛型类型参数的结构体不同，一个泛型类型参数一次只被一个具体类型替换，然而特质对象允许多种具体类型在运行时替换特质对象

### 实现特质

```rust
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        // code to actually draw a button
    }
}
```

我们为每个要绘制的组件类型实现`Draw`特质，实现代码中定义不同的绘制方式

```rust
use gui::Draw;

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        // code to actually draw a select box
    }
}
```

我们的库用户可以写如下代码

```rust
use gui::{Button, Screen};

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No"),
                ],
            }),
            Box::new(Button {
                width: 50,
                height: 10,
                label: String::from("OK"),
            }),
        ],
    };

    screen.run();
}
```

只关心一个值响应的消息而不是这个值的具体类型的概念类似于动态语言中的**鸭子类型**（*duck typing*）。`run`方法不需要知道每个组件的具体类型，它只需要调用所有组件的`draw`方法。通过指定`Box<dyn Draw>`作为`components`vector中值的类型，我们就定义了`Screen`需要的可以调用`draw`方法的类型的值。使用特质对象和Rust的类型系统写类似于使用鸭子类型的代码，让我们不需要在运行时检查一个值是否实现了某个特定方法，或者如果一个值没有实现某方法但是我们调用后不需要担心会出现错误，因为如果值没有实现这个特质对象所需要实现的特质，Rust不会编译此代码

### 特质对象执行动态分发

单态化的代码做的是**静态分发**（*static dispatch*），当编译器在编译期知道你调用的方法时执行。与之对应的是**动态分发**（*dynamic dispatch*），编译器会生成在运行时寻找要调用的方法的代码。使用特质对象时，Rust必须利用动态分发。编译器不知道使用特质对象的代码可能使用的所有类型，所以它不知道在什么类型上实现的什么方法会被调用。但是在运行时，Rust使用特质对象内部的指针就可以知道什么方法应该被调用。这个查找会产生运行时开销，而静态分发没有这项开销。动态分发也不会让编译器**内联**（*inline*）调用方法的代码，也就无法进行一些优化。Rust有一些**动态能力**（*dyn compatibility*）规则，即你可以和不可以使用动态分发的地方，规则详情参考[dyn compatibility](https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility)

## 实现一个面向对象设计模式

**状态模式**（*state pattern*）是一种面向对象的设计模式。这种模式的关键在于我们会在值的内部定义一组状态。状态可以通过一组**状态对象**（*state objects*）来表示，值基于它的状态改变执行不同的操作。状态对象共享功能。每个状态对象有它自己的行为，当它应该改变为另一个状态时会进行转换。持有状态对象的值不需要知道状态所拥有的不同行为或者什么时候在状态间转换。使用状态模式的优势在于当程序的业务需求改变时，我们不需要改变包含状态值的代码或者使用值的代码。我们只需要更新内部的状态对象的代码，修改它的规则，或者增加更多的状态对象即可

本例最后的功能目标

1. 一个博客以空草稿开始
2. 当草稿完成，博客发起审查请求
3. 当博客审查通过，就会被发布
4. 只有被发布的博客才能返回内容用来打印，所以未通过审查的博客不能被发布

```rust
use blog::Post;

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    assert_eq!("", post.content());

    post.request_review();
    assert_eq!("", post.content());

    post.approve();
    assert_eq!("I ate a salad for lunch today", post.content());
}
```

定义`Post`并在草稿状态创建一个新实例

```rust
pub struct Post {
    state: Option<Box<dyn State>>,
    content: String,
}

impl Post {
    pub fn new() -> Post {
        Post {
            state: Some(Box::new(Draft {})),
            content: String::new(),
        }
    }
}

trait State {}

struct Draft {}

impl State for Draft {}
```

`State`特质定义了博客不同的状态共享的行为。`Draft`、`PendingReview`和`Published`的状态对象，它们都需要实现`State`特质。当我们创建一个新的`Post`时，我们设置它的`state`成员为`Some`值，类型是`Box`，指向一个`Draft`结构体的实例。这保证了当我们创建一个`Post`的新实例时，状态都会从草稿开始

保存博客内容的文本

```rust
impl Post {
    // --snip--
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
}
```

这个行为不依赖博客当前的状态，它也不是状态模式的一部分

保证博客草稿的内容是空的

```rust
impl Post {
    // --snip--
    pub fn content(&self) -> &str {
        ""
    }
}
```

现在先将直接返回空字符串作为`content`方法的占位实现

请求审查方法会改变博客的状态

```rust
impl Post {
    // --snip--
    pub fn request_review(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.request_review())
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<dyn State>;
}

struct Draft {}

impl State for Draft {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        Box::new(PendingReview {})
    }
}

struct PendingReview {}

impl State for PendingReview {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }
}
```

`State`的`request_review`方法第一个参数是`self: Box<Self>`，表示这个方法只有调用的实例类型是`Box`时才有效，这里会获取`Box<Self>`的所有权，将`Post`的历史状态置为无效并转变为新状态。要消费旧状态，`request_review`方法需要获取状态的所有权，这就是`Option`的作用：我们调用`take`方法获取`state`成员的`Some`那的值，留下`None`，因为Rust不允许我们有未初始化的成员。这让我们将`state`从`Post`移除而不是借用，然后我们设置博客的`state`值为这个操作的结果。我们需要暂时将`state`设置为`None`，而不是直接使用`self.state = self.state.request_review();`来获取`state`值的所有权，这保证了`Post`在我们转移到新状态后不会使用旧的`state`值。从这里能看到状态模式的优点：`Post`上的`request_review`方法只有一个，无论其状态是什么值。每个状态只关心它自己的转变规则

添加`approve`方法来改变`content`的行为

```rust
impl Post {
    // --snip--
    pub fn approve(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve())
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<dyn State>;
    fn approve(self: Box<Self>) -> Box<dyn State>;
}

struct Draft {}

impl State for Draft {
    // --snip--
    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
}

struct PendingReview {}

impl State for PendingReview {
    // --snip--
    fn approve(self: Box<Self>) -> Box<dyn State> {
        Box::new(Published {})
    }
}

struct Published {}

impl State for Published {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }

    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
}
```

我们调用`Option`上的`as_ref`方法，因为我们需要`Option`内部的引用而不是所有权。`state`是一个`Option<Box<dyn State>>`，调用`as_ref`会返回一个`Option<&Box<dyn State>>`，之后调用`unwrap`是安全的，因为`state`总是包含`Some`

```rust
trait State {
    // --snip--
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        ""
    }
}

// --snip--
struct Published {}

impl State for Published {
    // --snip--
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        &post.content
    }
}
```

我们为`content`方法写了一个返回空字符串的默认实现。所以我们不需要再为`Draft`和`PendingReview`实现这个方法。`Published`需要重载这个方法，返回`post.content`内的值。这里需要提供生命周期注解，我们接收一个`post`的引用作为参数，返回它其中某一部分的引用，所以返回引用的生命周期和`post`一致

> 为什么不使用枚举？
>
> 使用枚举也可以实现这些功能。使用枚举的缺点之一是任何需要检查枚举变体的地方都需要写一个`match`表达式来处理所有的可能性，对比使用特质对象的方式会多出很多重复代码

### 状态模式的取舍

`Post`本身不提供任何行为，我们组织代码的方式是我们只需要查看一个地方就能知道一个已发布的博客可以执行的不同操作：`Published`结构体对`State`特质的实现。如果我们使用其它的实现方式，我们可能会在`Post`上定义方法，使用`match`表达式或者在`main`代码检查博客的状态然后执行不同的行为，这意味着我们需要检查很多代码才能理解一个处于发布状态的博客所有的可能变化。这会增加我们添加的状态数量：每个`match`表达式都需要一个默认分支

使用状态模式的实现很容易扩展添加更多功能，试试做下面这些事

- 添加`reject`方法，从`PendingReview`修改为`Draft`
- 在状态修改为`Published`之前需要两次`approve`
- 允许用户只在`Draft`状态添加文本内容。提示：让状态对象负责内容的改变但是不修改`Post`的API

状态模式的缺点之一是：因为状态实现的内容是状态之间的转换，一些状态会互相耦合。如果我们在`PendingReview`和`Published`之间添加一个`Scheduled`状态，我们就需要改变`PendingReview`的代码转为`Scheduled`，如果希望`PendingReview`在添加新状态后也不需要修改代码，但是这就不再是状态模式了。另一个缺点是有很多重复逻辑，为了减少重复，我们可能要对`State`特质的`request_review`和`approve`方法提供默认实现，可以直接返回`self`，但是这样做不行，在将`State`作为特质对象使用时，特质不知道具体的`self`是什么类型，所以编译期也不能确认返回类型是什么。其它的重复代码包括`Post`上的`request_review`和`approve`方法的实现。它们都使用了`Option::take`，即如果`state`为`Some`，获取其中值的所有权，并设置新值，如果有很多这种用法，我们可能需要考虑定义一个宏来减少重复。我们可以做一些改变，让`blog`crate中的无效状态转变为编译期错误

#### 将状态和行为编码为类型

我们将状态编码为不同的类型，Rust的类型检查系统会阻止在使用`Published`的地方使用`Draft`，这会产生编译错误

```rust
fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    assert_eq!("", post.content());
}
```

我们让新建的草稿博客没有`content`方法，这样我们就不可能意外展示出草稿博客的内容

```rust
pub struct Post {
    content: String,
}

pub struct DraftPost {
    content: String,
}

impl Post {
    pub fn new() -> DraftPost {
        DraftPost {
            content: String::new(),
        }
    }

    pub fn content(&self) -> &str {
        &self.content
    }
}

impl DraftPost {
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
}
```

使用类型间的转变实现博客状态转换

```rust
impl DraftPost {
    // --snip--
    pub fn request_review(self) -> PendingReviewPost {
        PendingReviewPost {
            content: self.content,
        }
    }
}

pub struct PendingReviewPost {
    content: String,
}

impl PendingReviewPost {
    pub fn approve(self) -> Post {
        Post {
            content: self.content,
        }
    }
}
```

`request_review`和`approve`方法获取`self`的所有权，因此消费了`DraftPost`和`PendingReviewPost`实例。因为唯一可以获取已发布的`Post`实例的方法是调用`approve`方法，我们将博客的**发布流程编码进了类型系统**

```rust
use blog::Post;

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");

    let post = post.request_review();

    let post = post.approve();

    assert_eq!("I ate a salad for lunch today", post.content());
}
```

状态之间的改变行为不再全部封装在`Post`的实现中，我们得到的效果是不再存在无效状态，因为类型系统和类型检查全部发生在编译期，这保证一些bug，例如展示未发布的博客，会在发布到生产环境之前被检查出来。由于一些面向对象语言所没有的特性的存在，例如所有权系统，面向对象的编码模式在Rust中并不总是最佳方案
