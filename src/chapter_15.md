# 第十五章 - 智能指针

**指针**（*pointer*）是编程中的一个通用概念，它的值是一个内存地址。这个指针引用了，或者说“指向”一些其它数据。最常见的指针就是引用，这种引用没有其它能力，同样它们也没有其它的性能消耗。**智能指针**（*smart pointers*）是一种数据结构，它的行为很像指针，但是包含其它的元数据，提供额外能力。Rust在标准库中定义了不同的智能指针，这些智能指针提供了比引用更丰富的功能。本章介绍的**引用计数**（*reference counting*）智能指针类型，允许数据有多个所有者，当数据没有所有者后被清理。根据所有权和借用规则，Rust在引用和智能指针之间还存在其它的区别，引用只是借用数据，多数情况下智能指针拥有（own）它们指向的数据

`String`和`Vec<T>`都是智能指针，因为它们自身拥有一些内存并且允许你操作它们，它们也包含元数据和额外的能力或者保护措施。例如`String`有一些元数据元数据，可以保证数据总是仅有效的UTF-8

智能指针通常使用结构体来实现。和普通结构体不同，智能指针需要实现`Deref`和`Drop`特质。`Deref`特质允许智能指针的实例可以以引用的方式使用，所以你的代码可以同时作用于智能指针和引用。`Drop`特质允许你自定义当智能指针的实例离开域时运行的代码

## 使用`Box<T>`指向堆上的数据

使用`Box`可以让你在堆上保存数据，栈上仅保留指向堆数据的指针。`Box`除了在堆上保存的数据以外没有性能开销。它具有很多额外的能力，在下面这些场景中会使用到

- 对于编译期不知道大小的类型，你想在那些需要知道类型确切大小的上下文中使用这种类型
- 有大量的数据，你想转移它们的所有权，但是需要保证数据不发生拷贝
- 当你想拥有一个值，你只关心它是实现了某个特质的类型而不关心它的具体类型（特质对象）

### 使用`Box<T>`保存堆上的数据

```rust
fn main() {
    let b = Box::new(5);
    println!("b = {b}");
}
```

`b`是`Box`类型的值，指向真实值`5`，值是分配在堆上的，我们可以像访问栈上的数据一样去访问指针中的数据，当指针出域时，它占用的内存会被释放。这个释放过程在`b`（栈上）和它指向的数据（堆上）都会发生

### 使用Boxes来允许递归类型

**递归类型**（*recursive type*）的值可以包含同样类型的值作为它的一部分。因为Rust需要在编译期知道某个类型的大小，所以无法直接声明递归类型。递归类型的大小由于嵌套关系在理论上是无限的，所以Rust无法计算类型大小。但是`Box`的大小是已知的

#### 以*cons list*为例

> 有关Cons List的更多信息
>
> **链表**（*cons list*）是来自于Lisp语言和它的方言中的一种数据结构，由嵌套对组成。它的名字来自于`cons`函数（construct function的缩写），这是一个由两个参数构建一个对的函数。通过调用`cons`，传入一个值和另一个对，我们可以构建由递归对组成的链表`(1, (2, (3, Nil)))`，这里的`Nil`只是表示递归的基本情形，而不是`nil`

下面是一个无法编译的List类型

```rust
enum List {
    Cons(i32, List),
    Nil,
}

fn main() {
    let list = Cons(1, Cons(2, Cons(3, Nil)));
}
```

编译错误会显示这个类型*has infinite size*。这是因为我们定义`List`时使用了递归类型作为变体的一部分

#### 计算一个非递归类型的大小

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

Rust会查看枚举的每个变体，找到需要最大空间的变体。因为同一时间只有一个变体会被使用，`Message`类型的值的最大空间就是所占空间最大的那个变体所需要的空间

计算`List`时，编译器开始查看`Cons`变体，发现包含一个`i32`和一个`List`类型的值，然后继续查看`List`类型的大小，编译器会继续查看它的变体，这个过程是无限循环的

#### 使用`Box<T>`来声明一个已知大小的递归类型

编译器的建议中，*indirection*表示我们应该改变存储值的数据结构，保存一个指针而不是值本身。因为`Box<T>`是一个指针，Rust知道它的具体大小，一个指针的大小是固定的，跟指向的数据大小无关。`Box<T>`指向下一个`List`值

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
}
```

现在任何`List`类型值的大小是一个`i32`加上一个`Box`指针。通过使用`Box`，我们破坏了无限递归链，所以编译器可以计算存储`List`类型值所需的空间。`Box`只提供了间接性和指向堆分配的内存，没有其它能力。同时它也没有那些提供额外功能的智能指针所需的性能消耗。`Box<T>`类型是一个智能指针，因为它实现了`Deref`特质，允许这个类型的值可以像引用一样使用。当它的值出域时，`Box`指向的堆数据会被清理，因为它也实现了`Drop`特质

## 使用`Deref`像普通引用一样使用智能指针

`Deref`特质允许你自定义**解引用操作符**（*dereferece operator*）`*`操作的行为，你可以写一些处理引用类型的代码，它们也可以直接接受智能指针类型

### 从指针到值

普通引用是一种类型的指针，可以将指针想象成一个箭头，它指向存储在其它位置的数据

```rust
fn main() {
    let x = 5;
    let y = &x;

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

我们必须使用解引用操作符来得到引用指向的值，像使用引用一样使用`Box<T>`

```rust
fn main() {
    let x = 5;
    let y = Box::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

唯一的区别是使用了`Box`来指向`x`的值

### 定义我们自己的智能指针

`MyBox<T>`类型是一个元组结构体，有一个元素

```rust
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}
fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

上面的代码会报错，`MyBox<T>`不能解引用，因为结构体没有实现这个功能

#### 实现`Deref`特质

在标准库的定义中，`Deref`特质需要我们实现一个`deref`方法，接受`self`的不可变借用为参数，返回一个内部数据的引用

```rust
use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

`type Target = T;`语法定义了`Deref`特质使用的关联类型。关联类型是一种定义泛型参数的方式。`deref`方法返回一个引用，引用的是我们想通过`*`访问的值，实现方法之后上一节的代码会通过编译，断言也会通过。如果没有`Deref`特质，编译器只能解引用`&`引用。`deref`方法可以让编译器去获取实现了`Deref`特质的类型的值，调用`deref`方法得到它，即指针内部值的引用

当我们输入`*y`时，背后的实际代码是`*(y.deref())`，Rust将`*`替换成`deref`方法的调用，然后再使用解引用操作符，这样我们就不需要自己去调用这个`deref`方法。之所以`deref`方法返回的是引用类型，是因为在目前的情况和大多数我们使用解引用操作符的情况下，我们都不想获取`MyBox<T>`内部的值的所有权

### 函数和方法中隐式的强制解引用

**强制解引用**（*Deref coercion*）将一个实现了`Deref`特质的类型的引用转换成其它类型的引用。例如它可以将`&String`转成`&str`，因为`String`实现了`Deref`特质，返回的是`&str`。强制解引用简化了Rust中函数和方法参数的类型声明，这个机制只作用于那些实现了`Deref`特质的类型上。当我们传入某个特定类型的引用作为函数或方法的参数，但是不满足参数类型声明，就会发生强制解引用。连续的`deref`方法会将我们传入的类型转成所需的类型

```rust
fn hello(name: &str) {
    println!("Hello, {name}!");
}
```

强制解引用可以让`hello`传入`MyBox<String>`类型的值的引用

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&m);
}
```

Rust将`&MyBox<String>`通过调用`deref`转成`&String`，标准库对`String`实现的`Deref`特质会返回一个字符串切片，Rust再次调用`deref`会将`&String`转成`&str`，匹配上了`hello`的签名，如果Rust没有提供强制解引用，则必须进行手工转换

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&(*m)[..]);
}
```

当类型实现了`Deref`特质，Rust会分析类型，按需调用`Deref::deref`来得到匹配函数或方法的参数类型的引用，这是在编译期实现的，所以利用强制解引用在运行时是没有性能开销的

### 强制解引用如何和可变性交互

可以实现`DerefMut`特质来重载可变引用上的`*`操作符。当它检查到类型和对应的特质实现有以下三种情况，Rust会执行强制解引用

- `T: Deref<Target=U>`，执行`&T`到`&U`
- `T: DerefMut<Target=U>`，执行`&mut T`到`&mut U`
- `T: Deref<Target=U>`，执行`&mut T`到`&U`

前两个例子中，除了第二种实现了可变性以外都相同。如果你有一个`&T`，`T`对于类型`U`实现了`Deref`，你可以透明地得到`&U`。Rust可以强行将可变引用转成一个不可变的，但是反过来不行。由于借用规则的限制，如果你有一个可变引用，这个可变引用必须是这个数据的唯一引用。将一个可变引用转换成不可变引用不会破坏借用规则，但是反过来需要这个不可变引用是数据的唯一引用，但是借用规则保证不了这一点，所以Rust不会做这种转换

## 使用`Drop`特质在清理阶段运行代码

`Drop`特质可以让你定制值离开域的时候要做的事情。你可以为任何类型提供`Drop`特质的实现，这些代码通常是用来释放文件、网络连接等资源。`Drop`特质在实现一个智能指针时总会用到。例如当`Box<T>`被清除的时候，它会释放指向的堆上的空间。Rust中，你指定好值离开域时要运行的代码后，编译器会自动在值离开域后添加这些代码。你不需要在程序中到处写清理代码，并且不会导致资源泄漏。`Drop`特质需要你实现一个接收`self`的可变引用作为参数的`drop`方法

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer {
        data: String::from("my stuff"),
    };
    let d = CustomSmartPointer {
        data: String::from("other stuff"),
    };
    println!("CustomSmartPointers created.");
}
```

`Drop`特质定义在预包含中。`drop`方法包含实现特质的类型实例离开域时想运行的任意逻辑，例子中是打印一些数据。离开域时变量会以它们创建顺序的逆序被清理。没有方法可以禁止自动的`drop`调用，通常禁止`drop`的调用没什么必要。但是有时你可能想提前清理一个值，例如一个锁对象，你需要强制调用`drop`方法来提前释放锁，Rust不允许你手动调用`drop`方法，必须调用标准库提供`的std::mem::drop`函数在值离开域之前被释放

`drop`是一个**析构器**（*destructor*）。因为Rust会自动调用值的`drop`方法，所以不允许我们手动调用，这会导致double free错误。不需要担心仍在使用的值被意外清理，保证引用总是有效的所有权系统也会保证`drop`对于那些不再使用的值只调用一次

## `Rc<T>`：引用计数器智能指针

有些情况下，一个值可能需要有多个所有者，例如在图结构中，多条边可能会指向一个节点，这个节点理论上被所有指向它的边所拥有。一个节点在没有被任何边指向之前不应该被清理，可以通过`Rc<T>`类型来允许多重所有权。它是*referece counting*的缩写。这个类型会追踪一个值的引用数来判断这个值是否还在使用，如果这个值没有被引用，那么这个值可以被清理。当我们想在堆上存放一些数据，让程序的其它部分读取，并且我们在编译期不确定哪部分代码最后使用这部分数据，可以使用`Rc<T>`类型。如果我们知道哪部分代码最后结束，只要让那部分变成数据的所有者就可以。`Rc<T>`只能在单线程环境使用

### 使用`Rc<T>`来分享数据

对于之前的*cons list*的例子，如果两个链表想共享第三个链表

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let a = Cons(5, Box::new(Cons(10, Box::new(Nil))));
    let b = Cons(3, Box::new(a));
    let c = Cons(4, Box::new(a));
}
```

我们可以将`Cons`的定义改成引用类型，但是这样就需要添加生命周期参数。因为指定了生命周期参数，我们就指定了链表中所有的元素都必须至少和整个链表存活时间一样长，在一些场景下并不能施加这种限制

```rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}
```

也可以调用`a.clone()`而不使用`Rc::clone(&a)`，Rust习惯是在这种情况下使用后者，`Rc::clone`在实现中没有对数据做深拷贝，所以不会花费很多时间，通过使用这种写法，我们可以很好地区分深拷贝的`clone`和只是增加引用计数的拷贝

### 克隆一个`Rc<T>`增加引用计数

```rust
fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!("count after creating a = {}", Rc::strong_count(&a));
    let b = Cons(3, Rc::clone(&a));
    println!("count after creating b = {}", Rc::strong_count(&a));
    {
        let c = Cons(4, Rc::clone(&a));
        println!("count after creating c = {}", Rc::strong_count(&a));
    }
    println!("count after c goes out of scope = {}", Rc::strong_count(&a));
}
```

`Rc::strong_count`函数可以打印引用计数的值。初始值是`1`，每次调用`clone`，这个值会增加`1`。当`c`离开域，计数会减`1`。`Rc<T>`的`Drop`特质实现中会在值离开域时自动将引用计数减`1`。通过不可变引用，`Rc<T>`允许你在程序的不同部分共享只读数据。如果`Rc<T>`允许你有多个可变引用，就会破坏借用规则：同一个值拥有多个可变引用会导致数据竞争和不一致

## `RefCell<T>`和内部可变性模式

**内部可变性**（*interior mutability*）是Rust中的一种设计模式，允许你使用数据的不可变引用修改数据。为了修改数据，这个模式在数据结构中使用了unsafe代码来突破Rust通常管理可变性和借用规则的限制。不安全代码表示告诉编译器我们会自己检查规则。只有当我们能保证在运行时遵循借用规则，才能使用那些内部可变性模式的类型。不安全代码被包裹在安全的API中，外部类型仍然是不可变的

### 使用`RefCell<T>`强制运行时检查借用规则

`RefCell<T>`表示数据上只有一个所有者，回忆借用规则的限制

- 任何时候对于一个值，可以存在一个可变引用或者任意多个不可变引用，但是不能同时拥有两种引用
- 引用总是有效

对于引用和`Box<T>`，借用规则的**不变性**在编译期被保证。对于`RefCell<T>`类型，这些不变性是在运行时被保证的。如果你破坏了这个规则，程序会panic并退出。运行时检查借用规则的优势在于，一些内存安全的场景是合理的，但是无法通过编译期检查。一些代码的性质无法通过分析代码来检测到：最著名的例子是*Halting Problem*（停机问题，一个程序不能验证其它程序在任意输入下的停机情况，即不是所有问题都可以被算法化）。当你确定你的代码遵循借用规则但是编译器无法理解代码逻辑的时候可以使用`RefCell<T>`。`RefCell<T>`只能用于单线程场景

选择`Box<T>`、`Rc<T>`或者`RefCell<T>`的理由

- `Rc<T>`可以让多个所有者拥有一份数据，另外两种指针只有一个所有者
- `Box<T>`在编译期检查可变和不可变引用；`Rc<T>`只有编译期的不可变检查；`RefCell<T>`在运行时检查可变和不可变引用
- 因为`RefCell<T>`在运行时进行可变引用检查，你可以修改里面的值，即使当它本身是不可变引用

### 内部可变性：对一个不可变的值的可变引用

在包含值的方法外部的代码不能修改这个值，使用`RefCell<T>`是一种获取内部可变性的方法，它也不能破坏借用规则，编译器的借用检查器允许这种内部可变性，只是在运行时检查借用规则

#### 一个内部可变性的用例：*Mock Objects*

**假对象**（*Mock Objects*）就像电影中的替身演员，在测试中记录发生了什么，让我们判断是否发生预期行为。Rust没有同等含义的对象，标准库也没有这项内置功能。本节的测试场景是：创建一个库来追踪一个值超过最大值然后基于它跟最大值的差距发送一条信息。我们的库只提供追踪一个值有多接近最大值，什么时候发送什么信息。使用库的代码会提供一种发送信息的机制，可以将信息使用这种机制发送出去

```rust
pub trait Messenger {
    fn send(&self, msg: &str);
}

pub struct LimitTracker<'a, T: Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T>
where
    T: Messenger,
{
    pub fn new(messenger: &'a T, max: usize) -> LimitTracker<'a, T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }

    pub fn set_value(&mut self, value: usize) {
        self.value = value;

        let percentage_of_max = self.value as f64 / self.max as f64;

        if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        } else if percentage_of_max >= 0.9 {
            self.messenger
                .send("Urgent warning: You've used up over 90% of your quota!");
        } else if percentage_of_max >= 0.75 {
            self.messenger
                .send("Warning: You've used up over 75% of your quota!");
        }
    }
}
```

`Messenger`特质的`send`方法接收一个`self`的不可变引用和信息文本作为参数。这个特质是我们的假对象需要实现的。我们要测试的是`set_value`方法的行为

```rust
#[cfg(test)]
mod tests {
    use super::*;

    struct MockMessenger {
        sent_messages: Vec<String>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: vec![],
            }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);

        limit_tracker.set_value(80);

        assert_eq!(mock_messenger.sent_messages.len(), 1);
    }
}
```

在测试中，我们要测试当`LimitTracker`被设置为超过`max`的75%的`value`时会发生什么。我们使用`MockMessenger`的信息是否会出现一条。因为`send`方法接受的参数是`self`的不可变引用，同时我们也不想因为测试修改特质中此方法的定义。这就是内部可变性起作用的场景

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: RefCell::new(vec![]),
            }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        // --snip--

        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```

我们调用了`RefCell<Vec<String>>`上的`borrow_mut`来获取内部值的可变引用，然后再调用`push`方法。最后的修改是在断言部分，查看内部的vector中有多少项，调用`borrow`方法获取vector的不可变引用

### 在运行时追踪`RefCell<T>`的借用

`borrow`和`borrow_mut`是`RefCell<T>`上的安全API。`borrow`返回智能指针类型`Ref<T>`，`borrow_mut`返回智能指针类型`RefMut<T>`，两个类型都实现了`Deref`，所以可以把它们当普通引用使用。`RefCell<T>`追踪有多少`Ref<T>`和`RefMut<T>`智能指针当前有效。每次我们调用`borrow`，它会增加不可变引用的数量，当这个指针的值离开域，不可变借用的量会减`1`，和编译期借用规则一样，我们可以同时拥有多个不可变借用或者一个可变借用。如果我们尝试破坏规则，比如同时创建两个可变借用，程序会崩溃，并且会报错*already borrowed: BorrowMutError*。在运行时捕获借用错误也会让你的代码在运行时有一些性能损耗（用于追踪）

### 使用`Rc<T>`和`RefCell<T>`来让可变数据有多个所有者

结合`RefCell<T>`和`Rc<T>`是一种通用的使用方式。`Rc<T>`让你对一些数据同时拥有多个所有者，但是它只能对数据进行只读访问。如果`Rc<T>`持有`RefCell<T>`，你可以让一个值有多个所有者并且还可以修改它。在之前`cons list`的例子中，我们使用`Rc<T>`来允许多个列表共享另一个列表的所有权。因为`Rc<T>`只持有不可变值，只要我们创建之后就无法改变列表中的值

```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(3)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(4)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {a:?}"); // 15
    println!("b after = {b:?}"); // 3 15
    println!("c after = {c:?}"); // 4 15
}
```

这项技术非常简洁！通过使用`RefCell<T>`，我们有了对外不可变的`List`值。但是可以使用`RefCell<T>`上的方法以可变方式访问内部数据，我们可以根据需要修改数据。运行时的借用规则检查可以避免数据竞争，有时对于数据结构来说用一些速度换取灵活性是值得的

## 循环引用会泄漏内存

Rust的内存安全保证**永远不会被清理的内存**（*memory leak*）很难出现，但不是完全不可能出现。Rust无法保证完全阻止内存泄漏，实际上内存泄漏的情况在Rust中也是内存安全的

### 创建一个循环引用

```rust
use crate::List::{Cons, Nil};
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match self {
            Cons(_, item) => Some(item),
            Nil => None,
        }
    }
}

fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));

    println!("a initial rc count = {}", Rc::strong_count(&a));
    println!("a next item = {:?}", a.tail());

    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));

    println!("a rc count after b creation = {}", Rc::strong_count(&a));
    println!("b initial rc count = {}", Rc::strong_count(&b));
    println!("b next item = {:?}", b.tail());

    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    println!("b rc count after changing a = {}", Rc::strong_count(&b));
    println!("a rc count after changing a = {}", Rc::strong_count(&a));

    // Uncomment the next line to see that we have a cycle;
    // it will overflow the stack.
    // println!("a next item = {:?}", a.tail());
}
```

本例中我们要修改的是`Cons`变体中`List`的值。`tail`方法可以方便地访问链表中的第二个值。`a`和`b`中`Rc<List>`的引用计数都是`2`。在`main`函数最后，Rust会清除`b`，然后`b`的引用计数会变成`1`，`Rc<List>`在堆上的内存在此时不会被清理，然后Rust会清除`a`，过程和`b`一样。如果将最后一行的注释删掉，那么Rust会死循环直到栈溢出。例子中循环引用导致的后果并不严重，因为程序很快就退出了。然而，如果在更复杂的程序中，循环引用分配了很多内存，并且持续了很长时间，程序就会使用更多的内存，可能会超过系统限制，消耗尽可用内存

如果你有包含`RefCell<T>`的类型的值的`Rc<T>`值或其它类似的嵌套类型组合，由内部可变性和引用计数组成，你一定要确保没有创建循环。不能依赖Rust去捕获它们。创建一个循环引用是程序的逻辑bug，你应该使用自动化测试、代码审查和其它软件开发实践来最小化影响。另一种避免循环引用的方法是重新组织你的数据结构，一些引用暴露所有权而一些不暴露。这样你的循环引用中包含一些所有权关系和非所有权关系，只有那些所有权关系会影响一个值是否会被清理

### 使用`Weak<T>`来防止循环引用

`Rc<T>`只有当`strong_count`为`0`时才被清理。你也可以通过调用`Rc::downgrade`创建一个**弱引用**（*weak reference*），传入一个`Rc<T>`的引用。强引用是分享`Rc<T>`实例的所有权的方式。**弱引用不表达所有权关系**，它们的计数不会影响`Rc<T>`什么时候被清理。它们不会导致循环引用，因为任何包含弱引用的循环，一旦强引用的计数为`0`，就会被破坏。当你调用`Rc::downgrade`后，你得到了`Weak<T>`类型的智能指针，增加了`weak_count`的值。因为`Weak<T>`引用的值可能已经被清理了，所以如果要对它引用的值做操作，你必须保证这个值仍然存在。通过`Weak<T>`上的`upgrade`方法，你会得到一个`Option<Rc<T>>`类型的值，如果`Rc<T>`的值还没有被清理会返回`Some`，否则就是`None`

#### 创建一个树结构：包含子节点的节点

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct Node {
    value: i32,
    children: RefCell<Vec<Rc<Node>>>,
}
```

我们想让`Node`拥有所有子节点，并且可以修改某个子节点的孩子节点，所以使用了`RefCell<Vec<Rc<Node>>>`

```rust
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        children: RefCell::new(vec![]),
    });

    let branch = Rc::new(Node {
        value: 5,
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });
}
```

现在可以从`branch`访问`leaf`，但是`leaf`不知道`branch`的信息

#### 添加从孩子到父亲的引用

一个父节点应该拥有它的子节点，如果父节点被删除，它的子节点也应该被删除。然而一个子节点不应该拥有它的父节点，如果我们删除子节点，父节点仍然存在

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());

    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
}
```

可视化展示`strong_count`和`weak_count`的变化

```rust
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf), // 1
        Rc::weak_count(&leaf),   // 0
    );

    {
        let branch = Rc::new(Node {
            value: 5,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![Rc::clone(&leaf)]),
        });

        *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

        println!(
            "branch strong = {}, weak = {}",
            Rc::strong_count(&branch), // 1
            Rc::weak_count(&branch),   // 1
        );

        println!(
            "leaf strong = {}, weak = {}",
            Rc::strong_count(&leaf),   // 2
            Rc::weak_count(&leaf),     // 0
        );
    }
    // branch 被清理
    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade()); // None
    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),    // 1
        Rc::weak_count(&leaf),      // 0
    );
}
```
