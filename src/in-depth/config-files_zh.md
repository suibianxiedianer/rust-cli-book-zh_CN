# 使用配置文件

处理配置可能会很麻烦，尤其是当你支持多个操作系统，
而这些操作系统有不同的固定或临时配置文件存放位置。

有许多种办法可以解决这个问题，一些的方式会更接近低层。
There are multiple solutions to this,
some being more low-level than others.

最容易使用的 crate 是 [`confy`]。
它要求你提供程序的名称，并要求你通过 `struct` 来定义配置
（需实现 `Serialize` 和 `Deserialize`），剩下的事儿交给它就可以了！

```rust,ignore
#[derive(Debug, Serialize, Deserialize)]
struct MyConfig {
    name: String,
    comfy: bool,
    foo: i64,
}

fn main() -> Result<(), io::Error> {
    let cfg: MyConfig = confy::load("my_app")?;
    println!("{:#?}", cfg);
    Ok(())
}
```

在你放弃可配置性时，毫无疑问它是极其方便好用的。
所以在你需要简单的配置时，使用这个 crate 会非常合适。

[`confy`]: https://docs.rs/confy/0.3.1/confy/

## 配置环境

<aside class="todo">

**TODO**

1. Evaluate crates that exist
2. Cli-args + multiple configs + env variables
3. Can [`configure`] do all this? Is there a nice wrapper around it?

</aside>

[`configure`]: https://docs.rs/configure/0.1.1/configure/
