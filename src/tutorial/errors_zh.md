# 更好地反馈错误

我们都知道，错误有时是无可避免的。与其它许多语言不同，
在使用 Rust 时很难不注意和而对这个现实：Rust 没有异常，
所有可能的错误状态通常都编码在函数的返回类型中。

## 结果

像 [`read_to_string`] 这样的函数不会返回字符串。它返回的是一个 [`Result`]，
里面可能是一个 `String` 或是其它错误类型（这里是 [`std::io::Error`]）。

[`read_to_string`]: https://doc.rust-lang.org/1.39.0/std/fs/fn.read_to_string.html
[`Result`]: https://doc.rust-lang.org/1.39.0/std/result/index.html
[`std::io::Error`]: https://doc.rust-lang.org/1.39.0/std/io/type.Result.html

那么如何得知是哪种类型呢？因为 `Result` 亦是 `enum` 类型，
可以使用 `match` 去检查里面是哪种变体：

```rust,no_run
let result = std::fs::read_to_string("test.txt");
match result {
    Ok(content) => { println!("File content: {}", content); }
    Err(error) => { println!("Oh noes: {}", error); }
}
```

<aside>

**注：**
若不清楚 enum 是什么或在 Rust 中如何使用它，可查看
[enums](https://doc.rust-lang.org/1.39.0/book/ch06-00-enums.html)。

</aside>

## 展开

现在，我们可以访问文件的内容，但在 `match` 代码块后无法对它做任何事情。
因此，我们需要以某种方式处理错误的情况。
难点在于，`match` 代码块的所有分支都应返回一个相同的类型。
好在这儿有个巧妙的技巧可以解决这个问题：

```rust,no_run
let result = std::fs::read_to_string("test.txt");
let content = match result {
    Ok(content) => { content },
    Err(error) => { panic!("Can't deal with {}, just exit here", error); }
};
println!("file content: {}", content);
```

我们可以在 match 代码块后使用 `content`。如果 `result` 是一个错误，
字符串就不存在。但好在，程序会在我们使用 `content` 之前就退出了。

这种做法看上去有些极端，却是十分实用的。如果你的程序需要读取一个文件，
且在文件不存在时无法执行任何操作，那么退出是十分合理、有效的选择。
在 `Result` 中还有一个快捷方法，叫 `unwrap`：

```rust,no_run
let content = std::fs::read_to_string("test.txt").unwrap();
```

## 无须 panic

当然，退出程序并非处理错误的唯一办法。除 `!panic` 之外，实现 `return` 也很简单：

```rust,no_run
# fn main() -> Result<(), Box<std::error::Error>> {
let result = std::fs::read_to_string("test.txt");
let _content = match result {
    Ok(content) => { content },
    Err(error) => { return Err(error.into()); }
};
# Ok(())
# }
```

然而，这改变了我们函数的返回值类型。实际上，一直以来我们的示例都隐藏了一些东西：
函数的签名（或者说返回值类型）。在最后的含有 `return` 的示例中，它变得很重要了。
下面是_完整_的示例：

```rust,no_run
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let result = std::fs::read_to_string("test.txt");
    let content = match result {
        Ok(content) => { content },
        Err(error) => { return Err(error.into()); }
    };
    println!("file content: {}", content);
    Ok(())
}
```

我们的返回值类型是 `Result`！这也就是为什么我们可以在 match 的第二个分支里写
`return Err(error)`。看到最下面的 `Ok(())` 了么？它是函数的默认返回值，
意为“结果没问题，没有内容”。

<aside>

**注：**
为什么不写成 `return Ok(())`？当然这样写也完全没问题。
在 Rust 中，任何代码块中的最后的表达式即为其返回值，因此习惯上省略了 `return`。

</aside>

## `?` 标识

正如调用 `.unwrap()` 相当于 `match` 中快捷设置错误分支中 `panic!`，
我们还有另一个快捷的调用 `?` 使得在 `match` 的错误分支中直接返回。

是的，就是这个问号。你可以在 `Result` 类型后加上这个运算符，
Rust 在内部会将之展开生成类似于我们刚写的 `match` 代码块。

来试试吧：

```rust,no_run
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string("test.txt")?;
    println!("file content: {}", content);
    Ok(())
}
```

很简洁是吧！

<aside>

**注：**
这里还发生了一些事情，但我们不需要去理解就能使用它。
比如，在我们的 `main` 函数中，错误类型是 `Box<dyn std::error::Error>`。
但我们在 `read_to_string` 中返回的却是 [`std::io::Error`]。
它能正常工作是因为 `?` 将代码扩展为 _converts_ 错误类型。

`Box<dyn std::error::Error>` 同样是个很有意思的类型。
 它是一个包含_任意_实现了标准 [`Error`][`std::error::Error`] 特征类型的 `Box`，
所以我们可以在所有返回 `Result` 的一般函数里使用 `?`。

[`std::error::Error`]: https://doc.rust-lang.org/1.39.0/std/error/trait.Error.html

</aside>

## Providing Context

在 `main` 函数中使用 `?` 来获取错误，可以正常工作，但它有一些不足。
比如：若使用 `std::fs::read_to_string("test.txt")?` 时，`test.txt` 文件不存在，
你将获得以下错误信息：

```text
Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

在这里你的代码里没有包含文件名，这会使得确认是哪个文件 `NotFound` 变得很麻烦。
但我们有许多种办法可以改进它。

比如，我们可以创建一个自己的错误类型，然后使用它去生成自定义的错误信息：

```rust,ignore
{{#include errors-custom.rs}}
```

现在，运行它将得到我们自定义的错误信息：

```text
Error: CustomError("Error reading `test.txt`: No such file or directory (os error 2)")
```

尽管不是很完美，但我们稍后可以很轻松地为我们的类型调试输出。

这种模式很觉，但它有一个问题：我们并没有保存下原始错误，而只存储了它呈现的字符串。
一个名为 [`anyhow`] 的常用库可巧妙的解决这个问题：类似于 `CustomError` 类型，
它的 [`Context`] 特征可以用来添加描述信息。此外，它还能存储下原始错误，
所以我们得到了一个指出根本原因的错误信息链。

[`anyhow`]: https://docs.rs/anyhow
[`Context`]: https://docs.rs/anyhow/1.0/anyhow/trait.Context.html

在 `Cargo.toml` 中的 `[dependencies]` 字段添加上 `anyhow = "1.0"`。

一个完整的示例如下：

```rust,ignore
{{#include errors-exit.rs}}
```

它将打印出一个错误：

```text
Error: could not read file `test.txt`

Caused by:
    No such file or directory (os error 2)
```
