# 退出状态码

程序不总是能成功地运行。当发生错误时，你需要确保正确地发出必要的信息。
此外如 [telling the user about errors](human-communication.html) 中所说，
在大部分系统中，当一个进程退出，它会发出一个退出状态码
（在大部分平台上是一个 0 至 255 的数字）。
你应该为你的程序发出正确的退出码。
比如，当你的程序成功运行后，它应该生成 `0` 的退出码。

但当发生错误时，它会变得更复杂一些。
大多数情况下，许多工具在发生一般性错误时会以 `1` 为退出码。
目前，Rust 为 panicked 的进程设置了 `101` 的退出状态码。
除此之外，人们在他们的程序中做了许多事情。

所以，要如何去做呢？BSD 生态系统为其退出码做了一个通用的定义
（你可以在[这里][`sysexits.h`]找到它们）。
Rust 的 [`exitcode`] 库也提供了一样的代码，且你可在你的程序中使用。
请参阅其 API 文档以了解其用法。

当你在你的 `Cargo.toml` 中添加 `exitcode` 依赖后，你可以这样使用：

```rust,ignore
fn main() {
    // ...actual work...
    match result {
        Ok(_) => {
            println!("Done!");
            std::process::exit(exitcode::OK);
        }
        Err(CustomError::CantReadConfig(e)) => {
            eprintln!("Error: {}", e);
            std::process::exit(exitcode::CONFIG);
        }
        Err(e) => {
            eprintln!("Error: {}", e);
            std::process::exit(exitcode::DATAERR);
        }
    }
}
```


[`exitcode`]: https://crates.io/crates/exitcode
[`sysexits.h`]: https://www.freebsd.org/cgi/man.cgi?query=sysexits&apropos=0&sektion=0&manpath=FreeBSD+11.2-stable&arch=default&format=html
