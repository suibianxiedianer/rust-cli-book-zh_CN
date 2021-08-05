# 15 分钟编写一个命令行应用来学习 Rust

本教程将引导你使用 [Rust] 编写 CLI（命令行界面）应用。
只需 15 分钟即可让你得到一个可正常运行的程序（大概 1.3 章之前）。
之后我们将继续调整程序，直到它可以被当作一个工具来发布。

[Rust]: https://rust-lang.org/

你将学到如何开始所需的所有基本知识，及如何去寻找更多有用信息。
当然，你可随意跳过当前你不需要了解的章节，或之后再翻回查看。

<aside>

**先决条件：**
本教程并不能替代编程的一般性介绍，你最好了解、熟悉一些常见的概念。
同样，你应该可以熟练的使用命令行或终端。如果你会使用其它的语言，
那么这对于你接触，学习 Rust 会很有帮助。 

**获取帮助：**
如果你在任何时候对所使用的功能感到不解或困惑，请查阅 Rust 提供的官方文档，
首先是这本《The Rust Programming Language》。一般安装 Rust 时也会安装它
（`rustup doc`），或者你也可以在线查看 [doc.rust-lang.org]。

[doc.rust-lang.org]: https://doc.rust-lang.org

也非常欢迎你来社区提问——Rust 社区以友好和乐于助人著称。
在[社区页面]你可以看到人们关于 Rust 的讨论的列表。
You are also very welcome to ask questions –

[社区页面]: https://www.rust-lang.org/community

</aside>

你想要写一个什么样的项目呢？不如我们先从一个简单的开始吧：让我们写一个简单的
`grep`。我们给这个工具一个字符串和一个文件路径，它将打印出每个包含所查字符串的行。
不如就叫它 `grrs` 吧。

最后，我们想让它像这样工作：

```console
$ cat test.txt
foo: 10
bar: 20
baz: 30
$ grrs foo test.txt
foo: 10
$ grrs --help
[some help text explaining the available options]
```

<aside class="note">

**注意：**
本书是基于 [Rust 2018] 来创作的，同时，这些代码同样适用于 Rust 2015，
只需做一点点调整，例如添加上 `extern crate foo` 声明。

确保你运行的是 Rust 1.31.0（或更高版本），同时在你的 `Cargo.toml` 文件中，
在 `[package]` 段添加上 `edition = "2018"`。

[Rust 2018]: https://doc.rust-lang.org/edition-guide/index.html

</aside>
