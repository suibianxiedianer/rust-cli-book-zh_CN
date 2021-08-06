# 输出

## 打印 "Hello World"

```rust
println!("Hello World");
```

很简单是吧，让我们进入下一个话题。

## 使用 println

使用 `println!` 宏你几乎可以打印出所有你喜欢的东西。这个宏有一些很棒的功能，
也有一些特殊的语法。它需要你写一个字符串作为第一个参数，其中包括占位符，
这些占位符将由后面的参数的值作为参数来填充。

比如：

```rust
let x = 42;
println!("My lucky number is {}.", x);
```

将打印

```console
My lucky number is 42.
```

上面字符串中的花括号（'{}'）就是占位符中的一种。这是默认的占位符类型，
它尝试以人类可读的方式打印出给定的参数的值。对于数字和字符串，这很好用，
但并不是所有的类型都可行。这就是为什么还有一个 "debug representation"，
你可以使用这个占位符来调用它 `{:?}`。

比如，

```rust
let xs = vec![1, 2, 3];
println!("The list is: {:?}", xs);
```

会打印

```console
The list is: [1, 2, 3]
```

如果你想在调试和日志中打印自己构建的类型，大部分情况下你可以在类型定义上添加
`#[derive(Debug)]` 属性。

<aside>

**注：**
“用户友好型”打印是通过 [`Display`] 特征实现的，而调试输出（适用于开发者）
是使用 [`Debug`] 特征。你可以在 [`std::fmt` 模块文档][std::fmt]
查看更多关于使用 `println!` 的语法信息。

[`Display`]: https://doc.rust-lang.org/1.39.0/std/fmt/trait.Display.html
[`Debug`]: https://doc.rust-lang.org/1.39.0/std/fmt/trait.Debug.html
[std::fmt]: https://doc.rust-lang.org/1.39.0/std/fmt/index.html

</aside>

## 打印错误

打印错误应通过 `stderr` 完成，
以便用户和其它工具更方便的地将输出通过管道传输到文件或更多的工具中。

<aside>

**注：**
在大部分操作系统中，一个程序可以将输出写到两个流中，`stdout` 和 `stderr`。
`stdout` 用于程序的实际输出，而 `stderr` 可将错误或其它信息与 `stdout` 分开。
这样，正确输出可以存储到文件或管道传输到其它程序中，同时将错误展示给用户。

</aside>

在 Rust 中，使用 `println!` 和 `eprintln!`，前者对应 `stdout` 而后者 `stderr`。

```rust
println!("This is information");
eprintln!("This is an error! :(");
```

<aside>

**当心**: 打印 [escape codes] 可能很危险，会导致用户的终端变成奇异的状态。
在手动打印时请务必小心使用！

[escape codes]: https://en.wikipedia.org/wiki/ANSI_escape_code

在你处理原始转义码时，最好使用 `ansi_term` 这样的 crate，
以便你（和你程序的用户）更安心。

</aside>

## 关于打印性能的说明

打印到终端时出奇的慢！如果你在循环中调用 `println!` 之类的东西，
它很容易成为其它运行速度快的程序的瓶颈。你可以做两件事来为它提提速。

首先，你需要尽量减少实际“刷新”到终端的写入次数。_每次_调用 `println!`时，
它都会告诉系统刷新到终端，因为打印每个新行是很常见的。如果你不需要如此，
你可以使用 [`BufWriter`] 来包装一下 `stdout` 的句柄，它的默认缓存为 8 kB。
（当你想立即打印时，在 `BufWriter` 上调用 `.flush()` 即可。

```rust
use std::io::{self, Write};

let stdout = io::stdout(); // get the global stdout entity
let mut handle = io::BufWriter::new(stdout); // optional: wrap that handle in a buffer
writeln!(handle, "foo: {}", 42); // add `?` if you care about errors here
```

其次，为 `stdout`（或 `stderr`）申请一把锁并使用 `writeln!` 来直接打印很有用。
它会阻止系统不停地锁死和解锁 `stdout`。

```rust
use std::io::{self, Write};

let stdout = io::stdout(); // get the global stdout entity
let mut handle = stdout.lock(); // acquire a lock on it
writeln!(handle, "foo: {}", 42); // add `?` if you care about errors here
```

你也可以结合使用这两种方式。

[`BufWriter`]: https://doc.rust-lang.org/1.39.0/std/io/struct.BufWriter.html

## 显示进度条

一些 CLI 程序的运行时间很长，会花费几分钟甚至数小时。
如果你在编写这种程序，你可能希望向用户展示，其正在正常工作中。
因此，你需要打印出有用的状态更新信息，最好是使用易于使用的方式打印。

你可以使用 [indicatif] crate 来为你的程序添加进度条，这是一个简单的例子：

```rust,ignore
{{#include output-progressbar.rs:1:9}}
```

细节可查看 [indicatif 文档][indicatif docs] 和 [示例][indicatif examples]。

[indicatif]: https://crates.io/crates/indicatif
[indicatif docs]: https://docs.rs/indicatif
[indicatif examples]: https://github.com/mitsuhiko/indicatif/tree/master/examples

## 日志

为了更方便了解到我们的程序做了什么，我们需要给它添加上一些日志语句，这很简单。
但在长时间后，例半年后再运行这个程序时，日志就变得非常有用了。在某些方面来说，
日志的使用方法同 `println` 类似，只是它可以指定消息的重要性（级别）。
通常可以使用的级别包括 _error_, _warn_, _info_, _debug_, and _trace_
（_error_ 优先级最高，_trace_ 最低）。

只需这两样东西，你就可以给你的程序添加简单的日志功能：
[Log] 箱（其中包含以日志级别命名的宏）和一个 _adapter_，
它会将日志写到有用的地方。日志适配器的使用是十分灵活的：
例如，你可以不仅将日志写入终端，同时也可写入 [syslog] 或其它日志服务器。

[syslog]: https://en.wikipedia.org/wiki/Syslog

因为我们现在最关心的是写一个 CLI 程序，所以选一个易于使用的适配器 [env_logger]。
它之所以叫 "env" 日志记录器，因为你可以使用环境变量来指定，
程序中哪部分日志需要记录及记录哪种级别日志。
它会在你的日志信息前加上时间戳及所在模块信息。
由于库也可以使用 `log`，你也可以轻松地配置它们的日志输出。

[log]: https://crates.io/crates/log
[env_logger]: https://crates.io/crates/env_logger

这里有个简单的例子：

```rust,ignore
{{#include output-log.rs}}
```

如果你有 `src/bin/output-log.rs` 这个文件，在 Linux 或 MacOS 上，你可以运行：
```console
$ env RUST_LOG=info cargo run --bin output-log
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
     Running `target/debug/output-log`
[2018-11-30T20:25:52Z INFO  output_log] starting up
[2018-11-30T20:25:52Z WARN  output_log] oops, nothing implemented!
```

在 Windows PowerShell，运行：
```console
$ $env:RUST_LOG="info"
$ cargo run --bin output-log
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
     Running `target/debug/output-log.exe`
[2018-11-30T20:25:52Z INFO  output_log] starting up
[2018-11-30T20:25:52Z WARN  output_log] oops, nothing implemented!
```

在 Windows CMD，运行：
```console
$ set RUST_LOG=info
$ cargo run --bin output-log
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
     Running `target/debug/output-log.exe`
[2018-11-30T20:25:52Z INFO  output_log] starting up
[2018-11-30T20:25:52Z WARN  output_log] oops, nothing implemented!
```

`RUST_LOG` 是设置 log 的环境变量名。
`env_logger` 还包含一个构建器，因此你可以以编程的方式调整这些设置，
例如，还默认显示 _info_ 级别的日志。

还有很多可选的日志适配器，以及日志库或其扩展。
如果你确定你的应用将生成很多日志，请务必查看它们，以便解决发现的问题。

<aside>

**小技巧：**
经验表明，即使是稍有用的 CLI 程序也可能在未来几年的时间里被使用。
（特别是当它们是临时解决方案时）如果你的程序不能正常工作了，
有人（可能是未来的你）需要去查明其原因，
若能通过添加 `--verbose` 参数来获取更多额外的日志输出会大大减少调试时间。
[clap-verbosity-flag] 箱提供了使用 `structop`
将 `--verbose` 添加到项目中的便捷的方法。

[clap-verbosity-flag]: https://crates.io/crates/clap-verbosity-flag

</aside>
