---
title: Shell Parameter Expansion
date: 2023-03-12 20:19:35 +0800
categories: [开发日志, linux]
tags: [开发日志, linux, 工具, shell]
---

https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html

例子

```rust
# CI 中运行所有例子
for example in $(ls examples/)
do
  cargo run --example ${example%%.rs}
done
```
