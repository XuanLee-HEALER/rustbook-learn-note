# 开始

## 安装

推荐使用`rustup`工具来下载Rust，它可以管理Rust版本和相关的工具，这种方式需要互联网

### 在Linux或macOS上安装`rustup`

`$ curl --proto '=https' --tlsv1.2 <https://sh.rustup.rs> -sSf | sh`

这条命令会下载一个脚本，然后自动执行安装`rustup`工具，之后安装最新的*稳定版Rust*。你还需要一个linker（链接器），Rust要用它将编译输出添加进文件中。如果安装过程碰到链接器错误，你应该安装一个C编译器，通常会带一个linker，并且一些常用的Rust包会依赖C代码，也会需要C编译器

macOS上可以使用`xcode-select --install`来安装C编译器。Linux用户通常安装GCC或者Clang，可以查看Linux发行版的文档

### 在Windows上安装`rustup`

直接在网站下载`rustup`的安装器：`https://www.rust-lang.org/tools/install`

### 可能遇到的问题

安装好Rust后通过`rustc --version`查看Rust是否安装成功。如果没有看到版本信息，需要检查Rust是否在系统的`%PATH%`变量中

### 更新和卸载

- 使用`rustup update`来更新Rust版本到最新的发行版
- 使用`rustup self uninstall`来卸载Rust和rustup

### 本地文档

Rust本身带有一份可以离线阅读的文档，运行`rustup doc`可以在浏览器打开文档

安装本地文档（macOS）：`rustup component add --toolchain stable-aarch64-apple-darwin rust-docs`

## Hello, World

找一个空目录，创建一个`main.rs`的源文件。在项目根目录下运行`rustc main.rs`会编译并运行文件

```rust
// Rust程序的结构
fn main() {
}
```

上面的代码定义了函数`main`，`main`函数是每个Rust**可执行程序**的程序入口。如果有参数，会写在`()`中。函数体被包裹在`{}`中。将`{`和函数名称放在同一行是推荐的风格（Rust有自动格式化工具`rustfmt`）

`println!("hello, world!");`，这行代码中有4个重要的细节

1. `println!`调用了Rust的宏（*macro*）。目前只要知道使用`!`是在调用宏而不是普通函数即可，宏不总是跟函数遵守相同的规则
2. `"hello, world!"`是字符串字面值，我们将字符串作为参数传递给`println!`
3. 行尾的`;`，表示这条语句已经结束，准备执行下一条
4. 大部分Rust代码结尾都有分号

### 编译和运行是分开的步骤

在运行Rust程序前，你必须使用编译器`rustc`命令来编译它，将文件名作为参数，执行`rustc main.rs`，编译完成后Rust会输出一个二进制可执行程序

Rust是AOT编译语言（*ahead-of-time complied*），即编译后的程序可以在没有安装Rust的环境执行，但是只能在同一编译目标平台运行，不能直接跨平台

## Hello, Cargo

Cargo是Rust的构建系统和包管理器，它能处理很多任务，例如构建代码、下载代码依赖的库并且编译这些依赖库（*dependencies*）。如果你使用`rustup`安装Rust，那么同时也会下载Cargo。使用`cargo --version`可以检查是否成功安装Cargo

### 使用Cargo创建一个项目

`cargo new hello_cargo`会创建叫`hello_cargo`的项目，并且生成一个同名目录。Cargo生成了两个文件和一个目录：*Cargo.toml*文件和`src`目录和`main.rs`文件。同时会初始化了一个Git仓库，有`.gitignore`文件。如果在一个已存在的Git仓库中运行`cargo new`则不会创建git文件，但是可以强制指定创建`cargo new --vcs=git`

```toml
[package]
name = "hello_cargo"
version = "0.1.0"
edition = "2021"

# 查看更多的key和它们的定义，https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

Cargo的配置文件是TOML格式。`[package]`是一个章节标题，表示接下来的语句在配置一个包。接下来的三行设置了Cargo编译所需要的信息，名称、项目版本和Rust的版本。最后一行`[dependencies]`也是一个章节，下面应该列出你的项目依赖的库。

在Rust中，包的代码被叫做**crates**。Cargo会生成一个`HelloWorld`程序。Cargo认为你的源文件在`src`目录。项目的顶层目录只放`README`文件，licence许可信息，配置文件和其它和代码有关的内容

将一个非Cargo生成的项目转成Cargo管理的项目很简单，只要把源代码放到src目录中，然后创建一个*Cargo.toml*文件，使用`cargo init`可以自动创建此文件

### 构建并运行一个Cargo项目

使用`cargo build`来构建项目，这个命令会创建文件`target/debug/hello_cargo`，不是在当前目录创建，因为默认构建模式是debug模式，Cargo会把二进制文件放到debug目录，可以直接运行这个文件。首次执行构建会生成*Cargo.lock*文件，它是用来跟踪项目使用的依赖库的版本，这个文件**不需要手动修改**，Cargo会管理其中的内容

使用`cargo run`可以构建并运行项目。Cargo可以检查出源码文件没有变化，所以在不修改源码的情况下重复执行命令，它不会重新构建而只是运行程序，如果你修改了源代码，Cargo会在运行前重新构建

`cargo check`命令会检查代码保证可以通过编译，但是不会创建可执行文件，因为跳过了生成可执行文件的步骤，这种方式执行更快

### Release构建

当项目准备发布的时候，可以使用`cargo build --release`进行优化编译，这个命令在`target/release`目录创建可执行程序。优化会让你的代码运行得更快，但是会延长编译时间。如果要对你的代码进行benchmark，则需要使用这个可执行文件

### 习惯用Cargo

对于任何Github上的Rust项目，你都可以使用Git拉取代码，然后进入项目目录，直接构建

```bash
git clone example.org/someproject
cd someproject
cargo build
```
