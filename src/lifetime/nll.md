# 什么是 NLL

> Stackoverflow: [What are non-lexical lifetimes?](https://stackoverflow.com/questions/50251487/what-are-non-lexical-lifetimes)
>
> 由 [@Shepmaster] 提供解答。

[@Shepmaster]: https://github.com/shepmaster

## 没有 NLL 遇到的麻烦

理解 nll (non-lexical lifetimes, 非词法生命周期) 的最简单的方式是先理解词法 ([lexical]) 生命周期。

[lexical]: https://en.wikipedia.org/wiki/Scope_(computer_science)#Lexical_scoping

在 nll 之前的 Rust 版本中，此代码将编译失败：

```rust
fn main() {
    let mut scores = vec![1, 2, 3];
    let score = &scores[0];
    scores.push(4);
}
```

Rust 编译器发现 `scores` 被 `score` 变量借用，因此不允许对 `scores` 进一步改变：

```rust,ignore
error[E0502]: cannot borrow `scores` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:5
  |
3 |     let score = &scores[0];
  |                  ------ immutable borrow occurs here
4 |     scores.push(4);
  |     ^^^^^^ mutable borrow occurs here
5 | }
  | - immutable borrow ends here 
```

然而，人类可以轻易地看到这个例子过于保守：从来没有使用过 `score`！

但问题是，`score` 借用 `scores` 是词法性的，这个借用一直持续到包含它的块的末尾：

```rust
fn main() {
    let mut scores = vec![1, 2, 3]; //
    let score = &scores[0];         //
    scores.push(4);                 //
                                    // <-- score stops borrowing here
}
```

nll 通过增强编译器来理解这一级别的详细信息，从而解决这个问题。

编译器现在可以更准确地判断何时需要借用，并编译此段代码。

nll 的一个奇妙之处在于，一旦启用，就没有人会考虑它们了。它将简单地变成“Rust 所做的事情”，事情将（希望）进展下去。

## 为什么允许词法生命周期

Rust 只允许编译已知安全的程序。然而，只允许安全的程序而拒绝不安全的程序是[不可能的](https://en.wikipedia.org/wiki/Halting_problem)。

对此，Rust 犯了保守的错误：一些安全的程序被拒绝。词法生命周期就是其中一个例子。

在编译器中实现词法生命周期要容易得多，因为对块的了解是“容易的”，而了解数据流就不那么简单了。

编译器需要重写以引入和使用“中级中间表示法” ([MIR])，然后要重写借用检查器 (borrowck) 以使用 MIR 而不是抽象语法树 
(AST)，然后，必须对借用检查器的规则进行细化，以使其更加细粒度。

[MIR]: https://github.com/rust-lang/rfcs/blob/master/text/1211-mir.md

词法生命周期并不总是妨碍程序员，而且在它确实妨碍的情况下也有很多方式来解决，即使确实很烦人。

在许多情况下，通过添加额外的花括号或布尔值来解决。这使得在 nll 实现之前， Rust 1.0 就已经发布并使用了很多年。

有趣的是，某些良好的模式是因词法生命周期而产生的。我觉得最好的例子就是 [`entry`] 模式。以下代码在 nll 出现之前失败，但现在可以编译：

[`entry`]: https://stackoverflow.com/q/28512394/155423

```rust
use std::collections::HashMap;
fn example(mut map: HashMap<i32, i32>, key: i32) {
    match map.get_mut(&key) {
        Some(value) => *value += 1,
        None => {
            map.insert(key, 1);
        }
    }
}
```

但是，这样代码的效率很低，因为它计算 hash 密钥两次。词法生命周期的解决方案更短、更高效，从而成为了解决方案：

```rust
fn example(mut map: HashMap<i32, i32>, key: i32) {
    *map.entry(key).or_insert(0) += 1;
}
```

## 关于 nll 这个名字

值的生命周期是该值停留在特定内存地址的时间跨度（参考《[自引用](self-referential.html)》一章）。

所谓的 nll 特性并不会改变任何值的生命周期，因此它不能使生命周期变成非词法。

nll 只让这些值的借用的跟踪和检查更加精确。

所以，一个更准确的名称可能是“非词法借用” (non-lexical *borrows*)。而一些编译器开发人员称之为底层的“基于 MIR 的借用检查”。

从本质上讲，nll 从来都不是一个“面向用户”的功能。

它的概念在我们的脑海中变得越来越大，因为它不在我们的脑海中，我们得到的是小小的代码碎片。

它的名字主要是为了内部开发目的，而为了宣传目的更改它从来都不是优先事项。

## 如何使用 nll

> 译者注：（即将发布的） Rust 1.63 及其之后的版本已经完全开启了 nll，[不再是迁移模式](https://github.com/rust-lang/rust/issues/58781)。

在 Rust 1.31 （发布于 2018-12-06）中，需要在 `Cargo.toml` 写明 Rust 2018 版本：

```toml
[package]
name = "foo"
version = "0.0.1"
authors = ["An Devloper <an.devloper@example.com>"]
edition = "2018"
```

从 Rust 1.36 开始，Rust 2015 版本也支持 nll。

当前（Rust 1.63 之前） nll 的实现处于“迁移模式”：
* 如果 nll 借用检查器通过，编译将继续
* 如果编译不通过，则调用以前的借用检查器
* 如果旧的借用检查器允许该代码，则会打印一条警告，告知你的代码可能会在未来版本的 Rust 中中断，应进行更新

在 Rust 的夜间版本中，你可以通过一个功能标志选择强制使用 nll：

```rust
#![feature(nll)]
```

甚至可以通过使用编译器标志 `-Z polonius` 来选择使用 nll 的实验版本。

## nll 解决实际问题的例子

（均为 stackoverflow 链接）：

* [Returning a reference from a HashMap or Vec causes a borrow to last beyond the scope it's in?](https://stackoverflow.com/q/38023871/155423)
* [Why does HashMap::get\_mut() take ownership of the map for the rest of the scope?](https://stackoverflow.com/q/32761524/155423)
* [Cannot borrow as immutable because it is also borrowed as mutable in function arguments](https://stackoverflow.com/q/41187296/155423)
* [How to update-or-insert on a Vec?](https://stackoverflow.com/q/47395171/155423)
* [Is there a way to release a binding before it goes out of scope?](https://stackoverflow.com/q/41601197/155423)
* [Cannot obtain a mutable reference when iterating a recursive structure: cannot borrow as mutable more than once at a time](https://stackoverflow.com/q/37986640/155423)
* [When returning the outcome of consuming a StdinLock, why was the borrow to stdin retained?](https://stackoverflow.com/q/43590162/155423)
* [Collaterally moved error when deconstructing a Box of pairs](https://stackoverflow.com/q/28466809/155423)
