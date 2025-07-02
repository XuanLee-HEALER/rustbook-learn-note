# 第十一章 - 自动化测试

## 如何写测试

测试是用来验证非测试代码是否可以符合预期行为的函数。测试函数的内容执行下列三种操作

- 设置需要的数据或状态
- 运行希望测试的代码
- **断言**（*assert*）结果是否是期望值

### 测试函数的分析

Rust中的测试函数就是注解`test`属性的函数。属性是Rust代码的元数据。要将一个函数变成一个测试函数，在`fn`的上一行加上`#[test]`即可。使用`cargo test`命令，Rust会构建一个测试运行程序来运行这些有注解的函数，并且生成测试报告。当我们使用cargo创建一个新的库项目时，自动生成的代码中包含一个测试函数

```rust
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

我们也可以在`tests`模块中放入非测试函数，来设置测试场景或者执行公共操作，所以我们需要标记哪些函数是测试函数。测试报告中的`0 measured`是性能测试的指标。Benchmark测试只在nightly Rust中才有。我们可以向`cargo test`传入参数来运行那些匹配了指定模式的测试函数，这叫做**过滤**（*filtering*）。`Doc-tests adder`是文档测试的结果。Rust可以编译在API文档中出现的任何代码例子。当测试函数中的一些内容崩溃测试就会失败，每个测试在一个新线程中运行，当主线程看到测试线程死亡后，这个测试就被标记为失败

### 使用`assert!`宏来检查结果

当你想确保在测试中一些条件总是计算为true时应该使用`assert!`，我们传入的参数是可以计算为布尔类型的表达式，如果结果是`true`，什么事都不会发生，测试通过，如果是`false`，Rust会调用`panic!`来让测试失败

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };

        assert!(larger.can_hold(&smaller));
    }
}
```

因为`tests`模块是内部模块，我们需要将外部模块的代码引入域中，这里使用`use super::*;`，外部模块定义的所有内容在内部模块都能看到

### 使用`assert_eq!`和`assert_ne!`宏来测试相等性

一种常见的测试方式是验证代码结果和预期结果的相等性。可以通过结合`assert!`和`==`来实现。但是使用`assert_eq!`和`assert_ne!`更方便，它们接收两个参数来比较两个参数相等或不相等。如果断言失败会打印出两者的值，而`assert!`只会得到最终的比较值是`false`。在Rust中，被比较的两个参数叫做`left`和`right`，我们指定值时没有位置要求。`assert_ne!`在那些不确定测试的值是什么，但是可以确定它不是什么的时候很有用。因为这两个宏底层使用的是`==`和`!=`比较，以`debug`格式打印值，所以传入的值需要实现`PartialEq`和`Debug`特质。对于那些你自己实现的类型，需要实现这些特质才能在宏中使用。因为这些特质都是可继承的，所以使用`#[derive(PartialEq, Debug)]`注解到结构体上即可

### 添加自定义错误信息

在`assert!`、`assert_eq!`和`assert_ne!`宏中，必填参数后面的参数会传入`format!`宏。自定义信息对于解释断言含义很有用

```rust
#[test]
fn greeting_contains_name() {
    let result = greeting("Carol");
    assert!(
        result.contains("Carol"),
        "Greeting did not contain name, value was `{result}`"
    );
}
```

### 使用`should_panic`来检查崩溃

除了检查返回值，检查代码是否如期出现错误也是很重要的。我们可以通过在测试函数上加`#[should_panic]`属性来做这件事，如果函数中的代码崩溃那么测试通过，否则测试失败

```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {value}.");
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

使用`should_panic`的测试可能不太精确，因为其它不是我们希望测试的原因导致测试函数崩溃也会导致测试通过，所以我们可以为这个属性额外添加`expected`参数，测试会确保错误信息必须包含参数指定的字符串

```rust
// --snip--

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 {
            panic!(
                "Guess value must be greater than or equal to 1, got {value}."
            );
        } else if value > 100 {
            panic!(
                "Guess value must be less than or equal to 100, got {value}."
            );
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

如何指定错误信息取决于导致崩溃的错误信息是独有的还是动态的，还有你希望你的测试有多精确

### 在测试中使用`Result<T, E>`

```rust
#[test]
fn it_works() -> Result<(), String> {
    let result = add(2, 2);

    if result == 4 {
        Ok(())
    } else {
        Err(String::from("two plus two does not equal four"))
    }
}
```

当测试通过我们返回`Ok(())`，当测试失败返回`Err`。写这种测试函数可以让你在测试函数中使用`?`语法，如果测试函数中任何操作报错，整个测试就会失败。这种测试函数不能注解`#[should_panic]`，要断言一个操作返回了`Err`，不要在`Result<T, E>`上使用`?`，而是使用`assert!(value.is_err())`，因为前者会直接退出测试

## 控制测试如何运行

`cargo test`会在测试模式下编译你的代码，运行最终的测试执行程序。这个程序的默认行为是并行运行所有测试，捕获在测试过程中生成的所有输出，不直接打印这些输出内容，将其改为更易读的测试报告输出。你可以通过命令行参数来改变此行为。有些参数影响`cargo test`命令，有些会影响最终执行的二进制程序。要区分这两种参数，你需要在`cargo test`后列出参数，然后使用`--`分隔测试程序的参数

### 并行或者顺序运行测试

并行运行测试会更快地运行测试函数。你必须确保你的测试没有互相依赖或者依赖任何共享状态，包括共享环境，例如当前工作目录或者环境变量。如果你不想并行运行测试或者如果你想更细粒度地控制使用多少线程运行测试，你可以发送`--test-threads`和线程数量给测试程序`$ cargo test -- --test-threads=1`

### 显示函数输出

默认情况下，如果测试通过，Rust的测试库会捕获任何打印到标准输出的内容，我们什么都看不到。如果测试失败，我们会看到打印的测试信息和错误信息。如果你想看到通过的测试函数打印的值，需要使用`--show-output`，`$ cargo test -- --show-output`

### 通过指定名称来运行一部分测试

你可以通过向`cargo test`传入你想运行的测试名称作为参数

```rust
pub fn add_two(a: usize) -> usize {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_two_and_two() {
        let result = add_two(2);
        assert_eq!(result, 4);
    }

    #[test]
    fn add_three_and_two() {
        let result = add_two(3);
        assert_eq!(result, 5);
    }

    #[test]
    fn one_hundred() {
        let result = add_two(100);
        assert_eq!(result, 102);
    }
}
```

#### 运行单个测试

`$ cargo test one_hundred`，测试结果中的`2 filtered out`告诉我们还有两个的没有运行的测试

#### 使用过滤器运行多个测试

我们可以指定测试函数名称的模式，任何匹配这个模式的测试函数都会被运行，`$ cargo test add`会运行所有名字中有`add`的测试，还需要注意的是测试名称中**模块名也是名字的一部分**，所以可以通过模块名来运行一个模块下所有的测试

#### 除非指定否则忽略一些测试

有时一些测试函数非常耗时，所以你需要忽略它们。不需要在执行`cargo test`时列出它们，只要在它们的声明上加上`ignore`属性即可。如果我们只想运行那些被忽略的测试，我们可以使用`cargo test -- --ignored`。如果你想运行所有测试，无论它们是否被忽略，可以使用`cargo test -- --include-ignored`

## 测试结构

Rust社区将测试大致分为两类：单元测试和集成测试。**单元测试**（*unit tests*）是那些短小的、专注于单一功能的、每次单独测试一个模块的、可以测试私有接口的测试。**集成测试**（*integration tests*）是完全独立于你的库的外部测试，以和外部代码一样的形式调用你的代码，只使用公共接口并且会在每个测试中调用多个模块

### 单元测试

单元测试的目标是测试独立于其它代码的孤立代码，快速定位代码运行结果是否符合预期。通常你会将单元测试放到`src`目录中每个它们所测试的源代码文件中。习惯是创建一个`tests`模块，使用`cfg(test)`来注解这个模块，里面是测试函数

#### 测试模块和`#[cfg(test)]`

`#[cfg(test)]`告诉Rust，只有运行`cargo test`才会编译和运行模块中的测试代码。如果是编译库，会节省时间和空间，因为测试代码不会包含在内。因为集成测试在其它目录，所以不需要这个注解

`cfg`属性表示*configuration*，告诉Rust注解的项只有在提供了某些配置选项时才应该被包含。对于测试模块，这个属性是`test`。通过使用`cfg`属性，Cargo只有在我们运行`cargo test`时才编译测试代码。这包括模块中任何的辅助函数，不止是那些注解了`#[test]`的函数

#### 测试私有函数

Rust的隐私规则允许你测试私有函数

```rust
pub fn add_two(a: usize) -> usize {
    internal_adder(a, 2)
}

fn internal_adder(left: usize, right: usize) -> usize {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal() {
        let result = internal_adder(2, 2);
        assert_eq!(result, 4);
    }
}
```

> 测试模块本身也只是另一个普通的模块，当你将外部模块的内容引入域中，就可以进行测试。如果你认为私有函数不应该被测试，那么Rust也不强迫你这样做

### 集成测试

集成测试的目的是测试你的库的各个部分是否可以正确工作。有时不同代码单元各自可以正确工作，但是当它们集成时可能会出现问题。要创建集成测试，你首先需要创建`tests`目录

#### tests目录

我们在项目顶层创建`tests`目录，和`src`在同一层级，然后我们可以创建任意数量的测试文件，Cargo会将每个文件编译成单独的crate。目录中每个文件都是一个单独的crate，所以我们需要将我们的库引入每个测试crate的域。`cargo test`的输出显示三部分内容，包括单元测试、集成测试和文档测试。如果有一部分测试出错，那么之后的测试不会被运行。集成测试部分每行显示一个函数，最后是测试结果行，每个集成测试文件有自己的一部分。同样我们也可以通过指定测试函数的名称作为`cargo test`的参数，指定希望运行的集成测试。要运行一个指定集成测试文件中所有的测试，使用`--test`指定这个文件的名称

#### 集成测试中的子模块

当你添加了更多的集成测试后，你可以想创建更多的文件来组织它们。例如你可以根据它们测试的功能将测试函数放到一起。每个文件编译成一个单独crate无法让`tests`目录下的文件共享一些行为。如果我们有一些多个集成测试文件中都使用的辅助函数，如果你直接在`tests`目录下创建一个文件，然后写一些函数，这个文件会被编译成单独的crate。所以需要按照老式的方式，先创建一个模块同名的目录，然后在里面创建一个`mod.rs`，将这些函数写进去，这个文件不会被Cargo认为是一个独立的集成测试。然后在其它的集成测试文件中使用`mod common;`来声明这个模块，要注意每个集成测试文件都需要声明才能使用这个模块

#### 对二进制crate的集成测试

如果我们的项目只有二进制crate，没有库crate，那么我们不能在`tests`目录创建集成测试，只有库crate才能暴露其它crate可以使用的函数
