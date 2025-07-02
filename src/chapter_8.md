# 第八章 - 常见的集合

## 使用Vector来存储值的列表

`Vec<T>`集合类型，也叫vector（不翻译）。vector允许你将多个值保存到一个数据结构中，这些值在内存中的位置是相邻的。vector只能存储**相同类型的值**

### 创建一个新vector

创建一个空vector`let v: Vec<i32> = Vec::new();`，这里加了类型注解，因为我们还没有向vector中添加任何值，Rust不知道它存储的值的类型是什么。vector使用泛型来实现。当我们创建包含特定类型的vector时，我们可以在尖括号中指定它们的类型。Rust还提供了`vec!`宏，根据你提供的值来创建新的向量`let v = vec![1, 2, 3];`这个向量的类型就是`Vec<i32>`，因为默认的数字类型是`i32`

### 更新一个vector

```rust
let mut v = Vec::new();

v.push(5);
v.push(6);
v.push(7);
v.push(8);
```

和任何变量一样，如果你想修改它的值，就需要使用`mut`关键字让它成为可变的。因为后面加入的都是`i32`的值，所以Rust可以推断出`v`的类型，不需要显式添加类型注解

### 读取vector中的元素

有两种方式来引用存储在向量中的值：使用索引或者`get`方法

```rust
let v = vec![1, 2, 3, 4, 5];

let third: &i32 = &v[2];
println!("The third element is {third}");

let third: Option<&i32> = v.get(2);
match third {
    Some(third) => println!("The third element is {third}"),
    None => println!("There is no third element."),
}
```

使用`&`和`[]`会返回索引位置的元素的引用，使用`get`方法传入索引参数会返回`Option<&T>`类型的值，后面可以用`match`来做进一步处理。Rust对一个元素提供两种查找方式，是为了当你尝试使用超出所有元素长度的索引时，你可以选择程序如何反应。使用`[]`方法会导致程序崩溃，这种方法适合于当你想让你的程序在访问一个位于超出vector长度的索引位置的元素时直接退出（crash）的情况。`get`方法传入一个不合法的索引后它会返回`None`而不会导致程序崩溃。这个参数可能是外部传入的，使用这种方式可以让程序对于用户更加友好

当程序中有一个有效的引用时，借用检查器会使用所有权规则和借用检查来保证这个引用和其它对这个vector的引用是有效的。例如在一个域中不能既有可变引用又有不可变引用

```rust
let mut v = vec![1, 2, 3, 4, 5];

let first = &v[0];

v.push(6); // 这里 first 还需要生效，所以不能使用/创建可变引用

println!("The first element is: {first}");
```

代码中为什么访问第一个元素会被从vector末尾插入的元素影响？因为vector在内存中将值相邻存放，添加一个新元素可能会导致vector分配新内存，然后将老的元素移动到新的内存空间，这种情况下之前的引用会指向已经被释放的内存，这是不被允许的

### 遍历vector中的元素

要按顺序访问vector中的每个元素，我们可以直接遍历所有元素而不是使用索引每次读取一个值

```rust
let v = vec![100, 32, 57];
for i in &v {
    println!("{i}");
}
```

我们也可以在vector的可变引用上遍历每一个元素，在遍历过程中修改元素的值

```rust
let mut v = vec![100, 32, 57];
for i in &mut v {
    *i += 50;
}
```

要改变可变引用的值，我们需要使用`*`解引用操作符来获取`i`的值，之后才能使用`+=`操作符。遍历向量，因为借用检查的存在，无论是可变还是不可变引用，都是安全的。如果我们尝试在`for`循环体中插入或删除项，都会得到一个编译错误，for循环包含的对vector的引用会阻止对整个vector的并发修改

### 使用枚举来存储多种类型

有时我们需要存储不同类型的一组值，枚举的变体是定义在同一个枚举类型下的，所以当我们需要一种类型来表示不同类型的元素时，我们可以使用枚举

```rust
enum SpreadsheetCell {
    Int(i32),
    Float(f64),
    Text(String),
}

let row = vec![
    SpreadsheetCell::Int(3),
    SpreadsheetCell::Text(String::from("blue")),
    SpreadsheetCell::Float(10.12),
];
```

例如我们要存储excel中的单元格，单元格的内容的类型可能是数字、浮点数和字符串，我们就可以按照上面的定义来表示一行的内容。Rust需要在**编译期**知道vector保存的值的类型，这样它才知道堆上应该分配多少内存来存储每个元素。如果Rust允许vector可以包含任意类型，那么可能出现同一个操作对于不同类型的元素会出现错误。使用枚举加上`match`表达式意味着Rust可以在编译期保证所有可能的情况都被处理

如果在编译期不知道程序要处理的向量中元素所有的类型，这种枚举技术也不适用，但是可以使用**特质对象**（*trait object*）

### 删除一个vector就是删除它的元素

当vector值出域后，和其它的结构体一样会被清理。当vector被清理后，它内部的所有元素也被清理。借用检查会保证对vector内容的所有引用只有在vector有效时才能被使用

## 使用字符串来保存UTF-8编码文本

Rust倾向于将潜在错误暴露出来，字符串是非常复杂的数据结构，UTF-8编码字符串的内容之所以在本章讨论，是因为字符串类型是使用字节数组实现的，提供了这些字节表示文本时有用的功能

### 什么是字符串

首先定义**字符串**术语。Rust语言中只有一种字符串类型，那就是字符串切片`str`，是在它的借用形式`&str`中出现的。标准库提供的`String`类型，是一个可增长的、可变的、自有（owned）的UTF-8编码的字符串类型。Rustaceans在讨论字符串时，可能说的是`String`或者`&str`。在标准库中这两种类型都大量使用，并且它们都是UTF-8编码的

#### 创建一个新字符串

`Vec<T>`中的大部分操作在`String`中都存在，因为`String`实际上是字节类型vector的包装器，加上其它的一些保障、限制和额外功能。例如`new`函数都是创建一个实例，`let mut s = String::new();`创建了一个新的、空的字符串实例，之后我们可以向其中加载数据。通常我们有一些初始数据，想使用它们来创建字符串，可以使用它们的`to_string`方法，对于所有实现了`Display`特质的类型都可以使用这个方法。还可以使用`String::from`来从字符串字面值创建`String`

有很多操作字符串的API，其中一些看起来有些多余，但是它们都有各自的作用。`String::from`和`to_string`的作用相同，选择哪个取决于你的代码风格和可读性要求。因为字符串是UTF-8编码，所以我们可以在字符串中包含任何正确编码的文本内容

### 更新一个字符串

一个`String`可以改变大小，它的内容可以改变，就像`Vec<T>`一样，可以插入或删除其中的数据。此外可以使用`+`操作符或者`format!`宏来拼接`String`值

使用`push_str`和`push`来追加字符串

```rust
let mut s = String::from("foo");
s.push_str("bar");
```

`push_str`接收一个字符串切片，因为我们不想获取参数的所有权，之后还要使用它

```rust
let mut s = String::from("lo");
s.push('l');
```

push方法接收单个字符作为参数，并加入到`String`中

使用`+`操作符或者`format!`宏来拼接字符串。如果你想组合两个已存在的字符串，可以使用`+`操作符

```rust
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = s1 + &s2; // note s1 has been moved here and can no longer be used
```

`s1`在追加完字符串后不能使用和使用`s2`的引用的原因是使用`+`操作符时所调用的方法的签名所限制的，类似于`fn add(self, s: &str) -> String {`。标准库中`add`定义使用了泛型和关联类型，这里的示例签名直接使用具体类型。这个签名可以帮助我们理解`+`操作符的复杂之处。首先，`s2`有`&`即我们使用第二个字符串的引用，将内容追加到第一个字符串上。这是因为`s`参数：我们只能将`&str`追加到`String`上，我们不能直接使用`+`拼接两个`String`值。`&s2`的类型是`&String`而不是`&str`仍然可以通过编译，因为编译器进行了**强制转换**（*coerce*）将`&String`转为`&str`，当我们调用`add`方法，Rust使用了解引用强制转换（def coercion）将`&s2`转换为`&s2[..]`。因为`add`没有获取`s`参数的所有权，所以`s2`在操作后仍然是有效的`String`。`add`获取了`self`的所有权，因为`self`没有`&`，所以`s1`此时被move进函数，之后就不能再使用。方法获取了`s1`的所有权，将`s2`的拷贝追加到了s1中，然后返回了结果字符串的所有权。虽然看起来发生了多次拷贝，实际的实现中没有，具体实现要比拷贝更高效

如果要拼接多个字符串，使用`+`操作符看起来会很繁琐。我们可以使用`format!`宏

```rust
let s1 = String::from("tic");
let s2 = String::from("tac");
let s3 = String::from("toe");

let s = format!("{s1}-{s2}-{s3}");
```

`format!`宏返回一个`String`，这种用法更易读，宏生成的代码使用的参数全部是引用形式，所以这个调用不会获取任何参数的所有权

### 索引字符串

在很多其它编程语言中，使用索引来访问字符串的单个字符是有效的。但是在Rust中这样做会出现编译错误，原因是Rust的字符串不支持索引操作

#### 内部表示

`String`是`Vec<u8>`的包装器，字符串的长度是使用UTF-8编码字符串内容所需要的字节数量，因为每个字符串中的Unicode标量值会占1～4个字节，所以字符串的某个索引不总是一个有效的Unicode标量值。因为返回某个位置的字节值不一定是用户希望得到某个“字符”的行为，并且用户通常也不需要某个字节的值，即使可以保证字符串只包含latin字母（编码为1个字节）。因为要避免返回非期望值或者触发无法发现的bug，Rust完全不会编译这种代码，可以在开发过程初期就避免出现这种错误

#### 字节、标量值和字形簇

在Rust视角有三种关联的方式来看待字符串：作为字节数组、标量值和字形簇（类似我们所说的字母）。对于字符串“नमस्ते”，这三种表示形式分别为

```text
[224, 164, 168, 224, 164, 174, 224, 164, 184, 224, 165, 141, 224, 164, 164, 224, 165, 135]
['न', 'म', 'स', '्', 'त', 'े']
["न", "म", "स्", "ते"]
```

Rust提供不同的方式来表示原始字符串数据，不同的程序可以选择它需要的表示方式。Rust不允许索引`String`的原因之一是索引字符串操作的预期时间复杂度应该是`O(1)`，但是为了保证`String`本身的性能这是不可能实现的，因为Rust必须从头开始遍历内容才能确定有效字符的具体位置

#### 字符串切片

如果你真的需要创建字符串切片，Rust需要你提供更具体的信息。可以使用`[]`结合一个range表达式来创建一个包含指定字节的字符串切片。但是如果我们尝试创建一个只包含某个字符编码的部分字节的切片，那么Rust在运行时会程序崩溃，即访问了vector中一个无效的索引

### 遍历字符串的方法

操作字符串的最好方式是明确你想要字符还是字节。对于单个Unicode标量值，使用`chars`方法，`bytes`方法会返回每个原始字节

> ⚠️有效的Unicode标量值可能由超过一个字节组成

从字符串中获取字形簇很复杂，所以标准库没有提供对应的功能，可以寻找第三方crate

### 字符串不简单

Rust选择将程序对`String`数据进行正确处理作为默认行为，这意味着程序员不得不在处理UTF-8数据之前先做考虑。这种实现暴露了字符串处理的复杂性，但是它也防止你遇到很多错误，包括non-ASCII字符等

## 在Hash表中存储键值对

`HashMap<K, V>`类型使用**哈希函数**（*hashing function*）存储一组类型是`K`的键和类型是`V`的值，哈希函数决定了这些键和值在内存中的位置。当你不希望使用索引值来查找数据时可以使用哈希表

### 创建一个新的哈希表

创建一个空哈希表的一种方式是`new`，然后使用`insert`来添加元素

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
```

哈希表相对于vector和字符串来说不算常用，所以没有放在预包含内容中。哈希表在标准库中也没有提供太多支持，例如不存在内置的构建它们的宏（就像`vec!`）。哈希表将数据存储在堆中。哈希表中所有的键必须有一致的类型，所有的值也必须有一致的类型

### 访问哈希表中的值

通过`get`方法来使用键访问哈希表中对应的值

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

let team_name = String::from("Blue");
let score = scores.get(&team_name).copied().unwrap_or(0);
```

`get`方法返回一个`Option<&V>`类型的值。如果这个键没有对应值或键不存在，那么返回`None`。上面代码通过调用`copied`来得到一个`Option<i32>`，然后如果这个值不存在就设置为`0`

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

for (key, value) in &scores {
    println!("{key}: {value}");
}
```

### 哈希表和所有权

对于实现了`Copy`特质的类型来说，值会被拷贝进哈希表，对于自有数据，例如`String`，值会被移动，哈希表会成为这些值的所有者。如果我们将值（非`Copy`）的引用插入到了哈希表，值本身不会被移动进哈希表。但是引用指向的值至少要在哈希表有效期间保持有效

### 更新一个哈希表

键值对的数量会增加，但是每个单独的键同时只能有一个关联的值。当你要改变哈希表中的数据时，你需要处理键已经有对应值的情况

#### 覆盖一个值

如果我们插入一个键值对后，再插入一个键相同但值不相同的键值对，那么哈希表中该键对应的值会被替换掉。只有当键不存在时才会添加键值对。一种常见的操作模式是当键存在，那么已有的值不改变，否则插入新的键值对。哈希表有`entry`API来接收你要检查的键。它的返回值是叫做`Entry`的枚举来表示存在或不存在的值

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);

scores.entry(String::from("Yellow")).or_insert(50);
scores.entry(String::from("Blue")).or_insert(50);

println!("{scores:?}");
```

`or_insert`方法的效果是，如果键存在的会返回一个对应`Entry`键的值的**可变引用**，如果键不存在，它会将方法参数作为这个键对应的新值插入进去并把新值的可变引用返回出来

#### 基于旧值更新值

另一个常见的模式是查找键的值，然后根据这个值来更新它

```rust
use std::collections::HashMap;

let text = "hello world wonderful world";

let mut map = HashMap::new();

for word in text.split_whitespace() {
    let count = map.entry(word).or_insert(0);
    *count += 1;
}

println!("{map:?}");
```

像上一节所讲的，`or_insert`方法返回的是一个`&mut V`，使用`*`进行解引用之后，可以更新它的值。可变引用在离开`for`循环之后会失效，所以这些改变是安全的并且可以通过借用检查

### 哈希函数

默认情况下`HashMap`使用了叫做`SipHash`的哈希函数，可以抵抗哈希表的DoS（denial-of-service）攻击。这不是最快的哈希算法，选择它是性能和安全的权衡结果。如果你觉得默认哈希算法太慢，可以通过指定其它的`hasher`改变哈希函数。`hasher`是一个实现了`BuildHasher`特质的类型。通常不需要自己手动实现`hasher`，可以用第三方`hasher`
