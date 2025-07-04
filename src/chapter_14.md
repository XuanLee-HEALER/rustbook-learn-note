# 第十四章 - Cargo和Crates.io

## 使用发行文件来自定义构建

*release profiles*是预定义的、可以使用不同配置定制的策略，允许程序员对如何编译代码有更多的控制权。每个profile都和其它的profile互相独立。Cargo有两个主要的profile，*dev*是你运行`cargo build`使用的，*release*是你运行`cargo build --release`使用的。Cargo对每种profile有默认的设置，当你添加了`[profile.*]`章节来定制配置时，你可以覆盖一部分默认设置

```toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

`opt-level`是控制Rust编译器优化级别的编号，从`0`到`3`，更高等级的优化编译时间也更长。`dev`的`opt-level`是`0`，当你准备发布代码时，最好花费更多的时间来进行优化编译。通过在*Cargo.toml*中设置不同的值，你可以修改默认的配置

## 发布crate到Crates.io

在Crates.io上的发布crate会同时发布你的源码，所以它主要维护开源代码

### 制作有用的文档注释

为你的包添加准确的文档可以帮助其他人更好地使用。Rust对于文档提供了一种特殊的注释，叫做**文档注释**（*documentation comment*），根据注释可以自动生成HTML文档，HTML中会展示公共API的文档注释的内容。文档注释使用`///`，支持Markdown语法格式化文本。将文档注释放到它注释的项之前

我们可以运行`cargo doc`从文档注释生成HTML文档，它会运行`rustdoc`工具（Rust自带）将生成的HTML文档放到`target/doc`目录中。运行`cargo doc --open`会为你当前的crate构建HTML（包括外部依赖crate）并且在浏览器打开结果

#### 通常使用的章节

我们使用`# Examples`来为API的用法写简单用例，还有一些其它常用的章节

- `Panics`：被注释的函数有可能会崩溃。调用者如果想保证他们的程序不崩溃，就不应该调用这个函数
- `Errors`：如果函数返回一个`Result`，描述可能会遇到的错误类型以及哪些情况会导致这些错误，让调用者可以使用不同的方式处理错误
- `Safety`：如果函数是`unsafe`的，需要解释为什么函数是不安全的，并说明该函数期望调用者遵循的不变性（函数执行过程中必须有效的属性）

#### 文档注释作为测试

在文档注释中写例子可以帮助说明如何使用你的API，运行`cargo test`也会运行文档中的例子，作为测试的一部分

#### 注释包含项（*Contained Items*）

`//!`为**包含注释的项**添加文档，而不是那些跟在注释后的项。通常在crate根文件使用这种文档注释，或者在一个模块内，来描述整个crate或者整个模块的功能。在`//!`后面没有其它代码，通常是一个空行，因为它描述的是包含它的项

### 使用pub use来到处方便的公共API

如果你的crate有复杂的模块层次，用户很难找到他们想使用的内容。你可能想重新组织你的代码对外开放的结构，因为想使用你定义的很深的类型的用户可能很难确定这个类型是否存在。你可以使用`pub use`重新导出项，来让公共结构和你的私有结构不同，重新导出可以实现在一个地方处理公共项。另一个`pub use`的常见用法是将当前crate的依赖中的定义重新导出，让这些crate的定义作为你自己crate的公共API的一部分

创建一套有用的公共API结构是一种艺术而不是科学

### 设置Crates.io账号

登录Crates.io（Github账号）后，查看自己的API key，然后运行`cargo login`，输入这个key。Cargo会将这个key记录到`~/.cargo/creedentials`，这个key是一个秘密，不要跟其他人分享，如果必须要分享出去，之后应该回收它并生产新的key

#### 为一个新crate添加元数据

在发布前，你需要在`[package]`章节添加一些元数据。crate需要一个唯一的名称，Crates.io上的crate名称是先来先服务原则。其它的重要信息：描述和协议信息，描述就是一两句话，`license`需要提供一个license identifier value（license identifier value），如果要使用SPDX不支持的协议，你需要将协议内容放到文件中，将文件放到项目中，使用`license-file`来指定文件名称。使用`OR`来分隔多个协议

#### 发布到Crates.io

因为发布是永久的，版本不能被覆盖，代码也不会被删除。Crates.io的一个主要目标是作为代码的永久归档，保证所有依赖这上面的代码的项目总是能工作，如果允许删除指定版本则无法实现这个目标。但是对于提交的版本数是没有限制的

`cargo publish`是发布crate的命令

#### 发布一个已有crate的新版本

当你准备好发行一个新版本时，改变`version`的值然后重新发布即可

可以使用`cargo yank`来从Crates.io上过期版本。虽然不能移除之前的版本，但是可以阻止未来的项目将这个版本作为依赖。yanking（撤回）一个版本后不允许新项目依赖这个版本，但是那些已有的已经依赖这个版本的项目还可以继续工作。一次撤回意味着有*Cargo.lock*的项目不会破坏，但是未来生成的*Cargo.lock*文件不会使用被撤回的版本。在crate所在目录，运行`cargo yank --vers 1.0.1`可以撤回指定版本，通过添加`--undo`参数，可以再次撤回这个操作。撤回并没有删除任何代码

## Cargo工作空间

### 创建一个工作空间

一个**工作空间**（*workspace*）是包含共享一个*Cargo.lock*和构建输出目录的一组包的目录。以包括一个二进制crate和两个库crate的工作空间为例，这个二进制crate会依赖两个库crate

首先创建一个目录，然后在这个目录中创建一个Cargo.toml来配置整个工作空间，文件中没有[package]章节

```toml
[workspace]
resolver = "2"
```

通过指定`resolver`为`2`来使用最新的Cargo的解析器算法（现在已经到3了--2025）

接下来在这个目录下使用`cargo new`来创建二进制crate，这个命令会自动将新创建的包添加到工作空间的*Cargo.toml*中

```toml
[workspace]
resolver = "2"
members = ["adder"]
```

此时在工作空间中运行`cargo build`，工作空间顶层目录会出现一个`target`目录，具体的包中则没有这个目录，即使在具体的包中执行这个命令，输出目录也在最外层，因为工作空间中的crate是互相依赖的

### 在工作空间中创建第二个包

在*Cargo.toml*中的`members`列表里加上第二个包名。然后使用`cargo new add_one --lib`来生成一个新的库crate，在库中创建好函数后，在二进制crate所在包的*Cargo.toml*中添加依赖

```toml
[dependencies]
add_one = { path = "../add_one" }
```

要从工作空间中执行某个二进制crate，可以使用`-p`参数传入包名

### 在工作空间中依赖外部包

工作空间只有顶层的一个*Cargo.lock*文件，保证所有的包使用相同版本的依赖。如果我们将`rand`包添加到两个crate的*Cargo.toml*中，Cargo会将它们处理为一个版本，然后记录进*Cargo.lcok*中。注意在一个包中添加的依赖虽然被记录进了工作空间的*Cargo.lock*中，但是如果其它包不在*Cargo.toml*中声明，是不能直接用这个依赖的。如果工作空间中的crate使用了相同依赖的不兼容版本，Cargo会依次解决它们，但是仍然会尝试处理成尽量少的版本

### 在工作空间中写测试

在工作空间的目录下运行`cargo test`会运行工作空间中所有包的测试。通过`-p`来指定我们想运行测试的特定包名称。如果你想将工作空间中的crate发布到Crates.io，每个crate需要单独发布。我们可以使用`-p`来发布工作空间中特定的包

当你的项目变大时，可以考虑使用工作空间：理解小的、独立的组件比一整块代码更容易。将crate放在工作空间中，如果它们同时发生变化，crate之间的协作更容易

## 使用`cargo install`安装二进制程序

`cargo install`可以让你本地安装和使用二进制crate。对Rust开发者来说是方便安装其他开发者在Crates.io上分享的工具的方式。你只能安装那些有二进制目标的包。所有的二进制包会下载到Rust安装根目录的`bin`目录下，如果使用`rustup.rs`安装的Rust并且没有自定义配置，默认路径应该是`$HOME/.cargo/bin`

### 使用自定义命令扩展Cargo

如果你的`$PATH`中有名称是`cargo-something`的二进制程序，你可以使用`cargo something`的自命令格式来运行它。使用`cargo --list`可以列出所有支持的子命令
