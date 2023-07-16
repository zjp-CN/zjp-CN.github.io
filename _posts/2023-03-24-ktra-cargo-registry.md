---
title: 使用 ktra 搭建私人 Cargo registry
date: 2023-03-24 15:19:35 +0800
categories: [Rust]
tags: [开发日志, Rust, cargo, 工具]
img_path: /assets/img/2023-03-24-ktra-cargo-registry/
---

使用工具：
* [ktra](https://github.com/moriturus/ktra)：纯 Rust 编写的，用于搭建 Cargo registry 的命令行工具

安装它：

```shell
cargo install ktra
```

注意：
* 这开启了 `secure-auth` 和 `db-sled`，数据库是内置的 [sled](https://github.com/spacejam/sled)
* 如果你需要 redis 或者 mongo 作为后端数据库，自行添加 features，见 <https://book.ktra.dev/installation/cargo.html>
* 如果你不想通过命令行运行，而是通过 docker 部署，见 <https://book.ktra.dev/installation/docker.html>

# 步骤一：创建 Git Index 仓库

由于 Cargo 使用 [git 或者 sparse 协议][registry-protocols]，后者是最近才稳定的（2023 年），而 ktra
出现得比较早，所以目前只基于 git 协议。

[registry-protocols]: https://doc.rust-lang.org/cargo/reference/registries.html#registry-protocols

在 gitee 上创建新仓库，比如 `ktra-cargo-registry`，然后在本地新建这个仓库（主分支为 main），创建仓库配置文件，并提交和推送：

```shell
mkdir ktra-cargo-registry
cd ktra-cargo-registry
git remote add origin git@gitee.com:xxx/ktra-cargo-registry.git
echo '{"dl":"http://localhost:8000/dl","api":"http://localhost:8000"}' > config.json
git add config.json
git commit -am "initial commit"
git push origin main
```

注意：
* 本文描述的是私有 registry 部署（Index 仓库是私有的，发布的 crate 存于自己的私有服务器）
* 如果你想基于私有 registry 进行更大范围的公开使用，那么可能会对 ktra 的 [OpenId] 感兴趣（它更复杂）
* `dl` 和 `api` 填 IP 地址：对于服务器，把 localhost 换成服务器的公网 IP 地址（继续使用 localhost 则只供服务器本机使用这个源）

[OpenId]: https://book.ktra.dev/quick_start/openid.html

# 步骤二：增加本地 Cargo registry

在 Cargo 的配置文件中，比如全局配置文件 `~/.cargo/config` 

```toml
[registries]
ktra = { index = "git@gitee.com:xxx/ktra-cargo-registry.git" }
```

注意：
* 全局配置文件可以是 `~/.cargo/config` 或者 `~/.cargo/config.toml`，见 <https://doc.rust-lang.org/cargo/reference/config.html>
* 你可以填 HTTPS 地址，git 验证会简单些（因为 SSH 的方式需要设置 Cargo 的 `git-fetch-with-cli`，以及 gitee 设置 SSH 公钥）
  * 具体来说，在 HTTPS 情况下，登陆或者获取 registry 需要手动输入账号和密码

      ```shell
      $ cargo b
          Updating `ktra` index
      Username for 'https://gitee.com': xxx
      Password for 'https://xxx@gitee.com': xxx
      ```

  * 所以，你可能还需要设置 `git config --global credential.helper store` 或者 `git config --global credential.helper cache`
    来减少手动登录的次数。嗯，这就是 git 最基本的登录设置 :)

# 步骤三：创建 ktra 配置文件并部署服务

一个 TOML 文件，传递给 ktra 命令行，里面配置了 ktra 的内置数据库。我把它存放到 `~/.config/ktra/ktra.toml`：

```toml
[index_config]
branch = "main"

remote_url = "https://gitee.com/xxx/ktra-cargo-registry.git"
https_username = "xxx"
https_password = "xxx"

# 这两个设置似乎没有用
git_email = ""
git_name = "ktra"

# ssh 总是出现 git error: username not defined
# remote_url = "ssh://git@gitee.com/xxx/ktra-cargo-registry.git"
# ssh_username = "xxx"
# ssh_privkey_path = "~/.ssh/id_ed25519"
# ssh_pubkey_path = "~/.ssh/id_ed25519.pub"
```

```shell
cd ~/.config/ktra/
ktra # 或者 ktra -c ./ktra.toml，此命令部署服务到本地 8000 端口
```

# 步骤四：准备账户

由于服务需要一直运行，所以打开新的终端运行以下命令：

```shell
# 创建账户 admin 和密码，获得 {"token":"xxx"}
curl -X POST -H 'Content-Type: application/json' -d '{"password":"PASSWORD"}' http://localhost:8000/ktra/api/v1/new_user/admin

# cargo 登录 ktra：发布包，比如 cargo publish/yank 等
CARGO_NET_GIT_FETCH_WITH_CLI=true cargo login --registry ktra
#please paste the token found on http://localhost:8000/me below
# 对话过程中，粘贴 token，看到以下这行就登陆成功
#      Login token for `ktra` saved
```

注意，这里的环境变量表示使用 git 命令行进行验证，如果你不这样做，需要自行参考
<https://doc.rust-lang.org/cargo/appendix/git-authentication.html#ssh-authentication>。

自行验证看起来很复杂，而且并不支持完整的 SSH 验证（我没试过），建议使用 git cli 的方式验证。

（如果你的 git 不想折腾 SSH，就在 Cargo registry 中使用 HTTPS 地址，步骤二已经描述了。）

注意：由于 registry 写的是 SSH 地址，这意味着从 ktra 拉取库都需要 SSH 验证，建议不要像上面那样采用命令行的局部环境变量。

我那样做是为了快速验证，你最好设置成全局环境变量或者在全局或者局部 `.cargo/config.toml` 配置文件中写

```toml
[net]
git-fetch-with-cli = true
```

# 步骤五：试试自己的 registry

## 发布库

```shell
cargo new ktra-test --lib
cd ktra-test
cargo publish --allow-dirty --registry=ktra
```

```shell
# cargo publish 的输出
   Packaging ktra-test v0.1.0 (/rust/tmp/ktra-test)
   Verifying ktra-test v0.1.0 (/rust/tmp/ktra-test)
   Compiling ktra-test v0.1.0 (/rust/tmp/ktra-test/target/package/ktra-test-0.1.0)
    Finished dev [unoptimized + debuginfo] target(s) in 1.13s
    Packaged 3 files, 941.0B (731.0B compressed)
   Uploading ktra-test v0.1.0 (/rust/tmp/ktra-test)
    Updating `ktra` index
remote: Enumerating objects: 6, done.
remote: Counting objects: 100% (6/6), done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 5 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (5/5), 486 bytes | 486.00 KiB/s, done.
From ssh://gitee.com/xxx/ktra-cargo-registry
   7c6b370..90c97e3             -> origin/HEAD
```

根据我的观察，发布库到 ktra 做了以下事情：
* 在本地打包库，然后编译，成功之后
  * 推送这个库的信息到 INDEX 仓库（仅仅是一个 json 文件）

    ![](ktra.png)

  ```json
  {"name":"ktra-test","vers":"0.1.0","deps":[],"cksum":"89c08ea6c8ea00f06a7f0dd2ec468cf1116458aa38cd6aa5caf1cde05b7e0849","features":{},"yanked":false,"links":null}
  ```

  * 把打包的库数据上传到运行 ktra 的服务器
    * 如果你按照我的上面步骤操作，那么你是在 `~/.config/ktra/` 运行 ktra 的，那么库在 `crates/` 下

  ```shell
  $ ll ~/.config/ktra/
  total 28K
  drwxr-xr-x 3 root root 4.0K Mar 24 18:44 crates/
  drwxr-xr-x 2 root root 4.0K Mar 24 16:09 crates_io_caches/
  drwxr-xr-x 3 root root 4.0K Mar 24 21:33 db/
  drwxr-xr-x 4 root root 4.0K Mar 24 18:44 index/
  -rw-r--r-- 1 root root  348 Mar 24 22:57 ktra.toml

  $ ll ~/.config/ktra/crates/ktra-test/0.1.0/download
  -rw-r--r-- 1 root root 731 Mar 24 18:44 /root/.config/ktra/crates/ktra-test/0.1.0/download
  ```

## 拉取库

```shell
cargo new use-ktra
cargo add serde # 来自 crates.io 的库           
cargo add ktra-test --registry ktra
cargo check # bingo!
```

```toml
# cat Cargo.toml
[dependencies]
serde = "1.0.158"
ktra-test = { version = "0.1.0", registry = "ktra" }
```

# 回答一些基本问题

## 我为什么需要私有 registry？

如果你的代码是公开的、给别人使用的，当然不需要一个私有 registry。

如果不是，那么你可能想弄一个私人控制的、小范围的东西。

## 除了 ktra，还有别的选择吗？

其他选择见 <https://github.com/rust-lang/cargo/wiki/Third-party-registries>。

ktra 是 Rust 编写的，而且功能简单小巧，特别适合个人使用或者小团体使用。

## Index 仓库与 ktra 的关系？

我没有仔细研究过，但我认为：
* Index 仓库只存有 crate 的基本信息（名称、版本、其依赖信息等等，不含代码），步骤五有一个真实示例，或者去
  <https://github.com/rust-lang/crates.io-index> 看看。
* ktra 可视为一个服务端，管理 crate 和 Index，把 crate 存于 ktra 所运行的本地目录下。（cargo publish 打包的数据就是项目源代码）

## 有没有比 ktra 更轻量的选择？

Cargo 支持从 HTTPS/SSH 拉取私人仓库，所以这是最简单的做法，比如：

```toml
[dependencies.my-private-crate]
git = "ssh://xxx"
tag = "xxx"
```

但 git 标签没有 semver。你可以 follow [这个讨论](https://www.reddit.com/r/rust/comments/l0bcmz/whats_the_best_way_to_run_a_private_cargo_registry/)。

## 在项目中混用不同的 registries？

当然可以！

Cargo 把 [crates.io](https://crates.io/) 作为默认的 registry，如果需要指定成 ktra，只需要相关的命令行参数或者配置参数。

步骤五就是示例。但这么做会导致你的项目无法发布到 crates.io 上，因为那上面不支持别的 registries 的库。而 ktra
也不支持从 crates.io 拉过来 —— 不过你自己需要什么，完全可以自定义。
