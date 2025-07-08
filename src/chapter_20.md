# 第二十章 - 高级特性

## 不安全的Rust

Rust存在第二种语言，在这种语言里没有强制的内存安全保障：叫做**不安全的Rust**（*unsafe Rust*），虽然它和安全的Rust工作方式相同，但是提供了额外的能力。因为编译器对代码的静态分析总是保守的，所以不安全的Rust存在是合理的。当编译器为了保证代码处于安全的保障中，拒绝有效的程序要比接受无效的程序更好。尽管代码可能没问题，如果Rust编译器没有足够的信息，它就会拒绝编译这部分代码。这些情况下你可以使用不安全的Rust告诉编译器“我知道我正在做什么”。要注意，如果你错误地使用不安全代码，由于内存不安全导致的问题仍然会出现，例如解引用空指针。另一个不安全的Rust存在的原因是底层的计算机硬件本身就是不安全的。如果Rust不让你做不安全的操作，你可能无法完成一些工作。Rust会给予你做底层的系统编程的能力，例如直接和操作系统交互甚至于写你自己的操作系统

### 不安全的超能力

要切换到不安全的Rust，需要使用`unsafe`关键字，然后开启一个新的块，里面包含不安全代码。在块中支持五种安全的Rust中做不了的操作，这叫做`不安全的超能力`（*unsafe superpowers*）

- 解引用**裸指针**（*raw pointer*）
- 调用不安全的函数或方法
- 访问或修改可变的静态变量
- 实现一个不安全的特质
- 访问`union`中的成员

要理解`unsafe`，其中一个重点是，它并没有关闭借用检查器，或者禁用了任何Rust的其它安全检查：如果你在不安全代码中使用了引用，它仍然会被检查。`unsafe`关键字只是允许你做上述五种操作，编译器不会检查它们的内存安全性。但是你还可以在不安全块中得到一定程度的安全性保证。此外，`unsafe`不代表里面的代码一定是危险的或者一定有内存安全问题：关键在于程序员，你需要保证`unsafe`中的代码以有效的方式访问内存

对于`unsafe`中使用的五种不安全操作，人们可能会犯错，但是内存安全的错误一定在unsafe块中出现。保持`unsafe`块尽量小。并且为了尽可能隔离不安全代码，最好将这些代码包装进安全的抽象中，提供一个安全的API。标准库的一部分实现就是在不安全代码上套一层安全抽象。将不安全代码嵌套进安全抽象可以阻止`unsafe`泄漏到所有你或者你的用户使用这些代码功能的地方，所有使用一个安全抽象是更安全的

#### 解引用一个裸指针

不安全的Rust有两种新类型，统一叫*raw pointers*，和引用类似。裸指针也可以是可变的和不可变的，写作`*const T`和`*mut T`。这里的星号不是解引用操作符，而是类型名的一部分。在裸指针的上下文中，不可变表示这个指针在解引用之后不能被直接赋值

和引用以及智能指针不同，裸指针

- 允许忽略借用规则，即允许同时拥有值的可变和不可变指针或者同一位置有多个可变指针
- 不保证指向有效内存
- 允许为空
- 不实现任何自动清理

通过放弃Rust对于普通引用和智能指针应用的强制限制，你可以得到更好的性能，还能与不提供Rust安全保障的其它语言或者硬件进行交互

```rust
let mut num = 5;

let r1 = &raw const num;
let r2 = &raw mut num;
```

我们可以在安全代码中创建裸指针，只是不能在**不安全块之外**解引用裸指针。我们使用`&raw const num`创建了一个`*const i32`不可变裸指针，`&raw mut num`创建了`*const mut i32`可变裸指针。因为我们直接从局部变量创建了它们，所以这些特殊的裸指针是有效的，但是我们不能假设所有裸指针都有效。接下来我们创建一个无法确定有效性的裸指针，使用`as`来转换值的类型。尝试直接使用任意内存是未定义的行为：这个地址上可能有数据也可能没有，编译器可能会优化代码，导致不访问内存，或者项目以段错误终止

```rust
let address = 0x012345usize;
let r = address as *const i32;
let mut num = 5;

let r1 = &raw const num;
let r2 = &raw mut num;

unsafe {
    println!("r1 is: {}", *r1);
    println!("r2 is: {}",*r2);
}
```

创建一个指针本身没有危害，只有当我们访问它指向的值时，程序可能会由于访问无效值而终止。在之前的代码中，我们同时创建了可变和不可变的裸指针，指向同一个内存地址。在安全代码中这样做是无法通过编译的。我们可以对裸指针这样操作，通过可变指针也能修改值，但是这样隐式地引入了数据竞争。主要使用裸指针的场景之一是和C代码交互。另一个场景是构建借用检查器无法理解的安全抽象

#### 调用一个不安全函数或方法

不安全的函数和方法看起来和普通函数和方法相似，只是在定义前有`unsafe`标记。这个上下文中的`unsafe`关键字表示当我们调用这个函数时需要满足它的需求，因为编译器无法检查我们是否满足这个要求。通过在`unsafe`块中调用不安全函数，就表示我们已经查看了函数的文档并且我们负责满足符合这个函数的要求

```rust
unsafe fn dangerous() {}

unsafe {
    dangerous();
}
```

我们必须在单独的`unsafe`块中调用`dangerous`函数。在`unsafe`块中，我们告诉Rust我们已经阅读了函数的文档，我们已经理解了如何正确使用它，确定我们满足了函数的要求。要在不安全函数的函数体内执行不安全操作，需要使用`unsafe`块，就像在普通函数中一样，如果忘记使用编译器会发出警告。这也帮助你将`unsafe`块大小尽可能保持简短，因为不安全操作通常不需要覆盖整个函数体

#### 在不安全代码的基础上创建安全抽象

使用安全函数嵌套不安全代码是常见的抽象

```rust
let mut v = vec![1, 2, 3, 4, 5, 6];

let r = &mut v[..];

let (a, b) = r.split_at_mut(3);

assert_eq!(a, &mut [1, 2, 3]);
assert_eq!(b, &mut [4, 5, 6]);
```

我们不能完全使用安全的Rust代码来实现这个函数，像下面这个实现示例不会通过编译

```rust
fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = values.len();

    assert!(mid <= len);

    (&mut values[..mid], &mut values[mid..])
}
```

函数体中，我们先获取切片长度。然后断言给定的分隔点一定小于等于长度，如果给定的分隔点大于切片长度，函数会直接`panic`。然后我们返回包含两个可变切片的元组。Rust的借用检查器无法理解这两个引用分别指向原切片的不同部分，它只知道我们借用了这个切片两次，虽然这两个切片没有重叠部分，即这个操作本身是安全的，但是Rust不会知道这一点

```rust
use std::slice;

fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = values.len();
    let ptr = values.as_mut_ptr();

    assert!(mid <= len);

    unsafe {
        (
            slice::from_raw_parts_mut(ptr, mid),
            slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```

切片是一个指向一些数据指针，包括切片的长度。我们使用`len`方法获取切片长度，`as_mut_ptr`方法来获取切片的裸指针。因为操作的是`i32`的可变切片，`as_mut_ptr`返回一个类型为`*mut i32`的裸指针，保存到`ptr`中。`slice::from_raw_parts_mut`函数接受裸指针和长度为单位，会创建一个切片。然后我们调用`ptr`的`add`方法，使用`mid`作为参数获取指向`mid`位置切片的裸指针，然后使用这个裸指针创建剩余部分的切片。`slice::from_raw_parts_mut`函数是不安全的，因为它接受裸指针参数，所以它必须相信这个指针是有效的。裸指针上的`add`方法也是不安全的，因为它必须相信这个偏移位置也是一个有效的内存地址。因此，我们必须在`unsafe`块中调用这两个方法。我们不需要将`split_at_mut`函数标记为`unsafe`，这样我们就可以在安全的Rust中调用它，我们在不安全代码上创建了安全的抽象，其中以安全方式使用`unsafe`代码，因为它只从这个函数访问的有效数据上创建了有效的指针。相反，下面代码中使用`slice::from_raw_parts_mut`的方式可能会导致程序崩溃

```rust
use std::slice;

let address = 0x01234usize;
let r = address as *mut i32;

let values: &[i32] = unsafe { slice::from_raw_parts_mut(r, 10000) };
```

我们不拥有在这个随机位置的内存，所以也无法保证这个代码创建的切片包含有效的i32值，尝试使用`values`的值会导致未定义行为

#### 使用`extern`函数来调用外部代码

有时你可能需要和其它语言写的代码进行交互。Rust提供关键字`extern`来辅助创建和使用*Foreign Function Interface(FFI)*。FFI是一种编程语言定义函数和允许不同的编程语言来调用这些函数的方式。Rust代码调用的`extern`块中声明的函数通常是不安全的，所以`extern`块**必须**同时被标记为`unsafe`，理由是其它编程语言没有提供Rust安全规则的保障，所以Rust无法检查它们，由程序员来负责保证安全性

```rust
unsafe extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```

`unsafe extern "C"`中列出了我们想调用的来自其它语言的外部函数的名字和签名。`"C"`定义了外部函数使用了哪种*application binary interface(ABI)*，ABI定义了以何种汇编级别调用函数。`"C"`ABI是最常用的，符合C编程语言的`ABI`。在`unsafe extern`中声明的项默认都是`unsafe`的，然而一些FFI函数是可以安全调用的。例如C标准库的`abs`函数没有内存安全问题，我们知道它可以使用任何`i32`值调用。像这种情况，我们可以使用`safe`关键字指定这个函数是安全的。当我们做了指定，调用它就不需要unsafe块了

```rust
unsafe extern "C" {
    safe fn abs(input: i32) -> i32;
}

fn main() {
    println!("Absolute value of -3 according to C: {}", abs(-3));
}
```

标记一个函数为`safe`并不会让它变得安全，它就像一个**承诺**，你告诉Rust这个函数是安全的，但是还需要你负责保证这一点

##### 从其它语言中调用Rust函数

我们也可以使用`extern`创建一个接口，让其它编程语言调用Rust函数提供的接口。不是创建整个`extern`块，我们使用`extern`关键字，只在那些有关的函数的`fn`关键字之前指定ABI。我们也需要加上`#[unsafe(no_mangle)]`注解告诉编译器不要**混淆**（*mangling*）这个函数的名字。混淆是编译器为编译过程的其它部分提供更多信息会修改我们给定函数的名称的过程，但是这个名称不是人可读的。所有编程语言编译器都或多或少会混淆函数名称，所以Rust函数必须允许其它编程语言可以重命名，所以我们要禁止Rust编译器修改名称。这个操作也是不安全的，因为不使用内置的混淆可能会导致库出现名称冲突，所以我们需要确保选择的名称在不混乱的情况下可以安全导出

```rust
#[unsafe(no_mangle)]
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```

`call_from_c`可以在C代码中访问，将它编译为共享库并链接到C上。这种`extern`使用的方法只在属性中使用`unsafe`，不需要在extern块上添加

#### 访问或修改可变静态变量

Rust支持全局变量，在Rust的所有权规则下使用全局变量很容易出问题。如果两个线程访问同一个可变全局变量，还会造成数据竞争。在Rust中，全局变量叫做**静态**（*static*）变量

```rust
static HELLO_WORLD: &str = "Hello, world!";

fn main() {
    println!("name is: {HELLO_WORLD}");
}
```

静态变量和常量类似。静态变量的命名规则通常是*SCREAMING_SNAKE_CASE*。静态变量只能保存`'static`生命周期的引用，Rust编译器可以识别生命周期，所以我们不需要显式注解。访问不可变的静态变量是安全的。常量和不可变静态变量之间有一些微小的区别：静态变量在内存中有固定的地址。使用这个值总会访问到相同的数据，而常量可以在使用它们时进行数据的复制。另一个区别是静态变量可以是可变的，只是访问和修改可变静态变量是不安全的操作

```rust
static mut COUNTER: u32 = 0;

/// SAFETY: Calling this from more than a single thread at a time is undefined
/// behavior, so you *must* guarantee you only call it from a single thread at
/// a time.
unsafe fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    unsafe {
        // SAFETY: This is only called from a single thread in `main`.
        add_to_count(3);
        println!("COUNTER: {}", *(&raw const COUNTER));
    }
}
```

任何读写`COUNTER`的代码必须放在`unsafe`块中。多个线程访问`COUNTER`可能会导致数据竞争，所以也是未定义行为。因此我们需要将整个函数标记为`unsafe`，并且还应该为这个函数写有关安全限制的文档，让调用函数的人知道他们在保证安全的情况下应该做什么和不做什么

当我们写不安全函数时，按照惯例需要写`SAFETY`开头的文档，解释调用者如果想安全调用函数需要做什么事情。同样的，当我们执行不安全操作时，符合惯例的方式也是写`SAFETY`开头的注释来解释我们是如何确保不安全的代码符合安全规则的。此外，编译器不允许你创建可变静态变量的引用。你只能通过裸指针来访问它，使用**裸借用操作符**（*raw borrow operators*）。这包括在引用被隐式创建的情况下，例如在上面代码的`println!`中。对静态可变变量的引用只能通过裸指针创建的要求，有助于更明显地体现出使用它们的安全性

对于可以全局访问的可变数据，很难保证不发生数据竞争，这就是为什么Rust认为可变静态变量是不安全的。最好使用并发和线程安全提供的智能指针，这样编译器可以在不同线程对数据的访问中检查操作是否安全

#### 实现一个不安全的特质

当它定义的方法中有一些编译器无法验证的不变量，这个特质就是不安全的，我们可以使用`unsafe`来实现一个不安全的特质。我们将`unsafe`添加到`trait`之前，声明这个特质是不安全的，并且要对实现这个特质的代码也标记为`unsafe`

```rust
unsafe trait Foo {
    // methods go here
}

unsafe impl Foo for i32 {
    // method implementations go here
}
```

例如`Sync`和`Send`，如果我们的类型完全由其它实现了这些特质的类型组成，编译器会自动为这些类型实现这些特质。如果类型包含没有实现这两个特质的类型，例如裸指针，然后我们想标记这些类型是`Send`或`Sync`，那么我们必须使用`unsafe`。Rust无法验证我们的类型可以保证它能安全地在线程之间传递并且可以支持多线程访问它的引用，因此我们需要手动做这些检查并使用`unsafe`标识

#### 访问`Union`中的成员

`union`和结构体类似，在一个特定实例中，一次只能使用其中一个声明的字段。`union`主要用来和C代码中的`union`交互。访问成员是不安全的操作，因为Rust无法保证`union`实例中存储的数据类型[Union](https://doc.rust-lang.org/reference/items/unions.html)

### 使用Miri来检查不安全代码

在写不安全的Rust时，你会想检查你写的是否安全和正确。Rust的官方工具Miri可以检测程序中的未定义行为。借用检查器是静态工具，在编译期工作，Miri是一个工作于运行时的动态（dynamic）工具，它通过运行你的程序来检查，或者使用它的测试套件，如果你的程序破坏了它所理解的Rust的工作规则，它会检测到问题

使用Miri需要Rust的nightly构建，通过`rustup +nightly component add miri`可以安装Rust的nightly版本和Miri工具。你可以在一个项目中通过`cargo +nightly miri run`或者`cargo +nightly miri test`来运行Miri。Miri是一个动态分析工具，所以它只会捕获实际运行的代码的问题。所以你需要结合优秀的测试技术来使用它，从而提高不安全代码的质量。要注意Miri没有捕获的bug不代表没有问题

### 什么时候该使用不安全代码

使用`unsafe`来利用之前提到的五种超能力没有错误，也不该被排斥，但是让`unsafe`代码正确工作非常困难，因为编译器无法帮助你检查内存安全。当你有理由使用`unsafe`代码时，就去用它，利用`unsafe`注解可以很容易地在追踪到问题源。你也可以使用Miri来写更符合Rust规则的不安全代码

## 高级特质

### 关联类型

**关联类型**（*associated types*）将类型占位符和特质联系起来，特质中的方法定义可以在签名中使用这些占位符类型。特质的实现者需要指定具体类型。根据这种方式，我们能定义一个使用被实现前未知的类型的特质。关联类型的使用频率低于之前的常用功能，但是要高于本章的其它内容。一个定义了关联类型的特质的例子是标准库提供的`Iterator`

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

`Item`是一个占位符，`next`方法的返回类型是`Option<Self::Item>`，`Iterator`的实现者会为`Item`指定具体类型，`next`方法就会返回这个具体类型的`Option`。关联类型看起来和泛型很像，泛型可以让我们定义一个不需要指定它能处理的类型的函数

```rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // --snip--

pub trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```

为什么不直接使用泛型定义`Iterator`呢？关联类型和泛型的区别是，如果使用泛型，我们必须在每个实现中都注解类型，我们可以`impl Iterator<String> for Counter`或者其它任何类型，我们可能会对`Counter`实现多个`Iterator`。也就是说，当特质有一个泛型参数，它可以对一个类型实现多次，每次泛型的具体类型不同。当我们使用`next`方法时，我们必须提供类型注解来表明我们想使用哪个实现。利用关联类型，我们不需要注解类型。我们只要选择一次`Item`的类型就可以了。关联类型是特质规则的一部分。特质的实现者必须提供为关联类型提供一个具体类型。关联类型的名称通常会描述它如何使用，在API文档中注释关联类型是一个优秀实践

### 默认的泛型类型参数和操作符重载

当我们使用泛型类型参数时，我们可以指定一个默认的具体类型。这减少了特质的实现者在使用默认类型时还需要指定它的具体类型的工作。`<PlaceholderType=ConcreteType>`语法可以在声明泛型类型的时候指定默认类型。这种技术在**操作符重载**（*operator overloading*）中很有用。Rust不允许你创建自己的操作符或者直接重载任意操作符，但是你可以通过实现操作符有关的特质重载`std::ops`中列出的操作符和相关特质

```rust
use std::ops::Add;

#[derive(Debug, Copy, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    assert_eq!(
        Point { x: 1, y: 0 } + Point { x: 2, y: 3 },
        Point { x: 3, y: 3 }
    );
}
```

`add`方法将`x`的值和`y`的值相加来创建一个新的`Point`，`Add`有一个关联类型叫`Output`，是`add`方法的返回类型。下面代码中的默认泛型类型定义在`Add`特质中

```rust
trait Add<Rhs=Self> {
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}
```

`Rhs=Self`这个语法叫做**默认类型参数**（*default type parameters*），`Rhs`泛型类型参数定义了`add`方法的`rhs`参数的类型，如果我们不指定`Rhs`的具体类型，那么默认为`Self`。当我们为`Point`实现`Add`时，我们使用了默认类型，因为我们相加的是两个`Point`实例。下面的代码中有两个结构体`Millimeters`和`Meters`，表示不同的单位。这种将已有类型精简包裹到另一个结构体叫做**新类型模式**（*newtype pattern*）。我们通过实现`Add`来将米的值加到毫米上

```rust
use std::ops::Add;

struct Millimeters(u32);
struct Meters(u32);

impl Add<Meters> for Millimeters {
    type Output = Millimeters;

    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + (other.0 * 1000))
    }
}
```

有两种使用默认类型参数的主要场景

1. 在不破坏已有代码的情况下扩展类型
2. 大多数用户不需要的特殊情况下的定制化

对于标准库中`Add`特质，它属于第二种场景。第一个目的和第二个相似但是相反：如果你想为已有的特质添加一个类型参数，你可以给它提供一个默认值来扩展特质的功能，并没有破坏已有的实现代码

### 消除同名方法的不明确性

Rust中允许两个特质有同名方法，Rust也不会阻止你在同一个类型上实现这两种特质。在一个类型上直接实现和这个类型实现的某个特质中的方法同名的方法也是允许的。但是在调用同名方法时，你需要告诉Rust你想调用的是哪个方法

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

impl Human {
    fn fly(&self) {
        println!("*waving arms furiously*");
    }
}
```

当我们调用`Human`的`fly`方法时，编译器默认调用直接类型自身的方法

```rust
fn main() {
    let person = Human;
    person.fly(); // *waving arms furiously*
}
```

要调用从`Pilot`或者`Wizard`特质实现的方法，我们需要显式语法来指定具体的`fly`方法

```rust
fn main() {
    let person = Human;
    Pilot::fly(&person);
    Wizard::fly(&person);
    person.fly();
}
```

因为`fly`方法的参数是`self`，如果有一个类型实现了两个特质，Rust可以基于`self`的类型来判断使用哪个特质的实现。但是关联函数没有`self`参数，当有多个类型或者特质定义了和函数名同名的关联函数，Rust不知道你想调用哪个函数，这时你需要使用**全称语法**（*fully qualified syntax*）

```rust
trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Dog {
    fn baby_name() -> String {
        String::from("Spot")
    }
}

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn main() {
    println!("A baby dog is called a {}", Dog::baby_name()); // A baby dog is called a Spot
}
```

我们想要调用`Animal`特质中为`Dog`实现的方法。因为`Animal::baby_name`没有`self`参数，可能也有其它类型实现了`Animal`特质，Rust无法找到我们想要调用的具体函数。要清除这种不确定性，我们需要使用全称语法

```rust
fn main() {
    println!("A baby dog is called a {}", <Dog as Animal>::baby_name());
}
```

全称语法的定义是`<Type as Trait>::function(receiver_if_method, next_arg, ...);`对于那些不是方法的关联函数，没有`receiver`，只有参数列表。你可以在任何调用函数或方法的地方使用全称语法，但是当Rust可以从其它信息区分类型的时候可以省略大部分内容而只用函数名称和参数

### 使用父特质

有时你会写一个依赖其它特质的特质定义，表示当一个类型要实现这个特质时，它也需要实现第二个特质。这样做可以在特质定义中利用第二个特质的关联项。这个被依赖的特质称为特质的**父特质**（*supertraits*）。例如我们要实现边框特效`OutlinePrint`特质，它需要使用`Display`特质的功能，所以我们的`OutlinePrint`特质只能作用于实现了`Display`的特质的类型

```rust
use std::fmt;

trait OutlinePrint: fmt::Display {
    fn outline_print(&self) {
        let output = self.to_string();
        let len = output.len();
        println!("{}", "*".repeat(len + 4));
        println!("*{}*", " ".repeat(len + 2));
        println!("* {output} *");
        println!("*{}*", " ".repeat(len + 2));
        println!("{}", "*".repeat(len + 4));
    }
}
```

因为我们指定了`OutlinePrint`需要`Display`特质，我们可以调用`to_string`方法

### 使用新类型模式来在外部类型上实现外部特质

第十章中我们提到了孤儿规则，只有当特质或类型两者之一或者两者都在当前crate中，才能为该类型实现特质。使用新类型模式可以突破这个限制，它使用元组结构体中创建一个新类型。这个元组结构体只有一个成员，是我们要实现特质的类型的精简包裹。**新类型**（*Newtype*）是一个来自Haskell编程语言的术语。使用这个模式并没有运行时性能开销，包裹类型在编译期会被忽略

```rust
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    println!("w = {w}");
}
```

使用这个技术的缺点是`Wrapper`作为新类型，没有原类型定义的函数和方法。我们可以在新类型上实现所有原类型的方法。如果我们想让新类型拥有所有内部类型的方法，在`Wrapper`上实现`Deref`特质，返回内部类型是一种解决方案，就像把这个包裹器当成指针，如果我们不想让Wrapper类型拥有所有内部类型的方法，就必须手动实现我们想要的方法

## 高级类型

### 为了类型安全和抽象使用新类型模式

新类型模式的使用场景除了静态地强制值不搞混和表示值的单位以外还有很多。我们也可以使用新类型模式将一些类型的实现细节抽象出来：新类型可以暴露公共API，和内部类型的API不同。新类型也可以隐藏内部实现。例如`People`类型包裹一个`HashMap<i32, String>`，使用`People`的代码只需要和它暴露的公共API交互，而不需要知道内部的数据结构是什么。新类型模式是一个轻量级的隐藏封装细节的方式

### 使用类型别名创建类型同义词

Rust可以声明一个**类型别名**（*type alias*），为已有类型创建别名。`Kilometers`不是一个单独的新类型，它和`i32`是一样的

```rust
type Kilometers = i32;

let x: i32 = 5;
let y: Kilometers = 5;

println!("x + y = {}", x + y);
```

使用这种方式会让我们丧失类型检查的优势，就像使用新类型模式中那样。类型同义词主要用于减少重复代码，例如我们可能有一个很长的类型`Box<dyn Fn() + Send + 'static>`。在函数签名中写这个类型的全称很麻烦且容易出错。类型别名通过减少重复让代码更好管理

```rust
type Thunk = Box<dyn Fn() + Send + 'static>;

let f: Thunk = Box::new(|| println!("hi"));

fn takes_long_type(f: Thunk) {
    // --snip--
}

fn returns_long_type() -> Thunk {
    // --snip--
}
```

这样的代码更易于读写。为一个类型别名选择有意义的名称可以更清晰地展示代码的目的。类型别名也常用于`Result<T, E>`类型来减少重复。例如标准库的`std::io`模块。I/O操作在失败时经常返回`std::io::Error`类型表示所有可能的I/O错误。很多函数在`E`是`std::io::Error`时都会返回`Result<T, E>`，例如

```rust
use std::fmt;
use std::io::Error;

pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize, Error>;
    fn flush(&mut self) -> Result<(), Error>;

    fn write_all(&mut self, buf: &[u8]) -> Result<(), Error>;
    fn write_fmt(&mut self, fmt: fmt::Arguments) -> Result<(), Error>;
}
```

因为`Result<..., Error>`出现次数很多，所以`std::io`有一个类型别名`type Result<T> = std::result::Result<T, std::io::Error>;`这个类型别名有两个优点：让代码更好写，也给了我们访问`std::io`模块中API的统一接口。因为它只是一个别名，所以我们可以使用任何`Result<T, E>`上的方法，和`?`语法

### 永远不会返回的`Never`类型

Rust有一个特殊类型叫做`!`，在类型理论术语中被称为**空类型**，因为它没有值。我们更愿意叫它**从不类型**（*never type*），当一个函数永远不返回时用它占据返回类型的位置

```rust
fn bar() -> ! {
    // --snip--
}
```

永远不返回的函数称为**发散函数**（*diverging functions*）我们不能创建`!`类型的值，所以`bar`永远不会返回

```rust
let guess: u32 = match guess.trim().parse() {
    Ok(num) => num,
    Err(*) => continue,
};
```

之前我们提到`match`分支必须返回相同的类型，所以下面的代码不能通过编译

```rust
let guess = match guess.trim().parse() {
    Ok(*) => 5,
    Err(_) => "hello",
};
```

但是为什么在一个分支返回`u32`，另一个分支以`continue`结束就没问题？你可能会想到，`continue`返回一个`!`类型，当Rust计算`guess`的类型时，它会检查所有分支，之前的是`u32`，之后的是`!`，因为`!`没有值，Rust将`guess`的类型确定为`u32`。`!`类型的表达式可以被转换成任何其它类型。我们可以以continue结束分支是因为`continue`不会返回值，它会返回循环头，所以在`Err`分支中，我们没有给guess分配值

### `Never`类型在`panic!`宏中也可以使用

```rust
impl<T> Option<T> {
    pub fn unwrap(self) -> T {
        match self {
            Some(val) => val,
            None => panic!("called `Option::unwrap()` on a `None` value"),
        }
    }
}
```

最后一个返回类型`!`的表达式是`loop`

```rust
print!("forever ");

loop {
    print!("and ever ");
}
```

如果引入`break`，表达式的返回值类型就不是`!`了

### 动态大小类型和`Sized`特质

Rust需要知道类型的很多细节，例如一个特定类型的值需要分配多少空间。这对于**动态大小类型**（*dynamically sized types*）有一些问题。它们也被称为*DST*或者*unsized types*。这些类型可以让我们使用只有在运行时才知道大小的类型的代码。例如`str`，它本身是一个*DST*。只有在运行时才知道它的值的大小，也就是说我们不能创建一个`str`类型的变量，我们也不能接收类型为`str`的参数。Rust需要知道某个类型的所有可能的值需要分配的空间大小，一个类型的所有可能的值必须使用相同数量的内存。尽管`&T`存储了`T`类型的值的内存地址，`&str`实际上是两个值：`str`的地址和长度。我们可以在编译期知道`&str`的大小：一个`usize`的两倍。通常就这就是动态大小类型在Rust中的工作方式：它们包含额外的元数据来保存动态类型的信息。使用动态大小类型的黄金法则是我们必须总是将它们的值放到某种指针后面。我们可以将`str`和任何类型的指针绑定起来，例如`Box<str>`或者`Rc<str>`。所有特质都是动态大小类型，我们可以通过使用特性的名称来使用。之前我们提到当作为特质对象使用特质时，我们必须将它们放到指针后，例如`&dyn Trait`或者`Box<dyn Trait>`（或`Rc<dyn Trait>`）。为了使用*DST*，Rust提供了`Sized`特质来决定一个类型的大小是否在编译期可知。这个特质自动在所有编译期类型大小可知的类型上实现。此外，Rust为所有泛型函数隐式增加了Sized限制，即

```rust
fn generic<T>(t: T) {
    // --snip--
}

// 实际上是下面的形式
fn generic<T: Sized>(t: T) {
    // --snip--
}
```

默认情况下，泛型参数只能指定那些编译期大小已知的类型，但是你可以使用以下特殊语法来放宽限制

```rust
fn generic<T: ?Sized>(t: &T) {
    // --snip--
}
```

`?Sized`的特质限制表示“`T`可能是也可能不是`Sized`”，这个概念会覆盖默认的泛型类型的大小必须在编译期已知的限制。`?Trait`语法只作用于`Sized`，其它特质不能这样写。注意我们将`t`参数的类型从`T`转变为`&T`，因为类型可能不是`Sized`，我们需要将它放到指针后，这里我们选择使用引用

## 高级函数和闭包

### 函数指针

你可以将普通函数作为参数传入其它函数。当你想将已经定义的函数传入函数而不使用闭包时，可以使用这个技术。函数会转为类型`fn(f)`，需要和`Fn`闭包特质区分开。`fn`类型叫做**函数指针**（*function pointer*）使用函数指针将函数传入可以让你将函数作为其它函数的参数

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

fn main() {
    let answer = do_twice(add_one, 5);

    println!("The answer is: {answer}");
}
```

和闭包不同，`fn`是类型而不是特质，所以我们直接指定`fn`作为参数类型而不是声明一个泛型类型参数和一个`Fn`特质作为特质限制。函数指针类型实现了所有三种闭包特质（`Fn`、`FnMut`和`FnOnce`），表示你总是可以向需要闭包参数的函数传入一个函数。最好写那种使用泛型类型结合闭包特质的函数，这样你的函数可以同时接收函数和闭包参数。一个你只想接收`fn`而不是闭包的例子是和那些没有闭包功能的外部代码交互：C的函数可以接收函数作为参数，但是C没有闭包

```rust
let list_of_numbers = vec![1, 2, 3];
let list_of_strings: Vec<String> =
    list_of_numbers.iter().map(|i| i.to_string()).collect();
let list_of_numbers = vec![1, 2, 3];
let list_of_strings: Vec<String> =
    list_of_numbers.iter().map(ToString::to_string).collect();
```

我们必须使用全称语法，因为`to_string`有很多同名函数。这里我们使用`ToString`特质中的`to_string`函数，标准库对任何实现了`Display`的类型实现了这个特质。考虑一下枚举类型，每个变体的名称都是一个初始化函数，我们可以使用这些初始化函数作为实现了闭包特质的函数指针，也就是说我们可以将初始化函数作为参数传入接受闭包参数的方法中

```rust
enum Status {
    Value(u32),
    Stop,
}

let list_of_statuses: Vec<Status> = (0u32..20).map(Status::Value).collect();
```

有些人更喜欢这种形式，有些人更喜欢使用闭包，但它们会编译成相同的代码，所以选择自己喜欢的方式即可

### 返回闭包

闭包可以通过特质表示，这意味着你不能直接返回闭包。在多数你想返回一个特质的情况下，你可以使用实现了这个特质的具体类型作为函数的返回值。但是闭包不支持这种语法，因为没有具体的可以被返回的类型，如果闭包从它的域中捕获了任何内容，你也不能使用函数指针`fn`作为返回类型。你可以使用`impl Trait`语法，它可以返回任何闭包类型，使用`Fn`、`FnOnce`和`FnMut`

```rust
fn returns_closure() -> impl Fn(i32) -> i32 {
    |x| x + 1
}
```

就像之前说过的，每个闭包也有它自己单独的类型，如果你要使用相同签名但是实现不同的多个函数，你需要利用特质对象

```rust
fn main() {
    let handlers = vec![returns_closure(), returns_initialized_closure(123)];
    for handler in handlers {
        let output = handler(5);
        println!("{output}");
    }
}

fn returns_closure() -> impl Fn(i32) -> i32 {
    |x| x + 1
}

fn returns_initialized_closure(init: i32) -> impl Fn(i32) -> i32 {
    move |x| x + init
}
```

从上面代码的编译错误信息中可以看到，当我们返回`impl Trait`时，Rust为闭包创建了唯一的**模糊类型**（*opaque type*）我们没办法看到Rust为我们构建的类型细节。所以即使这些函数返回的都是实现了相同特质的闭包`Fn(i32) -> i32`，Rust生成的模糊类型都是唯一的

```rust
fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}

fn returns_initialized_closure(init: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x + init)
}
```

上面的代码可以正常编译

## 宏

**宏**（*macro*）表示Rust提供的特性：`macro_rules!`**声明宏**（*declarative*）和三种**过程宏**（*procedural*）

- 自定义`#[derive]`宏：为使用`derive`属性的结构体和枚举上生成代码
- 类属性宏：定义在任意项上的自定义属性
- 类函数宏：和函数调用类似，但是操作的内容是作为参数传入的token

### 宏和函数之间的区别

简单来说，宏是一种写生成其它代码的代码的方式，也叫做**元编程**（*metaprogramming*）。例如`derive`属性，它会为你的结构体生成很多特质的实现，我们也使用了`println!`和`vec!`宏。这些宏都将程序扩展为更多的代码，这些代码原本需要手写。元编程减少了那些需要手写和人工维护的代码数量，这也是函数扮演的角色之一。但是宏有一些函数不具备的能力。一个函数的签名必须声明参数的数量和类型。宏可以接受不同数量的参数，另外，宏在编译器解释代码之前扩展代码，所以宏可以为某个类型实现特质，而函数不可以，因为函数在运行时被调用，一个特质需要在编译期被实现。写宏的缺点是宏的定义比函数定义更复杂，因为写的是生成Rust代码的代码，由于缺少了直接联系，宏的定义更难读、理解和维护。另一个宏和函数之间重要的区别是你必须在文件中调用它们之前定义宏并将它们引入域中，而函数可以在任何地方定义和调用

### 用于通用的元编程的声明宏`macro_rules!`

用途最广的宏的形式是声明宏。有时也叫做“macros by example”、“macro_rules!宏”或者就叫“宏”。声明宏的核心是让你写一些类似Rust的`match`表达式类似的代码。宏也会比较值和关联特定代码的模式：这种情况下，被比较的值是传入宏的Rust源代码的字面值；模式和源代码的结构相比较，当匹配后，模式关联的代码会替换掉作为宏参数的代码，这都是在编译期发生的。定义一个宏需要使用`macro_rules!`结构。我们通过查看`vec!`宏的定义来研究用法。我们可以用`vec!`宏构建包括一个、两个整数或者五个字符串的vector。但我们不能使用函数这样做，因为我们无法提前知道传入值的数量和类型

```rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),*) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

注意：实际上`vec!`宏在标准库的定义中包含提前分配正确数量内存的代码。这里的代码示例只是为了让例子更简单

`#[macro_export]`的作用是当前crate定义的宏可以被导出，其它使用这个crate的代码可以使用这个宏。如果没有这个注解，宏无法被带入域中。声明宏可以在当前crate中定义并使用。使用`macro_rules!`作为宏定义的开头，然后是宏的名称，后面不需要加感叹号，之后是包含宏定义的花括号。`vec!`的内容和`match`表达式类似，这里有一个分支模式`( $( $x:expr ),* )`，后面是`=>`和这个模式关联的代码，如果模式匹配，关联的代码会替换原始代码。因为这个宏只有一个模式，所以只有一种有效的匹配方式，任何不匹配的值都会导致错误，更复杂的宏会有多于一个的分支

宏定义的模式语法和十九章介绍的模式语法不一样，因为宏的模式匹配的是Rust代码结构而不是某种类型的值。首先我们使用括号来包含整个模式，使用`$`在宏系统中声明一个变量，包含匹配这个模式的Rust代码。美元符号可以和普通的Rust变量区分开，然后又是一个括号来捕获与括号内模式匹配的值，以在替换代码中使用。在`$()`中的是`$x:expr`，它匹配任何Rust表达式，并且给这个表达式命名为`$x`。后面的逗号表示一个字面值逗号分隔符，它必须在匹配的`$()`代码之间出现，`*`指定之前的模式需要匹配0次或多次。当我们调用`vec![1, 2, 3];`时，`$x`模式匹配了三次，包含`123`。分支对应的代码中，`$()*`中的`temp_vec.push()`为每个匹配了`$()`中的模式`0`次或多次的部分生成代码。当我们调用这个宏`vec![1, 2, 3];`会生成的类似下面的代码

```rust
{
    let mut temp_vec = Vec::new();
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
}
```

这个宏可以接收任意数量的任意相同类型的参数，生成创建包含这些元素的vector的代码

### 从属性生成代码的过程宏

宏的第二种形式是过程宏，它和函数（过程的一种类型）更像，过程宏接受一些代码作为输入，然后操作这些代码，再创建一些代码作为输出，不是使用匹配模式的方式进行代码替换。有三种类型的过程宏，自定义`derive`、类属性、类函数，这些工作过程都是一样的。但是在创建过程宏时，它们必须定义在自己的crate中，并且有特殊的crate类型。这样做目前有复杂的技术原因（之后会去除这个限制）

```rust
use proc_macro;

#[some_attribute]
pub fn some_name(input: TokenStream) -> TokenStream {
}
```

这个函数定义了一个过程宏，接受`TokenStream`作为参数，创建`TokenStream`作为返回值。`TokenStream`类型定义在`proc_macro`crate中，这个crate包含在Rust中，表示一系列的token。这是宏的核心：宏作用于的源代码组成了输入的`TokenStream`，宏产生的代码是输出`TokenStream`。这个函数有一个属性，指定了我们创建的过程宏是哪种类型

#### 如何写自定义的`derive`宏

先创建一个crate叫做`hello_macro`，定义一个特质`HelloMacro`，定义关联函数`hello_macro`。我们不需要让用户自己为他们的类型手动实现这个特质，我们提供一个过程宏，用户可以使用`#[derive(HelloMacro)]`的方式注解他们的类型，得到`hello_macro`函数的默认实现。这个默认实现会打印`Hello, Macro! My name is {TypeName}!`，这里的`TypeName`是这个实现特质的类型

```rust
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
```

首先创建一个库crate`$ cargo new hello_macro --lib`，然后定义`HelloMacro`特质和它的关联函数

```rust
pub trait HelloMacro {
    fn hello_macro();
}
```

crate的用户可以实现这个特质来得到想要的功能

```rust
use hello_macro::HelloMacro;

struct Pancakes;

impl HelloMacro for Pancakes {
    fn hello_macro() {
        println!("Hello, Macro! My name is Pancakes!");
    }
}

fn main() {
    Pancakes::hello_macro();
}
```

我们不需要用户为自己的所有类型都实现这个特质，而且我们也不能使用特质方法的默认实现，因为不能提前确定实现特质的类型名称。Rust没有反射功能，所以无法在运行时查看类型名称，我们需要在编译期生成代码。下一步是定义一个过程宏。因为过程宏需要在它们单独的crate中。构造crate和宏crate的习惯是，对于`foo`crate，一个自定义`derive`过程宏crate叫做`foo_derive`，创建一个`hello_macro_derive`crate：`$ cargo new hello_macro_derive --lib`。我们的两个crate联系紧密，所以我们在`hello_macro`crate目录下创建这个过程宏crate。如果我们改变了`hello_macro`中特质的定义，我们也会修改`hello_macro_derive`中的过程宏定义。两个crate可能会单独发布，使用这些crate需要将它们加入依赖，并且将它们都带入域中。我们也可以让`hello_macro`使用`hello_macro_derive`作为依赖，然后重导出过程宏。然而现在构建项目的方式可以让用户在不想使用`derive`功能的情况下单独使用`hello_macro`。我们需要声明`hello_macro_derive`crate是过程宏crate，我们也需要`syn`和`quote`crate的功能，所以在`hello_macro_derive`的*Cargo.toml*内容如下

```toml
[lib]
proc-macro = true

[dependencies]
syn = "2.0"
quote = "1.0"
```

要开始定义过程宏，将下面的代码放到`hello_macro_derive`crate的`src/lib.rs`文件中

```rust
use proc_macro::TokenStream;
use quote::quote;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // Construct a representation of Rust code as a syntax tree
    // that we can manipulate.
    let ast = syn::parse(input).unwrap();

    // Build the trait implementation.
    impl_hello_macro(&ast)
}
```

我们将`hello_macro_derive`函数中的代码进行划分，它负责解析`TokenStream`，`impl_hello_macro`函数负责转换语法树：这种方式来写过程宏更方便。外部函数（*hello_macro_derive*）的代码和所有你见过的或创建的过程宏crate是一样的。但内部函数的代码会根据过程宏的目的不同而不同

我们引入了三个crate，`proc_macro`，`syn`和`quote`。`proc_macro`crate是Rust自带的，我们不需要将其添加到*Cargo.toml*的依赖中，`proc_macro`crate是编译器的API，让我们可以从代码中读取和操作Rust代码。`syn`crate可以解析Rust代码字符串，生成可以操作的数据结构。`quote`crate将`syn`生成的数据结构转换为Rust代码。这些crate让转换Rust代码的过程更简单：因为写一个Rust代码的完整转换器并不简单

`hello_macro_derive`函数当我们的库用户为类型指定`#[derive(HelloMacro)]`时被调用。因为我们在`hello_macro_derive`函数上注解了`proc_macro_derive`，并且指定了名称`HelloMacro`，它会匹配了我们的特质名称，这是大多数过程宏遵循的惯例。`hello_macro_derive`函数首先将`input`从`TokenStream`转换成我们可以理解和操作的数据结构。这是`syn`的作用。`parse`函数接受`TokenStream`为参数，返回`DeriveInput`结构，表示转换后的Rust代码，以下是部分转换代码的结构

```rust
DeriveInput {
    // --snip--

    ident: Ident {
        ident: "Pancakes",
        span: #0 bytes(95..103)
    },
    data: Struct(
        DataStruct {
            struct_token: Struct,
            fields: Unit,
            semi_token: Some(
                Semi
            )
        }
    )
}
```

这个结构体的成员显示，我们转换的Rust代码是一个单元结构体，`iden`（identifier）是`Pancackes`。还有一些成员说明了Rust代码的各个部分，需要查看`syn`的文档。在定义`impl_hello_macro`函数前，要注意`derive`宏的输出也是`TokenStream`。返回的`TokenStream`被添加进我们的crate用户写的代码中，编译crate时，它们会得到我们修改的`TokenStream`提供的额外功能。你可以注意到我们调用了`unwrap`，如果`syn::parse`函数调用失败就会崩溃。对于我们的过程宏来说，遇到错误崩溃是必须的，为了满足过程宏的API，`proc_macro_derive`函数必须返回`TokenStream`而不能是`Result`类型，。我们使用`unwrap`简化了这个例子，在生产代码中，你应该通过`panic!`或`expect`打印更详细的错误信息

```rust
fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let generated = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    generated.into()
}
```

使用`ast.ident`获取被注解类型的名称。使用`quote!`宏定义我们要返回的Rust代码，编译器期望的结果和`quote!`宏执行的直接结果类型不同，所以我们需要将其转换为`TokenStream`，所以调用`into`方法，它消费了中间表示层的内容，返回`TokenStream`类型的值。`quote!`宏也提供了很酷的模版机制，我们可以输入`#name`，`quote!`会用`name`变量的值替换它，你甚至可以做一些普通宏可以做的重复工作，需要查看`quote`的文档。我们想让过程宏对任何用户注解的类型生成`HelloMacro`特质的实现，通过`#name`得到类型名后，特质实现有一个函数`hello_macro`，它的函数体包含我们要提供的功能，打印`Hello, Macro! My name is`，后面是被注解类型的名称。`stringify!`宏是Rust内置的，它接收一个Rust表达式，在编译期将其转换为字符串字面值，例如`1 + 2`转换成`"1 + 2"`，它和`format!`或`println!`不同，那些宏会计算表达式，然后将其转换为`String`。`#name`输入可能是一个要打印的表达式，所以使用`stringify!`，它也节省了将`#name`在编译期转换为字符串字面值所需要分配的资源。此时执行`cargo build`，`hello_macro`和`hello_macro_derive`编译成功。然后创建一个新的二进制crate，将`hello_macro`和`hello_macro_derive`放到*Cargo.toml*的依赖中，因为是本地依赖，可以使用`path`

```toml
hello_macro = { path = "../hello_macro" }
hello_macro_derive = { path = "../hello_macro/hello_macro_derive" }
```

#### 类属性宏

类属性宏和自定义`derive`宏类似，它不是为`derive`属性生成代码，而是允许你创建新属性。它们更灵活，属性可以应用于其他项，例如函数，而`derive`只能应用于结构体和枚举上

```rust
#[route(GET, "/")]
fn index() {
```

`#[route]`属性可能是过程宏的定义，它的定义函数的签名就像下面的代码

```rust
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
```

这里有两个参数，第一个是属性的内容，`GET, "/"`的部分，第二个是属性作用于的项本身，本例中是`fn index() {}`和函数体。除了这些，类属性宏的工作方式和自定义`derive`宏一样：你创建一个crate，类型是`proc-macro`，实现一个生成代码的函数

#### 类函数宏

类函数宏定义了那些类似函数调用的宏，和`macro_rules!`宏类似，但它们比函数更灵活，例如他们可以接受未知个数的参数。`macro_rules!`宏只能使用类似`match`语法的方式定义。但是类函数宏接收`TokenStream`作为参数，在它们的定义中可以使用Rust代码处理`TokenStream`，就像另外两种过程宏一样。`let sql = sql!(SELECT * FROM posts WHERE id=1);`，这个宏会解析SQL语句，检查它的语法是否正确，这种宏能实现比`macro_rules!`更复杂的功能。`sql!`宏的定义可能像这样

```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
```

这个宏的定义和自定义的`derive`宏的签名类似：使用输入的`TokenStream`，返回我们想生成的代码
