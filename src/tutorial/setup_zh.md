# 开始项目

首先在你的电脑上[安装 Rust]（如果还没有，这只需大概几分钟）。
然后，打开终端，切换到你的工作目录，程序源码将放置在这里。

[安装 Rust]: https://www.rust-lang.org/tools/install

在你的工作目录里，运行 `cargo new grrs` 来生成软件项目。
进入 `grrs` 目录，你会看到典型的 Rust 项目目录结构：

- `Cargo.toml` 里包含了我们项目所有的元数据，包括我们使用依赖/外部库列表。
- `src/main.rs` 是我们程序的二进制入口文件（主程序）。

在 `grrs` 目录下执行 `cargo run`，若程序打印出 `Hello World`，那就大功告成了！

## 它应该是这样的

```console
$ cargo new grrs
     Created binary (application) `grrs` package
$ cd grrs/
$ cargo run
   Compiling grrs v0.1.0 (/Users/pascal/code/grrs)
    Finished dev [unoptimized + debuginfo] target(s) in 0.70s
     Running `target/debug/grrs`
Hello, world!
```
