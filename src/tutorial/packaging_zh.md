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

我们还可以使用它还构建二进制程序并上传到 GitHub 上！
那么，如果我们使用 `cargo build --release` 生成并将程序上传到某个地方，
就一切 OK 了么？也不尽然，我们需要确保我们的程序适用于尽可能多的系统。
比如，在 Linux 上我们可以不为当前系统编译，而是针对 `x86_64-unknown-linux-musl`
编译，从而使程序不依赖于默认的系统库。
在 macOS 上，我们可以设置 `MACOSX_DEPLOYMENT_TARGET` 为 `10.7`，
使得程序只会依赖于 10.7 及其之后版本上存在的系统功能。

你可以在[这儿][wasm-pack-travis]看到使用此方法在 Linux 和 macOS 上编译的例子，
若使用 Windows，请看[这个][wasm-pack-appveyor]（使用 AppVeyor）。

[wasm-pack-travis]: https://github.com/rustwasm/wasm-pack/blob/51e6351c28fbd40745719e6d4a7bf26dadd30c85/.travis.yml#L74-L91
[wasm-pack-appveyor]: https://github.com/rustwasm/wasm-pack/blob/51e6351c28fbd40745719e6d4a7bf26dadd30c85/.appveyor.yml

另一种方法是使用包含我们编译程序所需要到的所有工具的 pre-built (Docker) 镜像。
这也使我们能够轻松地编译适配于更多不同的平台。
在[trust] 项目中包含可以在你的项目中使用的脚本及其如何设置使用的说明。
它还支持使用 AppVeyor 的 Windows 系统。

如果你更愿意在本地设置，并在本机上生成要发布的文件，你也可以看看 trust。
它在里面使用了 [cross]，cross 的工作原理类似于 cargo，
它会将命令转发到 Docker 容器中的 cargo 进程。
此镜像的定义相关可在其仓库查看 [cross' repository][cross]。

[trust]: https://github.com/japaric/trust
[cross]: https://github.com/rust-embedded/cross

### 如何安装这些二进制程序

你的发布页面也许是像[这样][wasm-pack-release]的，
用户可以在这个页面下载我们刚生成的文件。
我们刚刚生成的发布文件没什么特别的：它们只是一个包括我们的二进制程序的压缩包！
这意味着你的工具的用户可以从浏览器上下载、解压，并将程序拷贝到任何地方。

[wasm-pack-release]: https://github.com/rustwasm/wasm-pack/releases/tag/v0.5.1

这需要用户有手动安装程序的经验，所以你最好在你的 README 文件中加上安装方法说明。

<aside class="note">

**注：**
如果你使用 [trust] 构建你的程序并在 GitHub releases 上发布，
你还可以告诉你的用户来运行
`curl -LSfs https://japaric.github.io/trust/install.sh | sh -s -- --git your-name/repo-name`
去安装，这也许会更简单些（相对于手动安装）。

</aside>

### 什么时候使用它

一般来说使用 release 的二进制文件来安装是非常好的主意，它几乎没啥缺点。
虽然它没解决自动安装及更新的问题，
但可以在不安装 Rust 的情况下得到并使用相应的程序。

### 除了二进制程序外还要打包什么

现在，当一个用户下载了我们的 release 构建程序，他会得到一个 `.tar.gz` 文件。
里面只包含了一些二进制文件。而在我们的示例项目中，只有一个可运行的 `grrs` 文件。
但在我们的仓库中，也许还有一些其它文件是用户想要的，
比如 README 中有程序的使用方法，license 文件有程序的许可信息。
我们可以非常容易地将这些已有的文件添加到 release 的压缩包中。

这里还有一些非常有意思且有用的文件，尤其是在命令行工具中：
除了 README 文件，我们再提供一个帮助手册怎么样，然后再添加一个支持 SHELL
自动补全功能的配置文件？你可以自己来手写这些，
或者使用 _clap_，
这个解析参数的库（structopt 就是基于它）提供了一个方法来帮我们生成这些文件。
详情可查看[深入了解][clap-man-pages]这一章。


[clap-man-pages]: ../in-depth/docs_zh.html


## 将你的软件放到软件仓库中

到目前为止，我们看到的两种方式都不是通常情况下你在计算机上安装软件的方式。
尤其在大多数操作系统上，会使用全局包管理器来进行命令行工具的安装。
这种包管理器对用户来说，优势非常明显：
他不需要去考虑如何安装你的软件，因为其安装方法和安装其它工具一样。
这些包管理器同样允许用户在软件有新版本时去更新它们。

不过，要支持不同的系统意味着你必须了解它们是如何工作的。
有一些是比较简单的
（比如，为 macOS 的 `brew` 添加一个 Formula 像[这个][rg-formula]），
还有的需要你自己来发送补丁并将你的工具添加他们的软件仓库中。
这里有一些很有用的工具，像
[cargo-rpm](https://crates.io/crates/cargo-rpm)、
[cargo-deb](https://crates.io/crates/cargo-deb) 和
[cargo-aur](https://crates.io/crates/cargo-aur)。
但它们是如何工作的及你要如何正确地为不同的系统打包你的工具不在本章的讨论范围中。。

[rg-formula]: https://github.com/BurntSushi/ripgrep/blob/31adff6f3c4bfefc9e77df40871f2989443e6827/pkg/brew/ripgrep-bin.rb

现在，让我们看一下一个使用 Rust 编写，且在许多不同的包管理器中都存在的一个工具。

### 一个示例：ripgrep

[ripgrep] 是一个用 Rust 编写的，类似于 `grep`/`ack`/`ag` 的工具。
这是一个十分成功的项目，且打包分发支持了许多不同的操作系统：
请查看项目中 README 的[安装][rg-install] 部分。

注意，其中列出了几种安装它的办法：
首先它提供了一个指向 GitHub releases 页面的链接，在这儿你可以直接下载二进制程序；
然后，也列出了，如何使用不同的软件管理器来安装；
最后，你也可以使用 `cargo install` 来安装它！

不要只选择这里介绍的一种方法，而是先使用 `cargo install`，
然后再发布二进制 releases，最后再尝试使用系统包管理器来分发你的工具。
相信我，这会是一个好主意！

[ripgrep]: https://github.com/BurntSushi/ripgrep
[rg-install]: https://github.com/BurntSushi/ripgrep/tree/31adff6f3c4bfefc9e77df40871f2989443e6827#installation
