# 信号处理

像命令行程序这种进程需要接受操作系统发出的信号并做出相应的反应。
最常见的就是 <kbd>Ctrl</kbd>+<kbd>C</kbd> 了，这是典型的终止进程的信号。
在 Rust 程序中处理信号，你需要考虑如何获取这些信号及如何做出反应。

<aside>

**注：**
如果你的程序不需要优雅地关闭，那么默认的处理方式就很好（比如，立即退出，
并让系统清理资源如打开的文件的句柄）。
在这种情况下，就无须理会本章告诉你的操作了！

然后，对于一些需要做自我清理工作的程序，本章讲的东西就非常有用了!
比如，如果你的程序需要正确地关闭网络连接（向另一端发送结束语句），
移除临时文件或重置系统设置等，请继续往下看。

</aside>

## 操作系统之间的差异

在 Unix 系统（比如 Linux、macOS 和 FreeBSD），进程可以接受 [signals]。
它可以以默认的方式（系统提供）来接收信号并以默认的方式处理它，
或者直接不理会这个信号。

[signals]: https://manpages.ubuntu.com/manpages/bionic/en/man7/signal.7.html

Windows 没有信号。你可以使用 [Console Handlers] 来定义某事件发生后的回调。
还有 [structured exception handling]，它处理所有各种类型的系统异常，
如被零除，无效访问异常，堆栈溢出等。

[Console Handlers]: https://docs.microsoft.com/en-us/windows/console/console-control-handlers
[structured exception handling]: https://docs.microsoft.com/en-us/windows/desktop/debug/structured-exception-handling

## 首先：处理 Ctrl+C

[ctrlc] 箱如其名：它允许你在用户按下 <kbd>Ctrl</kbd>+<kbd>C</kbd> 时，
去做出相应的反应，且它是跨平台的。其使用方法如下：

[ctrlc]: https://crates.io/crates/ctrlc

```rust,ignore
{{#include signals-ctrlc.rs}}
```

当然，这没啥意义：它只是打印出了一段信息，而并没有退出程序。

在现实的程序中，在信号处理时去设置一个变量，
并在程序中各个位置去检查，会是一个好主意。
比如，你可以在信号处理程序中设置一个 `Arc<AtomicBool>`
（一个可在线程中共享的 boolean），并在一个无限循环中，或在等待线程时，
周期性地检查它的值，当它为真时退出程序。

## 处理其它类型的信号

[ctrlc] 箱只会处理 <kbd>Ctrl</kbd>+<kbd>C</kbd>，或者在 Unix 系统中，称为
`SIGINT`（中断信号）。[signal-hook] 可以去处理更多的 Unix 信号。
在 [this blog post][signal-hook-post] 中描述了它的设计原理，
且它是目前社区里支持最为广泛的库。

一个简单的示例：

```rust,ignore
{{#include signals-hooked.rs}}
```

[signal-hook-post]: https://vorner.github.io/2018/06/28/signal-hook.html

## 使用通道（channels)

除了设置一个变量并在程序其它部分去检查它，你还可以使用通道：
创建一个通道，在信号处理器接收到信号后，向里面发送一个值。
在你的程序代码中，使用此通道和其他通道作为线程之间的同步点。
像这样使用 [crossbeam-channel]：

[crossbeam-channel]: https://crates.io/crates/crossbeam-channel

```rust,ignore
{{#include signals-channels.rs}}
```

## 使用 futures 和 streams

如果你使用 [tokio]，那你应该已经写了一个使用异步编程的事件驱动型的程序。
除了直接使用 crossbeam 的通道之外，
你还可以启用 signal-hook 的 `tokio-support` feture。
它允许你在 signal-hook 的 `Signals` 类上调用 [`.into_async()`]
来获得一个实现了 `futures::Stream` 的新类型。

[signal-hook]: https://crates.io/crates/signal-hook
[tokio]: https://tokio.rs/
[`.into_async()`]: https://docs.rs/signal-hook/0.1.6/signal_hook/iterator/struct.Signals.html#method.into_async

## 当处理 Ctrl+C 时又接收到新的 Ctrl+C 时怎么办

大部分用户会按下 <kbd>Ctrl</kbd>+<kbd>C</kbd>，
等待你的程序退出（也许需要几秒钟）或告诉他们接下来要怎么办。
如果没效果，他们会再次按下 <kbd>Ctrl</kbd>+<kbd>C</kbd>。
一般情况下，此时程序会立即退出！
