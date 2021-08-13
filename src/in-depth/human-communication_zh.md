# 与人交互

你需要先去阅读[输出][output]这一章节。
它介绍了如何在终端中写入输入，而本节将谈谈写入哪些输出。

[output]: ../tutorial/output_zh.html

## 当一切都正常时

即使一切正常，报告程序的进度也很有用。但要尽量提供简洁有用的信息。
不要在日志中使用过于专业的术语。
请记住，此时程序没有崩溃，所以用户没必要去查找错误。

最重要的是，要保证简洁，使用相同的前缀和语句结构以便日志更易于阅读。

要试着让你的程序的输出讲清楚它在做什么，及对用户有什么影响。
这可能涉及显示其步骤的时间线，甚至在长时间运行的程序中显示一个进度条和指示器。
用户在任何时候都不应该有程序在做一些他们无法理解的神秘的事情的感觉。

## 当很难说清楚发生了什么时

在传达非名义状态时，保持一致很重要。
与无日志应用相比，不严格遵循日志记录级别且生成大量日志的程序，
其提供的信息量会相同，甚至于更少。

所以，定义与之相关的事件和消息的严重性很重要；然后为它们使用一致的日志级别。
使用这种方法使得用户可以使用 `--verbose`
或设置环境变量（如 `RUST_LOG`）来得到大量的 log 信息。

通常会使用到 `log` 箱来 [defines][log-levels] 这些 log 等级（按严重级别排列）：

- trace
- debug
- info
- warning
- error

将 _info_ 设置为默认的日志级别来输出信息是个好主意。
（一些应用更倾向于更为安静的输出——默认只输出警告和错误信息）

此外，保持使用相同的前缀和日志语句格式，总会是最好的选择！
这样可以更方便地使用 `grep` 命令来筛选日志。
在一条日志消息中，需要提供足够的信息来方便进行日志筛选，但也不要太过详尽、冗长。

[log-levels]: https://docs.rs/log/0.4.4/log/enum.Level.html

### 日志句法示例

```console
error: could not find `Cargo.toml` in `/home/you/project/`
```

```console
=> Downloading repository index
=> Downloading packages...
```

[wasm-pack] 中的日志：

```console
 [1/7] Adding WASM target...
 [2/7] Compiling to WASM...
 [3/7] Creating a pkg directory...
 [4/7] Writing a package.json...
 > [WARN]: Field `description` is missing from Cargo.toml. It is not necessary, but recommended
 > [WARN]: Field `repository` is missing from Cargo.toml. It is not necessary, but recommended
 > [WARN]: Field `license` is missing from Cargo.toml. It is not necessary, but recommended
 [5/7] Copying over your README...
 > [WARN]: origin crate has no README
 [6/7] Installing WASM-bindgen...
 > [INFO]: wasm-bindgen already installed
 [7/7] Running WASM-bindgen...
 Done in 1 second
```

## 崩溃的时候

一个常被忽略的点是——程序在崩溃的时候也要输出一些东西。
在 Rust 中，崩溃通常是程序的 `panics`（“可控崩溃”不“操作系统直接杀死进程”不同）。
默认情况下，当 panic 发生时，“panic handler” 会在终端中打印一些有用信息。

比如，你使用 `cargo new --bin foo` 生成一个新的二进制项目，
并在 `fn main` 中添加一句 `panic!("Hello World")`，当你运行程序时：

```console
thread 'main' panicked at 'Hello, world!', src/main.rs:2:5
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

这对于你，开发者来说，是有用的信息。（哈哈：你的程序在 `main.rs` 第二行崩溃了）
但对于那些没源码的人，它没什么价值，而且事实上，它只会让人感到更为困惑。
所以，添加一个更为面向终端用户的自定义 panic 处理器是一个好主意！

[human-panic] 就是一个做这样的库。把它添加到你的 CLI 项目中，导入，
并在你的 `main` 函数的开头调用 `setup_panic!()` 宏：

```rust,ignore
use human_panic::setup_panic;

fn main() {
   setup_panic!();

   panic!("Hello world")
}
```

现在它将显示更为友好的信息，并告诉用户他们能做什么：

```console
Well, this is embarrassing.

foo had a problem and crashed. To help us diagnose the problem you can send us a crash report.

We have generated a report file at "/var/folders/n3/dkk459k908lcmkzwcmq0tcv00000gn/T/report-738e1bec-5585-47a4-8158-f1f7227f0168.toml". Submit an issue or email with the subject of "foo Crash Report" and include the report as an attachment.

- Authors: Your Name <your.name@example.com>

We take privacy seriously, and do not perform any automated error collection. In order to improve the software, we rely on people to submit reports.

Thank you kindly!
```

[human-panic]: https://crates.io/crates/human-panic
[wasm-pack]: https://crates.io/crates/wasm-pack
