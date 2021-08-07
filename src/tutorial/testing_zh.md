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

## ?????
## Making your code testable

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

We've just seen how to make this piece of code easily testable.
We have

1. identified one of the core pieces of our application,
2. put it into its own function,
3. and made it more flexible.

Even though the goal was to make it testable,
the result we ended up with
is actually a very idiomatic and reusable piece of Rust code.
That's awesome!

## Splitting your code into library and binary targets

We can do one more thing here.
So far we've put everything we wrote into the `src/main.rs` file.
This means our current project produces a single binary.
But we can also make our code available as a library, like this:

1. Put the `find_matches` function into a new `src/lib.rs`.
2. Add a `pub` in front of the `fn` (so it's `pub fn find_matches`)
   to make it something that users of our library can access.
3. Remove `find_matches` from `src/main.rs`.
4. In the `fn main`, prepend the call to `find_matches` with `grrs::`,
   so it's now `grrs::find_matches(…)`.
   This means it uses the function from the library we just wrote!

The way Rust deals with projects is quite flexible
and it's a good idea to think about
what to put into the library part of your crate early on.
You can for example think about writing a library
for your application-specific logic first
and then use it in your CLI just like any other library.
Or, if your project has multiple binaries,
you can put the common functionality into the library part of that crate.

<aside class="note">

**Note:**
Speaking of putting everything into a `src/main.rs`:
If we continue to do that,
it'll become difficult to read.
The [module system] can help you structure and organize your code.

[module system]: https://doc.rust-lang.org/1.39.0/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html

</aside>


## Testing CLI applications by running them

Thus far, we've gone out of our way
to test the _business logic_ of our application,
which turned out to be the `find_matches` function.
This is very valuable
and is a great first step
towards a well-tested code base.
(Usually, these kinds of tests are called "unit tests".)

There is a lot of code we aren't testing, though:
Everything that we wrote to deal with the outside world!
Imagine you wrote the main function,
but accidentally left in a hard-coded string
instead of using the argument of the user-supplied path.
We should write tests for that, too!
(This level of testing is often called
"integration testing", or "system testing".)

At its core,
we are still writing functions
and annotating them with `#[test]`.
It's just a matter of what we do inside these functions.
For example, we'll want to use the main binary of our project,
and run it like a regular program.
We will also put these tests into a new file in a new directory:
`tests/cli.rs`.

<aside>

**Aside:**
By convention,
`cargo` will look for integration tests in the `tests/` directory.
Similarly,
it will look for benchmarks in `benches/`,
and examples in `examples/`.
These conventions also extend to your main source code:
libraries have a `src/lib.rs` file,
the main binary is `src/main.rs`,
or, if there are multiple binaries,
cargo expects them to be in `src/bin/<name>.rs`.
Following these conventions will make your code base more discoverable
by people used to reading Rust code.

</aside>

To recall,
`grrs` is a small tool that searches for a string in a file.
We have previously tested that we can find a match.
Let's think about what other functionality we can test.

Here is what I came up with.

- What happens when the file doesn't exist?
- What is the output when there is no match?
- Does our program exit with an error when we forget one (or both) arguments?

These are all valid test cases.
Additionally,
we should also include one test case
for the "happy path",
i.e., we found at least one match
and we print it.

To make these kinds of tests easier,
we're going to use the [`assert_cmd`] crate.
It has a bunch of neat helpers
that allow us to run our main binary
and see how it behaves.
Further,
we'll also add the [`predicates`] crate
which helps us write assertions
that `assert_cmd` can test against
(and that have great error messages).
We'll add those dependencies not to the main list,
but to a "dev dependencies" section in our `Cargo.toml`.
They are only required when developing the crate,
not when using it.

```toml
{{#include testing/Cargo.toml:11:13}}
```

[`assert_cmd`]: https://docs.rs/assert_cmd
[`predicates`]: https://docs.rs/predicates

This sounds like a lot of setup.
Nevertheless –
let's dive right in
and create our `tests/cli.rs` file:

```rust,ignore
{{#include testing/tests/cli.rs:1:15}}
```

You can run this test with
`cargo test`,
just the tests we wrote above.
It might take a little longer the first time,
as `Command::cargo_bin("grrs")` needs to compile your main binary.

## Generating test files

The test we've just seen only checks that our program writes an error message
when the input file doesn't exist.
That's an important test to have,
but maybe not the most important one:
Let's now test that we will actually print the matches we found in a file!

We'll need to have a file whose content we know,
so that we can know what our program _should_ return
and check this expectation in our code.
One idea might be to add a file to the project with custom content
and use that in our tests.
Another would be to create temporary files in our tests.
For this tutorial,
we'll have a look at the latter approach.
Mainly, because it is more flexible and will also work in other cases;
for example, when you are testing programs that change the files.

To create these temporary files,
we'll be using the [`assert_fs`] crate.
Let's add it to the `dev-dependencies` in our `Cargo.toml`:

```toml
{{#include testing/Cargo.toml:14}}
```

[`assert_fs`]: https://docs.rs/assert_fs

Here is a new test case
(that you can write below the other one)
that first creates a temp file
(a "named" one so we can get its path),
fills it with some text,
and then runs our program
to see if we get the correct output.
When the `file` goes out of scope
(at the end of the function),
the actual temporary file will automatically get deleted.

```rust,ignore
{{#include testing/tests/cli.rs:17:32}}
```

<aside class="exercise">

**Exercise for the reader:**
Add integration tests for passing an empty string as pattern.
Adjust the program as needed.

</aside>

## What to test?

While it can certainly be fun to write integration tests,
it will also take some time to write them,
as well as to update them when your application's behavior changes.
To make sure you use your time wisely,
you should ask yourself what you should test.

In general it's a good idea to write integration tests
for all types of behavior that a user can observe.
That means that you don't need to cover all edge cases:
It usually suffices to have examples for the different types
and rely on unit tests to cover the edge cases.

It is also a good idea not to focus your tests on things you can't actively control.
It would be a bad idea to test the exact layout of `--help`
as it is generated for you.
Instead, you might just want to check that certain elements are present.

Depending on the nature of your program,
you can also try to add more testing techniques.
For example,
if you have extracted parts of your program
and find yourself writing a lot of example cases as unit tests
while trying to come up with all the edge cases,
you should look into [`proptest`].
If you have a program which consumes arbitrary files and parses them,
try to write a [fuzzer] to find bugs in edge cases.

[`proptest`]: https://docs.rs/proptest
[fuzzer]: https://rust-fuzz.github.io/book/introduction.html

<aside>

**Aside:**
You can find the full, runnable source code used in this chapter
[in this book's repository][src].

[src]: https://github.com/rust-cli/book/tree/master/src/tutorial/testing

</aside>
