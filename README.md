# 学Rust不学Cargo，等于没学Rust：workspace详解
上一篇文章我们介绍了Cargo.toml中的[features]配置块，这次我们再来看看[workspace]配置块的用法。

- [Cargo：features特性详解](https://juejin.cn/post/7315846799083749427)

Rust 中的`Workspace`是一种组织多个 Rust crate（项目或库）的结构。使得它们可以协同工作、共享依赖关系，以及更方便地进行管理和构建。
如果你是Java开发者，workspace这个概念类似Java中的`maven`父工程。
子工程可以共享父工程中的很多配置项，如依赖，版本等配置。子工程可以`选择性的继承`父工程的配置。

一个workspace组织的项目通常如下：

```shell
│  Cargo.lock
│  Cargo.toml (Workspace configuration)
├─target/
├─bin
│  │  Cargo.toml
│  │
│  └─src
│          main.rs
│
└─osk-json-lib
    │  Cargo.toml
    │
    └─src
            lib.rs

```

工程根目录下`Cargo.toml`是配置当前workspace, 同时该目录下还有多个`子包(crate)`，例如：bin、osk-json-lib。所有的crate共同编译到根目录下的target，且使用同一个Cargo.lock。

下面我们看一下工程根目录的`Cargo.toml`
```toml
[workspace]
resolver="2"
members = [
    "osk-json-lib",
    "bin"
]

[workspace.package] # 共享的package配置项
version = "1.2.3" 
authors = ["Nice Folks"] 
description = "A short description of my package" 
documentation = "https://example.com/bar"

[workspace.dependencies] # 共享的依赖项
cc = "1.0.73" 
rand = "0.8.5" 
regex = { version = "1.6.0", default-features = false, features = ["std"] }

[workspace.metadata.webcontents] 
root = "path/to/webproject" 
tool = ["npm", "run", "build"]
```

Workspace自己的配置，都定义在`[workspace]`配置项里面，包含:

- members： 当前workspace中包含哪些crate。
- resolver：当前workspace使用哪个版本的解析器。
- exclude：不常用，当前workspace中排除的crate，当目录中有crate不属于这个workspace可以使用这个属性排除掉。
- default-members：不常用，类似members。

> # resolver是什么?
> resolver用于指定当前workspace使用的依赖解析器版本，目前有两个版本：版本1，版本2。
>
> Rust作为一门现代语言，在语言迭代过程中会引入一些不兼容的语法。同一个只能使用一个版本的语法，因此crate在创建时就要指定采用那一版语法。目前有三个版本：Edition2015、Edition2018、[Edition2021](https://doc.rust-lang.org/edition-guide/editions/index.html#what-are-editions)。
>
> Edition2021和以后的依赖解析器默认是版本2，之前的是版本1。
>
> crate指定`edition`选项，Cargo就能识别到依赖解析器的版本。然而workspace没有这个选项，因此注明`resolver="2"`,就表示采用的是版本2.


### 1. 共享的package配置项

在根工程的Cargo.toml中我们看到`[workspace.package]`配置项，它的作用是让子包（crate）可以共享package属性。

/osk-json-lib/Cargo.toml ：
```toml
[package] 
name = "bar" 
version.workspace = true 
# authors.workspace = true
```

例如，crate需要统一版本号，无需在每个crate写版本号。只需在[package]中写入`version.workspace = true `就能继承workspace中的版本号。升级版本号只需修改[workspace.package]的版本号，就同步了所有的crate版本。

类似的配置还有下面这些：
|||
|---|---|
|`authors`|`categories`|
|`description`|`documentation`|
|`edition`|`exclude`|
|`homepage`|`include`|
|`keywords`|`license`|
|`license-file`|`publish`|
|`readme`|`repository`|
|`rust-version`|`version`|

需要注意的是，这些配置项都需要定义在`[workspace.package] `配置块中。

### 2. 共享的依赖项

如同继承`[workspace.package]`配置，子包也能从`[workspace.dependencies]` 中继承dependencies。

/osk-json-lib/Cargo.toml :
```toml
[package] 
name = "bar" 
version.workspace = true 

[dependencies] 
regex = { workspace = true, features = ["unicode"] } 

[build-dependencies] 
cc.workspace = true 

[dev-dependencies] 
rand.workspace = true
```

继承依赖有两种写法：
```
# 从workspace中继承cc
1. cc.workspace = true 
# 很明显这种写法可以仅引入需要的features，而无需引入整个依赖项。
2. regex = { workspace = true, features = ["unicode"] } 
```
### 3. Workspace可同时作为crate

我们前面讨论的把`workspace`当作组织多个`crate`的容器来使用。

根工程本身也可以作为一个`crate`，这种用法通常是子包都是`lib`，根工程是`bin`类型的，这样就可以看作是一个项目下细分了不同的`子模块`。

如下，我们在`[workspace]`配置快的上面加入`[package]`配置快，那么这个根工程就被当作一个crate来看待了。假如这个项目是一个桌面软件，有两个子模块分别是video、audio。

```toml
[package]
name = "osk-desktop"
version = "0.1.0"
edition = "2021"

[workspace]
members = [
    "video",
    "audio"
]
```

`Cargo.toml`同时含有[package] 和 [workspace] 被称作 `Root Package`。开始介绍的用法被称作`虚拟清单( virtual manifest )`。这是Rust文档给这两种用法的命名，最重要的是理解这两种用法的本质区别。


### 总结

- `[workspace]` — 定义工作空间。
    - `resolver` — 设置要使用的依赖解析器。
    - `members` — 要包含在工作区中的包。
    - `exclude` — 要从工作区中排除的包。
    - `default-members` — 当没有选择特定包时要操作的包。
- `[workspace.package]` — 在包中继承的配置项。
- `[workspace.dependencies]` — 用于继承包依赖项的依赖项。
- `[workspace.lints]` — 在lints中继承的配置项。
- `[workspace.metadata]` — 额外设置。
- `[patch]` — 覆盖依赖项。
- `[replace]` — 覆盖依赖项(deprecated)。
- `[profile]` — 编译器设置和优化。

<img src="docs/8b7f3e339f07a43d66dd696a3c66858.jpg" height="400px">
