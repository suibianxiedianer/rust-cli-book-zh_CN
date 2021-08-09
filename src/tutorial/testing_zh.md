# 测试

在数十年的软件开发中，人们发现了一个真理：未经测试的软件很难正常工作。
（许多人也会说，经过测试的软件也可能不工作，但我们都是乐观主义者，不是么）
所以，为了确保你的程序可如你预期的那样工作，最好先对其进行测试。

一种简单的方式是在 `README` 文件中写明你的软件将如何工作。
当你准备好进行一次新的发布时，再过一遍 `README` 的功能并确定其仍能正常工作。
你还可以写入你的程序如何应对错误的输入，使得这个测试变得更加严格。

另一个绝妙的主意是：在写代码前先写 `README`。

<aside>

**注：**
如果你没听过 [测试驱动开发] (TDD) 那么你最好去看一看它。

[测试驱动开发]: https://zh.wikipedia.org/wiki/Test-driven_development


</aside>

## 自动化测试

现在一切都妥了，但如果手动去完成测试，会耗费大量的时间。
现在，人们更喜欢让计算机来协助完成这些工作，下面我们来谈谈自动化测试。

Rust 有内建的测试框架，让我们试着写出一个测试吧：

```rust,ignore
#[test]
fn check_answer_validity() {
    assert_eq!(answer(), 42);
}
```

你可以把这段代码放在几乎任何文件，`cargo test` 将运行它。
这里的关键是 `#[test]` 属性。构建系统会寻找这类函数，并当作测试去运行它们，
确认他们不会出错（panic）。

<aside class="exercise">

**练习：**
让这个测试能正常工作。

完成后，测试的输出应该是这样的：

```text
running 1 test
test check_answer_validity ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

</aside>

现在我们已经了解了如何去编写测试项，那么下一个问题就是要测试什么？
如你所见，为函数写断言非常简单。但 CLI 程序通常不止一个功能函数！
更麻烦的是，它通常要处理用户的输入、读取文件并写入输出。

## 编写可测试的代码

测试功能有两种互补的方法：
测试构建成完整程序的功能小单元，叫作“单元测试”。
还有就是从外部测试最终的程序，称为“黑盒测试”或“集成测试”。
让我们先从单元测试开始。

为了弄清楚我们要测试什么，让我们先看看我们程序的功能。
`grrs` 应该打印出，文件中包含符合匹配字符串的行。
所以，让我们给**匹配**功能写一个单元测试：
我们希望确保我们最重要的逻辑部分能正常工作，
并且以一种不依赖于任何需要设置代码、变量的方式实现（比如，处理 CLI 参数）。

回到我们的 `grrs` 的[第一个实现](impl-draft_zh.md)，
当时我们在 `main` 函数中添加了如下代码块：

```rust,ignore
// ...
for line in content.lines() {
    if line.contains(&args.pattern) {
        println!("{}", line);
    }
}
```

遗憾的是，这样的代码很难测试。
首先，因为它在 `main` 函数中，所以我们不能简单的直接调用它。
而把这段代码移动到一个函数中，就可以很简单地解决这个问题了：

```rust,no_run
fn find_matches(content: &str, pattern: &str) {
    for line in content.lines() {
        if line.contains(pattern) {
            println!("{}", line);
        }
    }
}
```

现在我们可以在测试中调用这个函数了，让我们来看一下它的输出是什么：

```rust,ignore
#[test]
fn find_a_match() {
    find_matches("lorem ipsum\ndolor sit amet", "lorem");
    assert_eq!( // uhhhh
```

或者，我们还不能……现在，`find_matches` 会直接将输出打印到 `stdout`，比如在终端。
我们不能简单地在测试中捕获它的输出！
这是在实现功能后再编写测试时经常遇到的问题：
我们编写了一个牢固地集成在它所使用的上下文中的函数。

<aside class="note">

**注：**
这在编写一个小的 CLI 程序时问题不大。没必要去测试每个功能！
但重要的是要想清楚，哪些代码是需要去编写单元测试的。
虽然下面我们可以轻易的将这个函数修改为可测试的，但事情并非总是如此美好。

</aside>

那么，我们怎样才能让这个函数变得可测呢？我们得能获取它的输出。
Rust 的标准库有一些简单的抽象来处理 I/O，在这里我们将使用 [`std::io::Write`]。
这是一个 [trait][trpl-traits]，
它可以抽象我们可以写入的事物，包括字符串和标准输出。

[trpl-traits]: https://doc.rust-lang.org/book/ch10-02-traits.html
[`std::io::Write`]: https://doc.rust-lang.org/1.39.0/std/io/trait.Write.html

如果你是第一次在 Rust 中看到 “trait” 这个词，也没关系。
特性（trait）是 Rust 最强大的特性之一。
你可以把它看成 Java 中的接口，或 Haskell 的 type classes（看你了解哪个）。
它允许你抽象可由不同类型共享的行为。
使用 trait 的代码可以以非常通用和灵活的方式来实现功能，
但这也意味着它可能难以阅读。
但请不要被它吓到：即使是 Rust 中的老手也不一定能马上理解通用代码的作用。
在这种情况下，考虑具体用途会有所帮助。
比如，现在，我们抽象的行为是“写入”。实现了（“impl”）了它的类型有：
终端的标准输出、文件、内存中的缓存或 TCP 网络连接。
（在[`std::io::Write` 文档][`std::io::Write`] 中下翻可查看实现此 trait 的列表）

有了这些知识，让我们修改函数以便接受第三个参数。它应该是一个实现了 `Write`
的类型。这样，我们就可以在测试中提供一个简单的字符串并对其进行断言。
这是我们的 `find_matches` 的新版本：

```rust,ignore
{{#include testing/src/main.rs:24:30}}
```

第三个参数是 `mut writer`，即一个名为 `writer` 的可变变量。
它的类型是 `impl std::io::Write`，你可理解为“实现了 `Write` trait 的任意类型”。
还要注意，我们使用 `writeln!(writer, …)` 替换了之前的 `println!(…)`。
`println!` 的行为类似于 `writeln!`，只不过它总是写入到标准输出里。

现在我们可以测试它的输出了：

```rust,ignore
{{#include testing/src/main.rs:32:37}}
```

现在我们必须修改 `main` 函数中对 `find_matches` 的调用，给它加
[`&mut std::io::stdout()`][stdout] 作为第三个参数。
下面是修改完成之后，使用新版 `find_matches` 的 `mian` 函数：

```rust,ignore
{{#include testing/src/main.rs:14:22}}
```

[stdout]: https://doc.rust-lang.org/1.39.0/std/io/fn.stdout.html

<aside class="note">

**注：**
由于 `stdout` 接收的是字节（而不是字符串）,
所以我们使用 `std::io::Write` 而不是 `std::fmt::Write`。
因此，我们在测试中传入了一个空的 vector 作为 “writer”
（它会被判断为 `Vec<u8>`），在 `assert_eq` 中我们使用 `b"foo"`。
（`b` 使得它被当作 _byte 字节_，即其类型为 `&[u8]` 而非 `&str`。

</aside>

<aside class="note">

**注：**
我们也可以让这个函数（`find_matches`）返回一个 `String`，
但这样就改变了它的行为。即它不会直接将结果写入终端，
而是将所有的匹配行收集为一个字符串，并在最后返回一个结果。

</aside>

<aside class="exercise">

**练习：**
[`writeln!`] 返回值是 [`io::Result`]，因为写入可能会失败，
例如缓存被用尽又无法申请新空间时。
请在 `find_matches` 添加上错误处理。

[`writeln!`]: https://doc.rust-lang.org/1.39.0/std/macro.writeln.html
[`io::Result`]: https://doc.rust-lang.org/1.39.0/std/io/type.Result.html

</aside>

我们刚刚学习了如何使一段代码变得易于测试。我们做了：

1. 找到程序中的一段核心代码，
2. 将它封装到一个独立的函数中，
3. 让它的使用方法变得更为灵活。

尽管我们的目标仅仅是让它具有可测试性，
但我们最终得到了一段符合 Rust 语言风格且可被复用的代码，这非常棒！

## 将代码拆分为库和二进制

这里我们还需要做一些事情。到目前为止我们将所有的代码写到了 `src/main.rs` 里。
这意味着我们的项目只会编译成一个单独的二进制程序。
但我们也可以将我们的代码作为库提供，像这样：

1. 将 `find_matches` 函数放到新的 `src/lib.rs` 文件中。
2. 在函数名前（`fn`）添加上 `pub` 标记（现在函数为 `pub fn find_matches`）
   以便这个库的使用者可以访问这个函数。
3. 将 `src/main.rs` 中的 `find_matches` 函数移除。
4. 在 `fn main` 中使用 `grrs::find_matches` 去调用我们在库中的 `find_matches`
   函数。

Rust 处理项目的方式非常灵活，尽早地考虑清楚哪些功能要放到库中是个好主意。
例如，你可以考虑先为用于特定程序的逻辑编写一个库，然后像调用任意其它库一样，
在你的 CLI 程序中使用它。又或者，若你的项目会生成多个二进制程序，
你可以将其通用的功能放到库里，提高代码的复用性。

<aside class="note">

**注：**
如果将所有的代码都放到 `src/main.rs` 里，这会让我们的代码变得难以阅读。
查看 [module system] 可以帮助你去构建、组织你的代码结构。

[module system]: https://doc.rust-lang.org/1.39.0/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html

</aside>


## 通过运行 CLI 程序来测试它们

到目前为止，我们已经测试了我们程序中的_业务逻辑_，即 `find_matches` 函数。
这是非常有价值的，并且是迈向实现经过良好测试的代码的第一步。
（通常，这类测试被称为“单元测试”）

但还有许多代码是我们没有测试到的，因为：
我们写的一切都是为了与外界打交道！
想像一下，在你写 `main` 函数时，不小心留下了一段硬编码的路径字符串，
软件运行时会使用它而非用户提供的参数！
我们也需要编写这类的测试！（这个级别的测试一般被称为“集成测试”或“系统测试”）

从本质上讲，我们仍要编写函数并使用 `#[test]` 注释它们。
现在的问题是我们要在这些函数中做什么？
例如，我们想像运行一个普通程序一样使用我们项目的主程序。
我们还要将这些测试放入到一个全新的路径下：`tests/cli.rs`。

<aside>

**注：**
依约定，`cargo` 会在 `tests/` 目录中查找集成测试。
同样，它会在 `benches/` 目录寻找 benchmarks，在 `examples/` 中寻找示例。
这些约定也扩展到你的主要源代码：
库有一个 `src/libs.rs` 文件，主程序是 `src/main.rs`，
如果这里有多个二进制程序，`cargo` 期望它们放在 `src/bin/<name>.rs` 中。
遵循这些约定可以让你的代码对于习惯阅读 Rust 代码的人更为友好。

</aside>

回想一下，`grrs` 是一个在文件中搜索字符串的小工具。
我们已经测试过了查找匹配项功能。让我们再想想还能测试哪些功能。

这里我想到了一些。

- 如果文件不存在会怎样？
- 当没有搜索到匹配项时返回什么？
- 当我们少写一个（或都没写）参数时，我们的程序会以错误状态退出？

这些都是有效的测试用例。
此外，我们还应该有一个成功的测试用例，即程序正常运行且找到至少一个匹配并打印。

为了让这类的测试更容易，我们将使用到 [`assert_cmd`] 箱。
它提供了许多简洁的帮助程序，可以让我们运行我们的主程序并查看它的行为。
此外，我们还将添加 [`predicates`] 箱，
来帮助我们为 `assert_cmd` 的测试项编写断言（且具有很棒的错误消息）。
我们不会将这些依赖放到程序的主依赖中，
而是放到 `Cargo.toml` 的 `dev dependencies` 部分。
它们只会在开发时被使用到，而使用时则不会。
not when using it.

```toml
{{#include testing/Cargo.toml:11:13}}
```

[`assert_cmd`]: https://docs.rs/assert_cmd
[`predicates`]: https://docs.rs/predicates

设置完成后，让我们来创建我们的 `tests/cli.rs` 文件：

```rust,ignore
{{#include testing/tests/cli.rs:1:15}}
```

你可以像我们之前做测试时一样，通过运行 `cargo test` 来运行这个测试。
在第一次运行时可能会稍慢，因为我们要编译出项目的主程序，它在
`Command::cargo_bin("grrs")` 被调用。

## 生成测试文件

我们刚编写的测试，只会在检查我们的程序的输入的文件参数不存在时输出的错误信息。
这是很重要的一个测试项，却不是最重要的：
让我们测试下，正确运行程序并打印出文件中匹配项的用例。

首先我们要有一个已知内容的文件，这样我们才能确认正确的输出是什么从而进行测试。
当然我们可以在项目中添加一个文件专门用来进行测试，或者也可以创建临时文件来测试。
在本教程中，我们选择后面一种做法。因为它相对更为灵活且也适用于其它测试用例；
比如，当你要测试去修改一个文件的时候。

为了生成这些临时文件，我们要使用到 [`assert_fs`] 箱，
让我们把它添加到 `Cargo.toml` 的 `dev-dependencies` 中。

```toml
{{#include testing/Cargo.toml:14}}
```

[`assert_fs`]: https://docs.rs/assert_fs

这就是新的测试用例，首先去创建一个临时文件（并获取到它的路径），
然后在里面填充一些内容，再去运行我们的程序来检查我们能否获得正确的输出。
当执行完这个代码后，`file` 将自动被删除。

```rust,ignore
{{#include testing/tests/cli.rs:17:32}}
```

<aside class="exercise">

**练习：**
添加一个传入的匹配项为空字符串的集成测试，并按需去调整程序。

</aside>

## 去测试什么？
## What to test?

虽然编写集成测试很有意思，但毕竟编写测试是要消耗时间的，
而且当你的程序行为有所变动时可能也需要去更新这些测试。
为了让我们花费的时间变得更有意义，我们应该问下自己，我们要测试什么？

一般来讲，为用户可以观察到的所有类型的行为编写集成测试是一个好主意。
这意味着你不需要去涵盖所有的边界情况：通常会有不同类型的示例，
再依赖于单元测试即可覆盖边界情况。

同样，尝试去测试你不能掌控的东西，并不是一个好主意。
测试 `--help` 的确切输出布局，会是一个坏主意，
相反，你可能只想检查其中某些元素是否存在。

根据程序的特性，你还可以尝试添加更多的测试技术。
比如，当你尝试去测试到所有的边界情况时，如果你提取了你程序的一部分，
并且发现你已经写了许多作为单元测试的示例情景，你可以看看 [`proptest`]。
如果你的程序会使用任意一个文件并解析它们，请试着写一个 [fuzzer]
来查找边界条件下的 bugs。

[`proptest`]: https://docs.rs/proptest
[fuzzer]: https://rust-fuzz.github.io/book/introduction.html

<aside>

**注：**
你可以在[这里][src] 看到完整、可运行的源码。

[src]: https://github.com/rust-cli/book/tree/master/src/tutorial/testing

</aside>
