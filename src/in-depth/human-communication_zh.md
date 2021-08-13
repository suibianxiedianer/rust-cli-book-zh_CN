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

It's a good idea to think of _info_ as the default log level.
Use it for, well, informative output.
(Some applications that lean towards a more quiet output style
might only show warnings and errors by default.)

Additionally,
it's always a good idea to use similar prefixes
and sentence structure across log messages,
making it easy to use a tool like `grep` to filter for them.
A message should provide enough context by itself
to be useful in a filtered log
while not being *too* verbose at the same time.

[log-levels]: https://docs.rs/log/0.4.4/log/enum.Level.html

### Example log statements

```console
error: could not find `Cargo.toml` in `/home/you/project/`
```

```console
=> Downloading repository index
=> Downloading packages...
```

The following log output is taken from [wasm-pack]:

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

## When panicking

One aspect often forgotten is that
your program also outputs something when it crashes.
In Rust, "crashes" are most often "panics"
(i.e., "controlled crashing"
in contrast to "the operating system killed the process").
By default,
when a panic occurs,
a "panic handler" will print some information to the console.

For example,
if you create a new binary project
with `cargo new --bin foo`
and replace the content of `fn main` with `panic!("Hello World")`,
you get this when you run your program:

```console
thread 'main' panicked at 'Hello, world!', src/main.rs:2:5
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

This is useful information to you, the developer.
(Surprise: the program crashed because of line 2 in your `main.rs` file).
But for a user who doesn't even have access to the source code,
this is not very valuable.
In fact, it most likely is just confusing.
That's why it's a good idea to add a custom panic handler,
that provides a bit more end-user focused output.

One library that does just that is called [human-panic].
To add it to your CLI project,
you import it
and call the `setup_panic!()` macro
at the beginning of your `main` function:

```rust,ignore
use human_panic::setup_panic;

fn main() {
   setup_panic!();

   panic!("Hello world")
}
```

This will now show a very friendly message,
and tells the user what they can do:

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
