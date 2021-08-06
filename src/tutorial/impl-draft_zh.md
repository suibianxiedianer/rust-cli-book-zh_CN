# _grrs_ 的第一个实现

完成命令行参数章节后，我们获取了输入参数，现在我们可以开始编写实现工具了。
当前我们的 `main` 函数中仅有一行：

```rust,ignore
{{#include impl-draft.rs:17:17}}
```

首先让我们打开指定的文件。

```rust,ignore
{{#include impl-draft.rs:18:19}}
```

<aside>

**注:**
看到 [`.expect`] 方法了吧？这是一个快捷功能，当无法读取到值时
（在这里是输入的文件）会立即退出程序。它还并不完美，在下节
[更好地反馈错误] 中，我们将探究如何改进它。

[`.expect`]: https://doc.rust-lang.org/1.39.0/std/result/enum.Result.html#method.expect
[更好地反馈错误]:./errors_zh.html

</aside>

现在，让我们遍历所有行并打印出包含匹配字符串的每一行：

```rust,ignore
{{#include impl-draft.rs:21:25}}
```

## Wrapping up

你的代码现在看起来应该如此：

```rust,ignore
{{#include impl-draft.rs}}
```

来试试：`cargo run -- main src/main.rs` 应该能正常工作了！

<aside class="exercise">

**练习:**
这并非最好的实现：它将整个文件读取到内存中了——无论文件有多大。
去寻找一种优化的方法吧！（使用 [`BufReader`] 而非 `read_to_string()` 也许不错）

[`BufReader`]: https://doc.rust-lang.org/1.39.0/std/io/struct.BufReader.html

</aside>
