# 解析命令行参数

我们的 CLI 工具的调用方法应该如下：

```console
$ grrs foobar test.txt
```

我们希望此程序将查找 `test.txt` 并打印出包含 `foobar` 的行。
但我们如何获取这个两个值呢？

命令行中，程序名中后面的文本通常被称为“命令行参数”或“命令行标识”（比如 --this）。
操作系统通常会将它们识别为字符串列表——简单的说，以空格分隔。

有很多方法可以识别这些参数，解析，使它们变得更为易于使用。同时，也需要告诉用户，
程序需要哪些参数及对应的格式是什么。

## 获取参数

标准库中提供的 [`std::env::args()`] 方法，提供了运行时给定参数的[迭代器]。
第一项（索引0）是程序名（如 `grrs`），后面的即才是用户给定的参数。

[`std::env::args()`]: https://doc.rust-lang.org/1.39.0/std/env/fn.args.html
[迭代器]: https://doc.rust-lang.org/1.39.0/std/iter/index.html

以此方法获取原始参数就是这么简单（在 `src/main.rs` 的 `fn main()` 函数中）：

```rust,ignore
{{#include cli-args-struct.rs:10:11}}
```

## CLI 参数的数据类型

与其将它们视为单纯的一堆文本，不如将 CLI 参数看成程序输入的自定义的数据类型。

注意 `grrs foobar test.txt`：这里有两个参数，首先是 `pattern`（查看的字符串），
而后是 `path`（查找的文件路径）。

那么，关于这些参数，首先，这两个参数都是程序所必须的，因为我们并未提供默认值，
所以用户需要在使用此程序时提供这两个参数。此外，关于参数的类型：`pattern`
应该是一个字符串；第二个参数则应是文件的路径。

在 Rust 中，根据所处理的数据去构建程序是很常见的，
因此这种看待参数的方法对我们接下来的工作很有帮助，让我们以此开始
（在 `src/main.rs`，`fn main()` 之前）：

```rust,ignore
{{#include cli-args-struct.rs:3:7}}
```

这定义了一个新的，存储了 `pattern` 和 `path` 两项数据的结构体([`struct`])。

[`struct`]: https://doc.rust-lang.org/1.39.0/book/ch05-00-structs.html

<aside>

**注：**
[`PathBuf`] 是可跨平台使用的系统路径类型，特性类似于 [`String`]。

[`PathBuf`]: https://doc.rust-lang.org/1.39.0/std/path/struct.PathBuf.html
[`String`]: https://doc.rust-lang.org/1.39.0/std/string/struct.String.html

</aside>

我们并没有获得程序运行所需的参数，而这正是现在需要去做的。
一种做法是，我们可以手动解析从系统获取的参数列表并以此生成一个结构体，像这样：

```rust,ignore
{{#include cli-args-struct.rs:10:15}}
```

这种方式能正常工作，用起来却不是很方便。如何去满足像 `--pattern="foo"` 或
`--pattern "foo"` 这种参数输入？又如何实现 `--help`？

## 使用 StructOpt 解析 CLI 参数

使用现成的库来实现参数的解析，这是更明智的选择。
[`clap`] 是当前最受欢迎的解析命令行参数的库。它提供了所有你需要的功能，
如子命令、自动补全和完善的帮助信息。

[`structopt`] 库基于 `clap`，并提供了一个“派生”宏来为结构体生成 `clap` 代码。
这可太棒了：我们只需要去声明一个结构体，它就会生成将参数解析为结构体字段的代码！

[`clap`]: https://clap.rs/
[`structopt`]: https://docs.rs/structopt

首先，我们需要在 `Cargo.toml` 文件的 `[dependencies]` 字段里添加上
`structopt = "0.3.13"` 来导入 `structopt`。

现在，在我们的代码中，导入，`use structopt::StructOpt`，在之前创建的
`struct Cli` 的正上方添加上 `#[derive(StructOpt)]`，并顺便写一些文档注释。

它现在看起来是这样的：

```rust,ignore
{{#include cli-args-structopt.rs:3:14}}
```

<aside class="node">

**注意：**
你可以将许多自定义属性添加到字段中。例，我们告诉了 strcutopt 如何解析 `PathBuf`。
如果你想将 `-o` 或 `--output` 后的参数解析为某个字段，可在字段上方添加上
`#[structopt(short = "o", long = "output")]`。更多属性设置，请查看
[structopt documentation][`structopt`]。

</aside>

本例中，在我们的 `Cli` 结构体下方即是 `main` 函数。
当运行程序时，就会调用这个函数，第一行如下：

```rust,ignore
{{#include cli-args-structopt.rs:15:18}}
```

这将尝试解析参数并存储到 `Cli` 结构体中。

但如果解析失败会怎样？这就是使用此方法的美妙之处：Clap 知道它需要什么字段，
及所需字段的类型。它可以自动生成一个不错的 `--help` 信息，
并会依错误给出一些建议——输入的参数应该是 `--output` 而你输入的是 `--putput`。

<aside class="note">

**注意：**
`from_args` 方法应该在 `main` 函数中使用。在它失败后会立即退出并打印错误信息。
请不要在其它地方使用它！

</aside>

## Wrapping up

你的代码现在看起来应该是这样的：

```rust,ignore
{{#include cli-args-structopt.rs}}
```

无参数运行时：

```console
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 10.16s
     Running `target/debug/grrs`
error: The following required arguments were not provided:
    <pattern>
    <path>

USAGE:
    grrs <pattern> <path>

For more information try --help
```

我们可以直接在 `cargo run` 后添加 `--`，并在其后跟上参数来传递：

```console
$ cargo run -- some-pattern some-file
    Finished dev [unoptimized + debuginfo] target(s) in 0.11s
     Running `target/debug/grrs some-pattern some-file`
```

如你所见，没有任何输出。这非常好，意味着我们的程序运行完成且没有错误！

<aside class="exercise">

**练习：**
让这个程序打印出它的参数！

</aside>
