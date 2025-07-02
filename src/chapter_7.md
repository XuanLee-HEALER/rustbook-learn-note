# 第七章 - 使用Packages、Crates、Modules来管理项目

当你的程序规模变大，组织代码的方式也会更加重要。通过将相关功能分组，分离实现不同特性的代码，你可以更容易地查找实现了某些特性的代码，也知道在哪里修改某些特性的工作方式。随着项目增长，你应该将代码分成多个模块，放在多个文件中。一个**包**（*package*）可以包含多个二进制crates和一个库crate（可选）。随着包内容增多，你可以将一部分的crates提取出来变成外部依赖的包。对于包含一些互相关联的包的大型项目，Cargo提供了**工作空间**（*workspaces*），后续章节会介绍

域是一个相关的概念：代码所在的上下文中有一些被定义在“域内”的名称。当读、写和编译代码时，程序员和编译器都需要知道某个位置的特殊名称引用的变量、函数、结构体、枚举、模块、常量或其它内容是什么。你可以创建域并改变域内和域外的名称。在**同一个域中不能有两个相同的名称**

Rust的**模块系统**（*module system*）允许你利用模块管理自己的代码

- Packages：Cargo特性，用来构建、测试和分享crates
- Crates：模块树，可以生成库或者可执行文件
- Modules和use：控制代码的组织方式、域和路径的隐私
- Paths：命名一个项的方式，例如结构体、函数或者模块

## Packages和Crates

Rust编译器处理的最小代码单元是一个crate。即使你不用Cargo而是用`rustc`和源文件参数，编译器会将这个文件名作为一个crate处理。crate可以包括模块，这些模块可能定义在其它文件中，这些文件也会随着这个crate一起编译

crate有两种形式

- Binary crate：会被编译为可执行程序，可以运行，例如命令行程序或者一个服务器。每个crate必须有一个函数叫做`main`，定义当程序运行时做什么
- Library crate：没有`main`函数，它们不会编译成可执行文件，定义那些在多个项目中共享的功能。一般crate和library在Rust社区的交流中是可以互换使用的

*crate root*是Rust编译器开始构建你的crate的根模块的源文件。package是一个或多个crate的组合，提供一些功能。一个包包含*Cargo.toml*文件，描述如何构建那些crate。Cargo本身就是一个包，包含一个二进制crate作为命令行工具。Cargo包也包含程序依赖的库crate。其它的项目可以依赖Cargo包的库crate来使用Cargo命令行工具的逻辑。一个包可以包括**任意多个**二进制crate，但是只能包含**一个**库crate。一个包至少需要包含一个crate，可以是库crate或者二进制crate

执行`cargo new my-project`后，项目目录`my-project`中会出现一个*Cargo.toml*文件，这个项目目录就是一个包。`src`目录包含`main.rs`文件。在*Cargo.toml*文件中没有提到`src/main.rs`文件。Cargo约定如果有`src/main.rs`文件，它就是一个二进制crate的根，这个二进制crate的名称和包名相同。类似的，如果包目录包含`src/lib.rs`，这个包就包含一个和包名相同的库crate，这个文件就是库crate的根。Cargo会将crate的根所在文件传给`rustc`来构建库crate或者二进制crate

如果一个包同时包含`src/main.rs`和`src/lib.rs`的话，这个包就有两个和包同名的crate。一个包可以有多个二进制包，将它们的包含`main`函数的文件放到`src/bin`目录，每个文件都是一个单独的二进制crate

## 定义模块来控制域和隐私

### 模块速记

以下是模块、路径、`use`关键字和`pub`关键字的工作方式的快速介绍

- 从*crate root*开始：当编译一个crate的时候，编译器首先会找*crate root*文件（就是`src/main.rs`和`src/lib.rs`）的代码进行编译
- 声明模块：在crate根文件中，你可以声明新的模块，假设你使用`mod garden;`声明了garden模块，编译器会在下列地点查询这个模块的代码
  - inline（行内）：在替换`mod garden`后面分号的语句块中
  - 在文件`src/garden.rs`中
  - 在文件`src/garden/mod.rs`中
- 声明子模块：在任何除了crate根的文件中，你可以声明子模块。比如你在`src/garden.rs`中声明了`mod vegetables;`，那么编译器会在目录名为父模块中查找子模块的代码
  - inline（行内）：直接在`mod vegetables`后面的语句块中
  - 在文件`src/garden/vegetables.rs`
  - 在文件`src/garden/vegetables/mod.rs`
- 在模块中指向代码的路径：一旦一个模块是你crate的一部分，只要隐私规则允许，你可以在**同一个**crate中的任意地方通过使用代码的路径引用这个模块中的代码。例如`vegetables`模块中的`Asparagus`类型可以通过`crate::garden::vegetables::Asparagus`找到
- 私有 vs. 公有：**模块中的代码默认对于它的父模块是私有的**。要让一个模块是公有的，要使用`pub mod`声明。要让公有模块中的项是公有的，在项的声明之前使用`pub`
- `use`关键字：在一个域中，use关键字创建了其它项的快捷方式来减少长路径的重复。在任何域都可以使用全路径来引用项，但是使用use后可以只使用短路径

### 将关联代码放到模块中

模块让我们将crate中的代码以具有更好的可读性和可重用性的方式组织起来。因为模块中的代码默认是私有的，模块也允许我们修改项的隐私权。私有项是外部使用不需要看到的内部实现。我们可以让模块和它内部的项成为公共的，将它们暴露给外部要使用和依赖它们的代码

下面是一个饭店的例子，饭店包括前台和后厨，顾客在前台接受服务，后厨是厨师工作的地方。比如创建了一个饭店的项目（包）`cargo new restaurant --lib`，然后在`src/lib.rs`中定义一些模块和函数签名，下面的代码是前台代码

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```

模块中也可以包括其它项的定义，例如结构体、枚举、常量、特质、函数。通过使用模块，我们将关联的项的定义放在一起并且用它们之间的关系来为模块命名。程序员可以基于组织结构来导航查看代码而不是直接阅读所有项的定义。在添加新功能时也知道将代码放到哪里

之所以`src/main.rs`和`src/lib.rs`是crate的根，因为其中任意一个文件的内容会在crate的**模块树**（*module tree*）的根的位置构成一个叫做crate的模块

### 在模块树上使用路径来引用项

要调用一个函数，我们需要知道它的路径。路径有两种形式

- 绝对路径（absolute path）：从crate根开始的全路径，对于外部crate的代码来说，全路径是从它的crate名开始的。对于当前的crate直接从crate开始
- 相对路径（relative path）：从当前模块开始，使用`self`和`super`或者当前模块的名称开始

绝对路径和相对路径后面都可以跟多个标识符，使用`::`分隔

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```

我们第一次使用绝对路径调用函数，因为这个函数和`eat_at_restaurant`函数在同一个crate中，所以使用crate关键字来作为路径开头。第二次调用函数，使用了相对路径，路径以`front_of_house`开头，这个模块的名字和函数在模块树中与当前模块处于同一层级。选择使用绝对路径还是相对路径取决于，在你的项目中，你是否可能在调用它们的代码中移动这些项的定义代码或者两者一起移动。例如我们把上面的模块和函数一起移动到另一个模块中，那么绝对路径就需要修改，如果我们只把函数移到另一个模块中，那么相对路径就需要修改。通常我们会使用绝对路径，因为我们更可能单独地移动项的定义部分或调用部分

要注意上面的代码是无法通过编译的。`hosting`模块默认是私有的，在Rust中，所有的项对于自己的父模块默认都是私有的，如果你想让一个函数或者结构体变成私有的，将它们放到模块中就可以。父模块中的项不能使用子模块中的私有项，但是子模块可以使用它们的祖先模块中的项，这是因为子模块隐藏具体的实现细节，但是子模块可以看到它们被定义位置的上下文中的内容

Rust选择使用这种模块系统的工作方式，使得隐藏内部实现是默认行为。这种情况下，你可以在不破坏外部代码的情况下修改内部代码

### 使用`pub`关键字暴露路径

修改上面的代码片段，首先让`hosting`模块变成公有的，但这只是让模块变成公有的，这代表如果我们可以访问`front_of_house`，那么我们可以访问`hosting`，但是`hosting`内部的项仍然是私有的。让模块公有不能让它的项自动公有。模块上的`pub`关键字只是让它的祖先模块可以引用到自己，而不能访问其内部的代码。只需要在`add_to_waitlist`前再加上`pub`关键字就可以访问了

如果你计划分享你的库crate给其它项目用，你的公共API就是你和你的用户协商决定他们可以如何与你的代码交互的结果

### 包括一个二进制和一个库的包的最佳实践

`src/main.rs`和`src/lib.rs`是两个crate根，默认都用包名。通常二进制crate只包括调用库crate的代码，因为库crate的代码可以被共享。模块树应该定义在`src/lib.rs`中。任何公有项都能在二进制crate中使用，通过包名作为开始路径

### 用`super`开始相对路径

我们可以在路径前使用`super`从父模块开始相对路径，这允许我们引用父模块中的内容，当模块与自己的父模块关联紧密，但是父模块可能会被移走时我们通过使用`super`可以更简单地重新组织模块树

### 让结构体和枚举变为公有的

对于`pub`在结构体和枚举上的使用，还有一些额外的细节需要注意。在结构体前使用`pub`，只是让结构体变为公有，但是它的成员还是私有的，这种设计可以让内部和外部使用者对于同一个结构体中他们各自关心的数据进行分离。因为结构体有私有成员，结构体需要提供一个公有的关联函数来构建一个实例，如果没有这个函数我们无法就创建实例。相反，如果我们让一个枚举为公有的，那么它的所有成员都是公有的。枚举通常在它们的成员都是公有的情况下才有用

### 使用use关键字将路径带入域中

必须写全路径才能调用函数很不方便而且过于重复，无论使用绝对路径还是相对路径。我们可以使用use关键字来创建一个路径的快捷方式

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

在域中使用`use`和一个路径就像在文件系统中创建了一个链接，`hosting`变成了域中一个有效的名字，表示`hosting`模块，被引入的路径同样会检查隐私性

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

mod customer {
    pub fn eat_at_restaurant() {
        hosting::add_to_waitlist();
    }
}
```

`use`只会在出现的域中创建快捷方式，这段代码无法通过编译，因为`eat_at_restaurant`被放入`customer`模块中，已经看不到`hosting`这个名字了，只需要将这个路径移入`customer`模块或者在前面加上`super::`就可以使用

### 创建更符合习惯的use路径

在调用函数时引入它的父模块是更符合习惯的`use`用法，既不会让函数名称与局部变量搞混，也最大程度减少了代码的重复。另一方面，对于结构体、枚举和其它的项，使用`use`一般会把它们的全路径引入进来。这只是逐渐形成的约定，人们也已经习惯以这种方式来读写Rust代码。例外是当不同的模块下有相同的项时，不能将全路径引入，必须使用父模块名来引用

### 使用as关键字来提供一个新名字

还有一种方式可以将两个同名的类型引入一个域中，在路径后使用`as`和一个新名字

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```

### 使用`pub use`来重新导出名称

当我们使用`use`关键字将路径引入域，在新域中这个名字是私有的。要让调用我们代码的代码也可以访问这个名称，我们可以使用`pub use`，这种行为叫做**重导出**（*re-exporting*）

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

改动之前外部代码要调用`add_to_waitlist`函数必须使用全路径，并且`front_of_house`需要声明为`pub`。现在`hosting`模块已经从根模块被重导出，外部代码可以直接访问。通过`pub use`，我们可以在代码中使用一种结构，以另外一种结构将API暴露给调用者。让我们的库对于开发者和调用者都是组织得当的

### 使用外部包

调用外部包的步骤

1. 将包名加入你自己的包的*Cargo.toml*文件中
2. 从它们的crate中使用use将项引入域中

标准库`std`也是一个crate，因为标准库和Rust语言一起发布，所以不需要显式引入。但是需要使用`use`将它们引入包的域

### 使用嵌套路径来清理大量的use列表

在规模很大的程序中，使用嵌套路径可以减少单独的`use`语句的数量

```rust
// --snip--
use std::cmp::Ordering;
use std::io;
// --snip--

// --snip--
use std::{cmp::Ordering, io};
// --snip--

use std::io;
use std::io::Write;

use std::io::{self, Write};
```

### Glob操作符

如果我们要将某个路径下所有的公共项都引入域，我们可以在路径后加上`*`通配符。但是通配符很难确切知道哪些名字被引入了域、这些名字在哪里被使用，通配符通常在`tests`模块中测试所有内容时使用

### 将模块分到不同的文件中

当模块变得越来越大，你可能想将它们的定义放到单独的文件中让代码更容易导航

```rust
mod front_of_house;

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

将`front_of_house`模块的内容放到单独的文件中

```rust
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```

只需要在模块树中使用`mod`声明来加载文件。当编译器知道文件是项目的一部分之后，项目中的其它文件可以使用声明的路径来引用已经加载的文件。因为`hosting`是`front_of_house`的子模块，所以它应该在同名的新目录下

- `src/front_of_house.rs`
- `src/front_of_house/mod.rs`（老）

```rust
pub mod hosting;
pub fn add_to_waitlist() {}
```

如果我们将`hosting.rs`文件放到`src`目录下，编译器会认为这个模块声明在crate根下，而不是`front_of_house`模块的子模块

老的文件路径风格中，每个新模块都放到`mod.rs`文件中，并且要新建一个模块同名目录。一个项目中如果两种风格的模块路径风格同时使用会导致编译错误
