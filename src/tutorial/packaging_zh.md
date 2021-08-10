# 打包并发布一个 Rust 工具

如果你确信你的程序已经准备好提供给其它人使用了，那么是时候打包并发布它了！

我们可以有许多种办法，现在我们从“最快设置”到“对用户最方便”看其中的三种。

## 最快的方法：`cargo publish`

发布你的程序最简单的办法就是使用 cargo。
你还记得我们怎么往我们的项目中添加依赖么？
cargo 从默认的 “crate 仓库”[crates.io] 下载它们。
使用 `cargo publish`，你可以在 [crates.io] 上发布你的 crate。
这适用于包括二进制程序在内的所有 crates。

如果你已经在 [crates.io] 上创建了一个账号，
那么在上面发布一个 crate 是非常简单的。
目前，它是通过 GitHub 来认证的，所以你需要一个 GitHub 账号,
并在 [crates.io] 上登录。然后，你需要在你本地的机器上登录（终端登录）。
然后，[crates.io] 的账号设置页面，生成一个新的 token，
在你的本地终端运行 `cargo login <your-new-token>`。
每台机器上只需运行一次即可一直使用。
有兴趣可以去看 cargo 的 [publishing guide] 

现在你已经了解了 cargo 和 crates.io，可以去发布你的 crate 了。
在你急急忙忙地想着发布一个新的（或更新版本）crate 前，
你最好打开 `Cargo.toml` 并检查是否提供了必要的元数据。
你可以在 [cargo's manifest format] 的文档中找到所有需要设置的字段，
下面展示了其中的常见条件：

```toml
[package]
name = "grrs"
version = "0.1.0"
authors = ["Your Name <your@email.com>"]
license = "MIT OR Apache-2.0"
description = "A tool to search files"
readme = "README.md"
homepage = "https://github.com/you/grrs"
repository = "https://github.com/you/grrs"
keywords = ["cli", "search", "demo"]
categories = ["command-line-utilities"]
```

<aside class="note">

**注：**
这个示例包括了必须的 license 字段，它选择了 Rust 项目中常见的许可证，
编译器本身也是使用的这个 license。
它还引用了 `README.md` 文件。
它应该包括一个对你的项目的简单描述信息，这个信息不仅会显示在 crates.io
上你的 crate 页面中，还在会 GitHub 仓库页面显示。

</aside>

[crates.io]: https://crates.io/
[crates.io account page]: https://crates.io/me
[publishing guide]: https://doc.rust-lang.org/1.39.0/cargo/reference/publishing.html
[cargo's manifest format]: https://doc.rust-lang.org/1.39.0/cargo/reference/manifest.html

### 如何从 crates.io 上安装程序

我们刚刚学习了如何在 crates.io 上发布一个 crate，那么又如何去安装它呢？

我们同样可以使用 cargo 命令，运行 `cargo install <crate-name>`。
它默认会下载这个 crate，编译其中所有的二进制程序（使用 release 模式，
所以可能会稍慢）并将程序拷贝到 `~/.cargo/bin` 目录。
（需确保你的 PATH 中有这个路径，才可正常使用安装的命令）

我们还可以直接在源码仓库下安装程序，使用这种文件可以指定安装哪些二进制程序，
及程序安装到什么位置。详情可查看 `cargo install --help`。

### 什么时候使用它

使用 `cargo install` 可以很方便的安装一个 crate 程序。
但你需要了解到它的几个不足之处：
因为它总是会从头开始编译你的源码，所以使用你的工具的用户，
在他们的机器上需要安装上 Rust，Cargo 及其它所需的依赖。
而且编译一个大型的 Rust 代码库也需要较长的时间。

最好使用它来分发面向其它 Rust 开发者的工具。
比如，许多 cargo 的子命令，
像 `cargo-tree` 或 `cargo-outdated` 可以使用这种方法来安装。

## 发布二进制程序

Rust 会默认编译出使用静态链接的机器代码。
当你运行 `cargo build`时，若你的项目中有包含一个叫 `grrs` 的二进制 target，
你将得到一个叫 `grrs` 的二进制文件。
来试试吧：使用 `cargo build`，二进制文件在 `target/debug/grrs`，
若使用 `cargo build --release` 则生成的文件在 `target/release/grrs`。
除非你使用的 crates 中明确地说明了需要在目标系统上安装外部依赖库
（如需要使用系统提供的 OpenSSL），否则这个二进制文件只会依赖于通用系统库。
这意味着，将此二进制程序拷贝到任何相同的操作系统上，它都能完美地运行！

这已经很强大了！它解决了我们刚刚提到的，关于 `cargo isntall` 的两个不足：
它不需要用户的机器上安装 Rust，也不需要再去编译而可直接运行。

所以，`cargo build` _已经_为我们编译出了二进制程序。现在唯一的问题是，
还不能保证其可在所有的平台上都可用。如果你在 Windows 上运行 `cargo build`，
你并不会得到一个可在 Mac 上运行程序。
那么有没有办法为所有的常见平台自动生成这些二进制程序呢？

### 使用 CI 构建二进制版本

如果你的工具已经开源且托管在 GitHub 上，可以非常容易地设置一个免费的 CI
（持续集成）—— [Travis CI]。（或其它平台上使用其它的服务，Travis 是最受欢迎的）
在你每次 push 新代码到你的仓库时，它会在一个虚拟机上运行设置命令。
至于是执行什么命令，或使用哪种虚拟机，这都是可以配置的。
比如，在一个安装了 Rust 和常用构建工具的机器上，执行 `cargo test` 是一个好主意。
如果这个命令失败了，你就知道最近的提交的代码是有问题的。

[Travis CI]: https://travis-ci.com/

We can also use this
to build binaries and upload them to GitHub!
Indeed, if we run
`cargo build --release`
and upload the binary somewhere,
we should be all set, right?
Not quite.
We still need to make sure the binaries we build
are compatible with as many systems as possible.
For example,
on Linux we can compile not for the current system,
but instead for the `x86_64-unknown-linux-musl` target,
to not depend on default system libraries.
On macOS, we can set `MACOSX_DEPLOYMENT_TARGET` to `10.7`
to only depend on system features present in versions 10.7 and older.

You can see one example of building binaries using this approach
[here][wasm-pack-travis] for Linux and macOS
and [here][wasm-pack-appveyor] for Windows (using AppVeyor).

[wasm-pack-travis]: https://github.com/rustwasm/wasm-pack/blob/51e6351c28fbd40745719e6d4a7bf26dadd30c85/.travis.yml#L74-L91
[wasm-pack-appveyor]: https://github.com/rustwasm/wasm-pack/blob/51e6351c28fbd40745719e6d4a7bf26dadd30c85/.appveyor.yml

Another way is to use pre-built (Docker) images
that contain all the tools we need
to build binaries.
This allows us to easily target more exotic platforms, too.
The [trust] project contains
scripts that you can include in your project
as well as instructions on how to set this up.
It also includes support for Windows using AppVeyor.

If you'd rather set this up locally
and generate the release files on your own machine,
still have a look at trust.
It uses [cross] internally,
which works similar to cargo
but forwards commands to a cargo process inside a Docker container.
The definitions of the images are also available in
[cross' repository][cross].

[trust]: https://github.com/japaric/trust
[cross]: https://github.com/rust-embedded/cross

### How to install these binaries

You point your users to your release page
that might look something [like this one][wasm-pack-release],
and they can download the artifacts we've just created.
The release artifacts we've just generated are nothing special:
At the end, they are just archive files that contain our binaries!
This means that users of your tool
can download them with their browser,
extract them (often happens automatically),
and copy the binaries to a place they like.

[wasm-pack-release]: https://github.com/rustwasm/wasm-pack/releases/tag/v0.5.1

This does require some experience with manually "installing" programs,
so you want to add a section to your README file
on how to install this program.

<aside class="note">

**Note:**
If you used [trust] to build your binaries and added them to GitHub releases,
you can also tell people to run
`curl -LSfs https://japaric.github.io/trust/install.sh | sh -s -- --git your-name/repo-name`
if you think that makes it easier.

</aside>

### When to use it

Having binary releases is a good idea in general,
there's hardly any downside to it.
It does not solve the problem of users having to manually
install and update
your tools,
but they can quickly get the latest releases version
without the need to install Rust.

### What to package in addition to your binaries

Right now,
when a user downloads our release builds,
they will get a `.tar.gz` file
that only contains binary files.
So, in our example project,
they will just get a single `grrs` file they can run.
But there are some more files we already have in our repository
that they might want to have.
The README file that tells them how to use this tool,
and the license file(s),
for example.
Since we already have them,
they are easy to add.

There are some more interesting files
that make sense especially for command-line tools,
though:
How about we also ship a man page in addition to that README file,
and config files that add completions of the possible flags to your shell?
You can write these by hand,
but _clap_, the argument parsing library we use
(which structopt builds upon)
has a way to generate all these files for us.
See [this in-depth chapter][clap-man-pages]
for more details.


[clap-man-pages]: ../in-depth/docs.html


## Getting your app into package repositories

Both approaches we've seen so far
are not how you typically install software on your machine.
Especially command-line tools
you install using global package managers
on most operating systems.
The advantages for users are quite obvious:
There is no need to think about how to install your program,
if it can be installed the same way as they install the other tools.
These package managers also allow users to update their programs
when a new version is available.

Sadly, supporting different systems means
you'll have to look at how these different systems work.
For some,
it might be as easy as adding a file to your repository
(e.g. adding a Formula file like [this][rg-formula] for macOS's `brew`),
but for others you'll often need to send in patches yourself
and add your tool to their repositories.
There are helpful tools like
[cargo-rpm](https://crates.io/crates/cargo-rpm),
[cargo-deb](https://crates.io/crates/cargo-deb), and
[cargo-aur](https://crates.io/crates/cargo-aur),
but describing how they work
and how to correctly package your tool
for those different systems is beyond the scope of this chapter.

[rg-formula]: https://github.com/BurntSushi/ripgrep/blob/31adff6f3c4bfefc9e77df40871f2989443e6827/pkg/brew/ripgrep-bin.rb

Instead,
let's have a look at a tool that is written in Rust
and that is available in many different package managers.

### An example: ripgrep

[ripgrep] is an alternative to `grep`/`ack`/`ag` and is written in Rust.
It's quite successful and is packaged for many operating systems:
Just look at [the "Installation" section][rg-install] of its README!

Note that it lists a few different options how you can install it:
It starts with a link to the GitHub releases
which contain the binaries so you can download them directly;
then it lists how to install it using a bunch of different package managers;
finally, you can also install it using `cargo install`.

This seems like a very good idea:
Don't pick and choose one of the approaches presented here,
but start with `cargo install`,
add binary releases,
and finally start distributing your tool using system package managers.

[ripgrep]: https://github.com/BurntSushi/ripgrep
[rg-install]: https://github.com/BurntSushi/ripgrep/tree/31adff6f3c4bfefc9e77df40871f2989443e6827#installation
