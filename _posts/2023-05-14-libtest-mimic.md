---
title: Rust 自定义测试：组合型参数
date: 2023-05-14 15:31:09 +0800
categories: [Rust, test]
tags: [Rust, test, codesnippet]
img_path: /assets/img/2023-05-14-libtest-mimic
---

* 使用 `libtest-mimic` 给每组参数生成测试（自定义测试名和测试函数名、自动处理 panic、良好的结果汇总）
* 使用 `itertools` 生成参数的组合

```rust
/*
[dependencies]
itertools = "0.10"
libtest-mimic = "0.6"
*/

use libtest_mimic::{Failed, Trial};

#[test]
fn test() {
    let foo = [1, 2, 3];
    let bar = ['a', 'b', 'c'];
    let baz = ["xxx", "yyy", "zzz"];
    let kind = "combinatorial test";

    let tests = itertools::iproduct!(foo, bar, baz)
        .map(|(a, b, c)| {
            let name = format!("{a}-{b}-{c}");
            // write test code in the closure
            Trial::test(name, move || do_test(a, b, c)).with_kind(kind)
        })
        .collect();
    libtest_mimic::run(&Default::default(), tests).exit()
}

fn do_test(a: i32, b: char, c: &str) -> Result<(), Failed> {
    assert_ne!((a, b, c), (2, 'a', "xxx")); // panic style
    if (a, c) == (1, "zzz") {
        // custom display error
        return Err("Ops...".into());
    }
    Ok(())
}
```

```test
running 27 tests
test [combinatorial test] 1-a-xxx ... ok
test [combinatorial test] 1-a-yyy ... ok
test [combinatorial test] 1-a-zzz ... FAILED
test [combinatorial test] 1-b-xxx ... ok
test [combinatorial test] 1-b-yyy ... ok
test [combinatorial test] 1-b-zzz ... FAILED
test [combinatorial test] 1-c-xxx ... ok
test [combinatorial test] 1-c-yyy ... ok
test [combinatorial test] 1-c-zzz ... FAILED
thread '<unnamed>' panicked at 'assertion failed: `(left != right)`
  left: `(2, 'a', "xxx")`,
 right: `(2, 'a', "xxx")`', src/main.rs:23:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
test [combinatorial test] 2-a-xxx ... FAILED
test [combinatorial test] 2-a-yyy ... ok
test [combinatorial test] 2-a-zzz ... ok
test [combinatorial test] 2-b-xxx ... ok
test [combinatorial test] 2-b-yyy ... ok
test [combinatorial test] 2-b-zzz ... ok
test [combinatorial test] 2-c-xxx ... ok
test [combinatorial test] 2-c-yyy ... ok
test [combinatorial test] 2-c-zzz ... ok
test [combinatorial test] 3-a-xxx ... ok
test [combinatorial test] 3-a-yyy ... ok
test [combinatorial test] 3-a-zzz ... ok
test [combinatorial test] 3-b-xxx ... ok
test [combinatorial test] 3-b-yyy ... ok
test [combinatorial test] 3-b-zzz ... ok
test [combinatorial test] 3-c-xxx ... ok
test [combinatorial test] 3-c-yyy ... ok
test [combinatorial test] 3-c-zzz ... ok

failures:

---- 1-a-zzz ----
Ops...

---- 1-b-zzz ----
Ops...

---- 1-c-zzz ----
Ops...

---- 2-a-xxx ----
test panicked: assertion failed: `(left != right)`
  left: `(2, 'a', "xxx")`,
 right: `(2, 'a', "xxx")`


failures:
    1-a-zzz
    1-b-zzz
    1-c-zzz
    2-a-xxx

test result: FAILED. 23 passed; 4 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

其实终端带有颜色：

![](summary.png)

![](summary-reason.png)
