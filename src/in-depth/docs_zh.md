# 为你的 CLI 程序生成文档

CLI 程序的文档通常包括调用命令时使用 `--help` 及 `man` 手册。

当你使用 `clap` v3（目前还是 unreleased 状态），会通过 `man` 后端自动生成这些。

```rust,ignore
#[derive(Clap)]
pub struct Head {
    /// file to load
    #[clap(parse(from_os_str))]
    pub file: PathBuf,
    /// how many lines to print
    #[clap(short = "n", default_value = "5")]
    pub count: usize,
}
```

其次，你需要使用 `build.rs` 来在编译时从你的代码中生成手册。

在这里你要留意几件事（比如如何打包你的程序），
但现在我们只是简单地把 `man` 文件放到我们的 `src` 同级目录。

```rust,ignore
use clap::IntoApp;
use clap_generate::gen_manuals;

#[path="src/cli.rs"]
mod cli;

fn main() {
    let app = cli::Head::into_app();
    for man in gen_manuals(&app) {
        let name = "head.1";
        let mut out = fs::File::create(name).unwrap();
        use std::io::Write;
        out.write_all(man.render().as_bytes()).unwrap();
    }
}
```

现在你编译你的程序时，将在你的项目目录生成一个 `head.1` 文件。

如果你使用 `man` 打开它，就可以看到你的文档了。
