# 基础内容

## 过程宏是什么

过程宏必须定义在 `proc-macro` 类型的库：

```toml
[lib]
proc-macro = true
```

使用过程宏的目的是：在编译期间操作 Rust 的语法，包括消耗和生成 Rust 语法。

过程宏被视为从一个 AST 到另一个 AST 的函数。

像函数一样，过程宏：

1. 要么返回语法：替换或者删除语法
2. 要么导致 panic：编译器捕获这些 panic，并将其转化为编译器的错误信息
3. 要么一直循环：让编译器处于挂起状态 (hanged)


过程宏享受与编译器相同的资源，可以：

1. 访问编译器才能访问的标准输入/输出/错误 ；
2. 可以访问编译期间生成的文件（如同 [`build.rs`](https://doc.rust-lang.org/nightly/cargo/reference/build-scripts.html) 一样）。


过程宏报告错误的方式：

1. panic
2. 调用 [`compile_error`](https://doc.rust-lang.org/nightly/std/macro.compile_error.html)

## `proc_macro` crate

[`proc_macro`](https://doc.rust-lang.org/nightly/proc_macro/index.html) 是 Rust 编译器所提供的，提供了编写过程宏所需的类型和工具。

1. [`TokenStream`](https://doc.rust-lang.org/nightly/proc_macro/struct.TokenStream.html)
    类型是一系列标记，用于迭代 token trees 或者从 token trees 聚集到一起。它几乎等价于 
    `Vec<TokenTree>`（但它的 clone 是低成本的），其中 
    [`TokenTree`](https://doc.rust-lang.org/nightly/proc_macro/enum.TokenTree.html)
    几乎可以被视为词法标记 (lexical token)。
2. ``




参考：

1. [reference#procedural-macros](https://doc.rust-lang.org/nightly/reference/procedural-macros.html)

