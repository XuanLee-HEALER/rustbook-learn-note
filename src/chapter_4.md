# 第四章 - 理解所有权

Ownership（所有权）是决定Rust如何管理内存的一系列规则。编译器检查所有权系统的规则来管理内存，如果代码不符合规则，则程序无法通过编译。程序运行期间（运行时），所有权相关的特性**不会影响程序性能**

> **堆和栈（The Stack and the Heap）**
>
> 在像Rust这样的系统编程语言中，值存储在栈上还是堆上会影响语言的行为，也是你必须做的决定。栈和堆都是代码在运行时会使用的内存的一部分，但是它们的组织方式不同。栈是LIFO（后进先出）方式来存储值，添加数据叫做*pushing onto the stack*，移除数据叫做*popping off the stack*。所有在栈上的数据必须有**已知、固定的大小**。编译期未知大小的数据或者大小可能改变的数据都必须存储在堆上
>
> 堆的组织性更弱，当你将数据放到堆上，你需要请求一定数量的堆内存空间。内存分配器在堆上找到足够大的空间，将它标记为在使用，然后返回一个**指针**（*pointer*），它指向这块内存的地址。这个过程叫做*allocating on the heap*，有时简写为*allocating*。因为指向堆的指针大小是已知、固定的，所以可以将指针自身存储在栈中，但是当你需要真正的数据时，你需要依靠指针来访问
>
> 将值存到栈中比在堆上分配更快，因为内存分配器不需要寻找空间来存储新数据，这个位置总是栈的顶部，但是在堆上分配空间需要做很多工作，包括找到足够大的空间然后做好记录、为下一次分配做准备。访问堆上的数据也比访问栈上的数据更慢，因为必须通过指针找到具体位置。在现代处理器环境下，如果在内存间跳转次数更少，执行速度就更快。举个例子，如果处理器是饭店的服务员，他们要收集所有桌子的订单，一定是在一桌收集完成后继续收集下一桌，比每桌收一张这种大量时间浪费在不同桌子间的移动上的选择更好。而且如果处理器处理的数据和其它的数据更近（栈）那么它们完成工作会更快（**空间局部性原理**）
>
> 当你的代码调用一个函数，传入函数的值（可能包括指向堆数据的指针）和函数的局部变量都会插入到栈中，当函数结束，这些值被弹出栈

检查正在使用堆上数据的代码，最小化堆上的数据拷贝，清除堆上的无用数据以至于你不会用完所有的堆空间，这都是所有权要处理的问题

## 所有权的规则

- Rust中每个值都有一个owner（所有者）
- 同一时间一个值只能有一个所有者
- 当所有者离开域之后，它所拥有的值会被清理

### 变量域

一个**域**（*scope*）是程序中的一个范围，在其中的项是有效的

```rust
{                      // s is not valid here, it’s not yet declared
    let s = "hello";   // s is valid from this point forward

    // do stuff with s
}                      // this scope is now over, and s is no longer valid
```

上面的代码中有两个重要的时间点

- 当s进入域的时候，它是有效的
- 在s离开域之前，它是有效的

### `String`类型

上一章提到的数据类型都是已知大小的，可以被存储在栈中，当它们离开域后会被从栈中弹出，如果其它代码要在不同的域中使用这种类型的相同值的不同实例，可以快速且轻松地创建一个新的、独立的实例

我们已经见过的字符串字面值，它们是硬编码在我们的程序中。虽然使用它们很方便，但是并不是在所有我们想使用文本的地方用它们都适合。一个原因是因为它们是**不可变的**，另一个原因是不是所有的字符串的内容当我们在写代码的时候都提前知道。对于这些情况，Rust提供了`String`类型，它管理在堆上分配的数据，可以存储编译期我们不知道大小的文本内容

```rust
// 从字符串字面值创建一个String
let s = String::from("hello");
```

`::`操作符让我们可以将`from`方法组织到`String`类型的命名空间下，而不用单独提供一个函数名称。这种`String`类型的值可以被修改（需要声明为`mut`）

### 内存和分配

对于字符串字面值，我们在编译期知道内容的大小，所以文本就直接硬编码到了最后的可执行程序中，这就是为什么字符串字面值性能高效的原因。但是这些属性只来源于字符串字面值的不可变性质

为了支持一个**可变的、可增长**的编译期不知道文本内容的字符串，我们需要在堆上分配一定数量的内存来保存这些内容，这意味着

- 内存必须在运行时从内存分配器请求
- 我们需要一种机制，当使用完`String`的值之后将内存还给分配器

第一部分，我们通过`String::from`实现，这个函数的实现就是请求字符串需要的内存（在编程语言中很通用）。第二部分，对于有GC的语言，GC会跟踪并清理不再使用的内存。在大部分没有GC的语言中，我们需要在内存不再使用时手动清理掉它们，要做对这件事情非常困难，如果忘记清理，就会浪费内存，如果提前清理，我们会得到保存无效值的变量，如果清理同一块内存两次也会触发bug，我们必须保证一次*allocate*只对应一次*free*

Rust的做法是：一旦持有内存的变量离开它的域后内存会自动返还

```rust
{
    let s = String::from("hello"); // s is valid from this point forward

    // do stuff with s
}                                  // this scope is now over, and s is no
                                   // longer valid
```

当`s`离开域后（花括号结束），Rust调用了特殊的函数`drop`，`String`类型的实现者可以将返还内存的代码放到这个函数中。Rust会在程序执行到花括号结束后会自动调用`s`的`drop`方法

> C++中，这种在项的生命周期结束时返回资源的模式叫做*Resource Acquisition Is Initialization(RAII)*
这个模式对于Rust的内存管理方式有很深远的影响

### 与移动（*Move*）操作交互的变量和数据

在Rust中多个变量可以以不同的方式交换相同的数据

```rust
let x = 5;
let y = x;
```

因为整数是非常简单的值，有已知的固定大小，所以这两个5都会被插入栈中，即`x`和`y`都可以使用

```rust
let s1 = String::from("hello");
let s2 = s1;
// s2拷贝的是s1的指针/len/cap，而非堆上的数据
```

一个`String`由三部分组成

- `ptr`：指向存储数据的内存位置的指针
- `len`：字符串当前长度
- `cap`：字符串的容量

这三部分数据都存储在栈上。而字符串的具体内容`"hello"`存放在堆上。长度是字符串根据内容量向分配器申请了多少字节的内存。当我们将`s1`赋值给`s2`，`String`本身的数据被拷贝了，但是指针指向的在堆中的数据没有被拷贝。之前说到Rust在变量离开域后会自动调用变量的`drop`函数并且清理变量使用过的堆内存，但是现在两个`String`指向同一块内存。*double free*错误也是内存安全的bug之一。释放两次内存会导致内存崩溃，有潜在的安全隐患

为了保证内存安全性，在`let s2 = s1;`中Rust认为`s1`不再有效。因此当`s1`离开域后Rust不会释放任何东西。其它语言有*shallow copy*和*deep copy*的概念，这种不拷贝堆上数据的方式就是浅拷贝。但是因为Rust直接让第一个变量失效，所以这种行为叫做*move*。并且这里有一个设计选择：Rust永远不会自动创建数据的*deep copy*，因此任何automatic（自行）拷贝都可以被看作运行时中代价很低的操作

#### 域和赋值

移动方式在域、所有权和通过`drop`释放的内存之间的关系也是成立的。当你给一个已有变量赋值了一个全新的值，Rust会立即调用原始值的drop，像下面的代码展示的那样

```rust
let mut s = String::from("hello");
s = String::from("ahoy");

println!("{s}, world!");
```

当我们将新的`String`绑定到`s`上时，堆上原有的值已经没有东西来引用，因此原有的字符串直接离开了它的域

### 与克隆（*Clone*）操作交互的变量和数据

如果我们确实想对`String`引用的堆上的数据进行深拷贝，我们可以使用`clone`方法，当你看到了`clone`方法的调用，那就说明有一些额外的代码会被执行，并且这部分代码的运行时代价很高

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

// s1 和 s2 都有效
println!("s1 = {s1}, s2 = {s2}");
```

对于栈上分配的数据：`Copy`

```rust
let x = 5;
let y = x;

println!("x = {x}, y = {y}");
```

这段代码中，我们没有调用`clone`，但是`x`仍然有效，并没有被移动。原因是像整数这种编译期已知大小的基本类型是存储在栈上的，所以拷贝它们非常快，这就代表我们不需要在创建y之后让x的值失效。即对于这种类型，深拷贝还是浅拷贝没有区别，是否调用`clone`也没有任何区别

Rust有一个特殊注解是`Copy`特质，我们可以将它放到那些存储在栈上的类型。如果一个类型实现了`Copy`特质，那么使用它的变量给其它变量赋值时不会移动，而是直接拷贝，并且在赋值给其它变量后它自身依然有效。Rust不允许将`Copy`注解到那些本身或者它的一部分实现了`Drop`特质的类型上。如果某种类型在它的值离开域后需要做一些操作，然后我们将`Copy`注解放在这种类型上，我们会得到一个编译期错误

## 所有权和函数

将值传递给函数的机制和将值赋值给一个变量的机制类似。可能会移动或者拷贝，如果类型不能直接拷贝，那么就会发生移动

### 返回值和域

变量的所有权永远遵循相同的模式：将一个值赋给另一个变量会发生移动。当一个包含堆上的数据的变量离开域时，它的值会通过`drop`清理，除非这个数据的所有权被移动给其它变量。每次调用函数都将变量的所有权转移到，然后再返回此变量拿回所有权太麻烦了。我们想让函数使用一个值但是不转移所有权。如果我们之后还想使用转移掉所有权的值就更麻烦，虽然Rust允许我们使用元组来一次返回多个值

### 引用和借用

我们可以提供一个`String`值的引用，一个**引用**（*reference*）就像一个指针，它也是一个地址，我们可以利用这个地址来访问那个位置存储的数据。这个数据被其它的变量所拥有。和指针不同的是，引用在它自己的生命周期中可以保证引用的那些类型的值总是有效的

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{s1}' is {len}.");
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

`&`表示引用，它们允许你在不获取值的所有权的情况下也能使用值。引用操作的反操作是**解引用**（*dereferencing*）使用`*`操作符

因为创建的引用不拥有值，所以当引用不再被使用后，它指向的值也不会被清理。变量`s`的有效域和任何函数参数的域相同，但是引用停止使用后不会清理其引用的值，因为引用没有该值的所有权。当函数用引用当作参数而不是实际的值，我们不需要返回它们来返还所有权，因为我们不曾有过。我们称创建一个引用的行为叫**借用**（*borrowing*）。和变量默认是不可变的一样，我们默认也不能在引用上做任何修改

#### 可变引用

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

**可变引用**（*mutable reference*）有一个限制：如果一个值上存在一个可变引用，那么同一时间不能再对这个值创建其它任何引用。这个限制防止出现同一时间对相同数据存在多个可变引用的情况。这个限制的好处是让Rust可以在编译期来预防**数据竞争**（*data races*），数据竞争是一种**竞争条件**（*race condition*），当以下行为发生时就会出现

1. 同一时间存在两个或多个指针访问同一份数据
2. 至少有一个指针被用来写数据
3. 没有机制来同步对数据的访问

```rust
let mut s = String::from("hello");

{
    let r1 = &mut s;
} // r1 goes out of scope here, so we can make a new reference with no problems.

let r2 = &mut s;
```

我们可以使用花括号来创建一个新域，此时允许存在多个可变引用，但不是同时。Rust对于可变和不可变引用使用相同的规则限制，当我们在同一个值上有不可变引用时也不能创建可变引用。那些不可变引用的用户不希望这些值被突然修改。同时存在多个不可变引用是可以的，因为读操作不会影响其它用户。一个引用的生命周期从它们被创建开始，到它们最后一次被使用结束

#### 悬垂引用

在有指针的语言中，很容易错误地创建出一个**悬垂指针**（*dangling pointer*），它引用了可能已经分配给其他人的内存的地址。Rust的编译器保证引用永远不会是悬垂引用。如果你有一个引用，编译器会保证在引用没有失效前，数据不会离开域（被清理）

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String { // 返回一个String的引用
    let s = String::from("hello"); // s是一个新的String

    &s // 返回s的引用
} // s已经离开域，被drop，它的内存已经被返还
```

上面的代码会报一个错：`this function's return type contains a borrowed value, but there is no value for it to be borrowed from`，表示函数的返回值类型包含一个被借用的值，但是这个值已经不存在了

#### 引用的规则

- 在任意时间，一个值上可以拥有**一个可变引用***或者***任意多个不可变引用**
- **引用总是有效的**

## Slice类型

切片可以让你在一个集合中**引用**一段连续的子序列，而不是整个集合。**切片也是一种引用**，所以它没有集合的所有权

为了解释切片类型引入一个小问题：我们需要写一个函数，接受由空格划分的字符串参数，返回找到的字符串中第一个单词，如果字符串中没有找到空格，那么整个字符串就是一个单词，返回整个字符串长度

```rust
fn first_word(s: &String) -> usize {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return i;
        }
    }

    s.len()
}
```

`first_word`函数接受`&String`类型作为参数。Rust的习惯是函数只要不需要所有权就使用引用作为参数，现在暂时返回第一个空格的下标。现在我们可以找到字符串中第一个单词的结束下标，但是返回的这个下标和`&String`没有任何关系，没办法保证在之后这个下标总是有效，就像下面这样

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s); // word will get the value 5

    s.clear(); // this empties the String, making it equal to ""

    // `word` still has the value `5` here, but `s` no longer has any content
    // that we could meaningfully use with the value `5`, so `word` is now
    // totally invalid!
}
```

在`s.clear()`后，`word`的值实际上已经失效。让`word`的值总是跟`s`的数据保持同步会很麻烦并且非常容易出错，并且如果需要类似的`second_word`这种函数，维护起来就会更痛苦

### 字符串切片

一个字符串切片是`String`的部分的引用

```rust
let s = String::from("hello world");

let hello = &s[0..5];
let world = &s[6..11];
```

我们使用`[starting_index..ending_index]`范围语法来创建切片。在具体的实现中，切片的数据结构存储了起始位置和切片的长度`（ending_index - starting_index）`

对于Rust的`..`语法，如果你想从`0`开始，那么可以省略`0`。如果你的切片包含`String`的最后一个字节，你可以不带末尾的数字。也可以两边的索引位置都不带，那么会创建包含整个字符串的切片。字符串切片的范围索引包含的切片内容必须是有效的UTF-8字符。如果切片分割了某个多字节字符，程序会报错退出

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

我们得到单词的末尾下标，使用这个下标创建一个切片并返回。当我们调用`first_word`时，我们得到了一个绑定内部数据的值，这个值由指向切片的开始点的引用和切片中元素的个数组成。现在这个API更难被破坏，因为编译器会保证String上的引用总是有效，之前的代码虽然有问题但是会通过编译，在运行时会报错。使用切片不会出现这个bug，让我们能更快地看到这个问题

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    s.clear(); // error!

    println!("the first word is: {word}");
}
```

因为`clear`需要清除`String`，它需要得到一个可变引用。而后面的`println!`使用了`word`引用，所以那时不可变引用必须仍然存在。这就违反了借用规则

### 字符串字面值是切片

字符串字面值的类型是`&str`，它指向了二进制程序中的某个位置。这也是为什么字符串字面值是不可变的，因为`&str`是一个不可变引用

### 字符串切片作为参数

`fn first_word(s: &str) -> &str {`，我们可以使用`&String`和`&str`的值调用这个函数。如果我们有一个字符串切片，我们可以直接传进去，如果我们有String，我们可以传入它的切片或者它的引用。这个灵活性利用了deref coercions（解引用强制转换）特性

### 其它切片

可以将类似字符串切片的使用方式应用到所有其它集合上
