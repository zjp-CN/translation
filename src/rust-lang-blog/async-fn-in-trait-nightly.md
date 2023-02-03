# async fn in traits 在 nightly Rust 已经可用

> 原文：[Async fn in trait MVP comes to nightly](https://blog.rust-lang.org/inside-rust/2022/11/17/async-fn-in-trait-nightly.html)
> | 2022 年 11 月 17 日 | 作者：Tyler Mandry，代表 The Rust Async Working Group

异步工作组很高兴地宣布，traits 中的 `async fn` 现在可以在 nightly 编译器中使用。现在可以编写如下代码：

> 译者注：AFIT 是 async fn in traits 的缩写，有时用它来减少累赘的描述。

```rust
#![feature(async_fn_in_trait)]

trait Database {
    async fn fetch_data(&self) -> String;
}

struct MyDb;
impl Database for MyDb {
    async fn fetch_data(&self) -> String { todo!() }
}
```

一个完整的 playground [示例][example]。

我们将介绍一些限制，以及一些已知的 bug，但我们认为一些使用者可以尝试。详细内容请继续阅读。

[example]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=6ffde69ba43c6c5094b7fbdae11774a9

## 回顾 async/await 的工作原理

`async` 和 `.await` 是在 Rust 中编写异步代码的重大改进。在 Rust 中， `async fn` 返回一个 `Future`，它表示正在进行的异步计算的对象。

Future 的类型实际上不会出现在 `async fn` 的签名中。当你编写这样的异步函数时：

```rust,ignore
async fn fetch_data(db: &MyDb) -> String { ... }
```

编译器将其重写为如下内容：

```rust,ignore
fn fetch_data<'a>(db: &'a MyDb) -> impl Future<Output = String> + 'a {
    async move { ... }
}
```

这个“脱糖”的签名你可以自己也可以写，它对检查背后发生的事情很有用。此处的 `impl Future` 语法表示实现 Future 的某种不透明类型 (opaque type)。

Future 是一个状态机，负责知道下次醒来时如何继续下去。

当你在 `async` 块中编写代码时，编译器会为你生成特定于该异步块的 Future 类型。此 Future
类型没有名称，因此我们必须在函数签名中使用不透明类型。

## AFIT 的历史问题

trait 是 Rust 中抽象的基本机制。那么，如果你想将异步方法放在一个 trait 中，会发生什么呢？

每个异步块或函数都会创建一个唯一的类型，因此你希望表达每个实现可针对返回类型而具有不同的 Future。谢天谢地，我们有关联类型：

```rust,ignore
trait Database {
    type FetchData<'a>: Future<Output = String> + 'a where Self: 'a;
    fn fetch_data<'a>(&'a self) -> FetchData<'a>;
}
```

请注意，此关联类型是泛型的。Rust [直到现在][until now] 才支持泛型关联类型。。。不幸的是，即使使用 GAT，你仍然无法编写使用“异步”的 trait 实现：

[until now]: https://blog.rust-lang.org/2022/10/28/gats-stabilization.html

```rust,ignore
impl Database for MyDb {
    type FetchData<'a> = /* what type goes here??? */;
    fn fetch_data<'a>(&'a self) -> FetchData<'a> { async move { ... } }
}
```

由于无法命名异步块构造的类型，因此唯一的选择是使用不透明类型（我们前面看到的 `impl Future`）。但关联类型不支持 `impl Trait`[^tait]。

[^tait]: 这个功能被称为 [type alias impl trait](https://rust-lang.github.io/rfcs/2515-type_alias_impl_trait.html)，简称 TAIT。

### stable 中可用的次优方案

因此，要在 stable Rust 中给 trait 编写 `async fn`，我们需要在实现中指定一个具体类型。今天有几种方法可以做到这一点。

#### 运行时类型擦除

首先，我们可以通过用 `dyn` 擦除 Future 类型来避免写明它。以上面的例子为例，你可以这样写：

```rust,ignore
trait Database {
    fn fetch_data(&self)
    -> Pin<Box<dyn Future<Output = String> + Send + '_>>;
}
```

这明显更加冗长，但它实现了将异步与 trait 相结合的目标。此外，[`async-trait`] 库可为你重写为上述代码，让你只需简单地编写

```rust,ignore
#[async_trait]
trait Database {
    async fn fetch_data(&self) -> String;
}

#[async_trait]
impl Database for MyDb {
    async fn fetch_data(&self) -> String { ... }
}
```

对于能够使用它的人来说，这是一个极好的解决方案！

不幸的是，并不是每个人都能使用它：
* 你无法在 no_std 环境下使用 `Box`
* 动态分发和分配带来的开销对于高度性能敏感的代码来说可能是 [巨大的][overwhelming]
* 最后，它本身蕴含了许多假设：使用 `Box` 分配、动态分发和 Future 的 `Send` trait。这使得它不适合许多库。

此外，使用者 [希望][expect] 能够在 traits 中编写 `async fn`，而添加外部库依赖项的体验会造成 async Rust 难以使用的印象。

[overwhelming]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_benchmarks_async_trait.html
[expect]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_needs_async_in_traits.html

#### 手动实现 `poll`

还有另一种选择能在零开销或 no-std 环境中让这种 trait 工作：它们可以从 [`Future`] trait 中利用轮询的概念，并将其直接构建到接口中。

如果 Future 已完成，则 `Future::poll` 方法返回 `Poll::Ready(Output)`；如果 Future 正在等待其他事件，则返回 `Poll::Pending`。

例如，你可以在当前不稳定的 [`AsyncIterator`] trait 中看到这种模式。

`AsyncIterator::poll_next` 的签名结合了 `Future::poll` 和 `Iterator::next`。

[`Future`]: https://doc.rust-lang.org/stable/std/future/trait.Future.html
[`AsyncIterator`]: https://doc.rust-lang.org/stable/std/async_iter/trait.AsyncIterator.html

```rust,ignore
pub trait AsyncIterator {
    type Item;

    fn poll_next(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;
}
```

在 async/await 出现之前，编写手动 `poll` 实现是非常常见的。不幸的是，他们很难写正确。

在去年的 [愿景文档][vision] 中，我们收到了许多关于这是如何 [极其困难] 的报告，以及 Rust 使用者的 [bugs 来源]。

事实上，很难手动编写轮询实现是在核心语言中添加 async/await 的主要原因。

[vision]: https://blog.rust-lang.org/2021/03/18/async-vision-doc.html
[极其困难]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_hates_writing_a_stream.html
[bugs 来源]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_polls_a_mutex.html

## nightly 支持的内容

[landed]: https://github.com/rust-lang/rust/pull/101224

我们一直在努力在 traits 中直接支持 `async fn`，[最近一个实现][landed] 在 nightly 上发布了！

该功能仍有一些粗糙的边缘情况，但让我们看看你可以使用它做什么。

首先，正如你所期望的那样，你可以编写和实现如上所述的 trait。

```rust
#![feature(async_fn_in_trait)]

trait Database {
    async fn fetch_data(&self) -> String;
}

struct MyDb;
impl Database for MyDb {
    async fn fetch_data(&self) -> String { todo!() }
}
```

这将允许我们做的一件事是标准化我们一直在等待的新 trait。例如，上面的 `AsyncIterator` trait 比其类似的
`Iterator` 要复杂得多。有了新的支持，我们可以简单地改为：

```rust
#![feature(async_fn_in_trait)]

trait AsyncIterator {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}
```

标准库中的 trait 很有可能会变成这样！现在，你还可以使用 [`async_iterator`] 库中的迭代器，并使用它编写泛型代码，就像你通常所做的那样。

[`async_iterator`]: https://docs.rs/async-iterator/latest/async_iterator/

```rust,ignore
async fn print_all<I: AsyncIterator>(mut count: I)
where
    I::Item: Display,
{
    while let Some(x) = count.next().await {
        println!("{x}");
    }
}
```

### 限制 1：从泛型 `spawn`

有一个陷阱！如果你试图从 `print_all` 这样的泛型函数 spawn 任务，并且（像大多数异步使用者一样）使用工作窃取执行器
(work stealing executor)，此执行器要求任务是 `Send`，则会遇到一个不容易解决的错误。[^actual-error]

[playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=a79464992e284099166a6b91fe9b1098)

```rust,ignore
fn spawn_print_all<I: AsyncIterator + Send + 'static>(mut count: I)
where
    I::Item: Display,
{
    tokio::spawn(async move {
        //       ^^^^^^^^^^^^
        // ERROR: future cannot be sent between threads safely
        while let Some(x) = count.next().await {
            //              ^^^^^^^^^^^^
            // note: future is not `Send` as it awaits another future which is not `Send`
            println!("{x}");
        }
    });
}
```

[^actual-error]: 编译器产生的实际错误消息比这更嘈杂，但这会得到改善。（译者注：我翻译时看到错误消息已经不同，点击 playground 自己试试）

你可以看到，函数签名中添加了 `I: Send` 约束，但这还不够。为了解决这个错误，我们需要表达 `next` 返回的 Future 是 `Send`。

但正如在本文开头看到的，异步函数返回匿名类型，从而无法表达这些类型的约束。

我们将在后续文章中探讨这个问题的潜在解决方案。但目前，你可以做一些事情来摆脱这种情况。

#### 假设：这并不常见

首先，你可能会惊讶地发现，这种情况在实践中并不经常发生。

[例如][Countdown]，我们可以生成 (spawn) 一个调用上述 `print_all` 函数的任务，而不会出现任何问题：

[Countdown]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=6ffde69ba43c6c5094b7fbdae11774a9

```rust,ignore
async fn do_something() {
    let iter = Countdown::new(10);
    executor::spawn(print_all(iter));
}
```

这没有任何 `Send` 约束！这是因为 `do_something` 知道迭代器的具体类型为 `Countdown`。编译器知道此类型是 `Send`，因此
`print_all(iter)` 生成一个 `Send` 的 Future。[^auto-traits-special]

[^auto-traits-special]: 像 `Send` 和 `Sync` 这样的 auto traits 在这方面很特殊。当且仅当其参数为 `Send` 的类型时，编译器知道
`print_all` 的返回类型是 `Send`；并且与常规 traits 不同，编译器允许在类型检查程序时使用此 auto traits 知识。

一种假设是，虽然人们会遇到这个问题，但他们遇到这个问题的频率相对较低，因为大多数时候，不会在含异步函数的 trait 的泛型代码中调用 `spawn`。

我们想开始收集人们在这方面的实际经验。如果你有相关的经验可以分享，请再此留下
[评论](https://github.com/rust-lang/rust/issues/103854)。

#### 当它的确发生时

最终，你可能希望从一个异步 trait 的泛型上下文中 spawn 任务，。那怎么办！？

现在，可以使用另一个新的 nightly 专用功能 `return_position_impl_trait_in_trait` 来直接在 trait 中表达你需要的约束：

```rust
#![feature(return_position_impl_trait_in_trait)]

trait SpawnAsyncIterator: Send + 'static {
    type Item;
    fn next(&mut self) -> impl Future<Output = Option<Self::Item>> + Send + '_;
}
```

这里，我们将 `async fn` 脱糖为返回 `impl Future + '_` 的常规函数，它的工作方式与普通的 `async fn` 相同。

但它更详细，为我们提供了一个放置 `+ Send` 约束的位置！此外，你可以继续在这种 trait 的 `impl` 中使用 `async fn`。

这种方法的缺点是这样的 trait 变得不那么泛型，使得它不太适合于库代码。

如果你想维护一个 trait 的两个独立版本，可以这样做，并且（也许）提供宏以使两者更容易实现。

此解决方案是临时的。我们希望在语言本身中实现一个更好的解决方案，但由于这是 nightly 专属的功能，我们希望尽快让更多人尝试。

### 限制 2：动态分发

最后一个限制是：不能用 `dyn Trait` 调用 `async fn`。

支持这一点的设计已经存在[^dyn-designs]，但还处于早期阶段。如果你需要从 trait 中动态分发，那么现在最好使用 [`async-trait`] 宏。

[^dyn-designs]: 请参见 [AFIT 提议网站][AFIT-initiative]，以及 Niko [系列文章][Niko] 的第 8 和第 9 篇文章（译者注：[中文翻译][series-chinese]在这！）。

[AFIT-initiative]: https://rust-lang.github.io/async-fundamentals-initiative/explainer/async_fn_in_dyn_trait.html
[Niko]: https://smallcultfollowing.com/babysteps/blog/2022/09/21/dyn-async-traits-part-9-callee-site-selection/
[series-chinese]: ../dyn-async-traits/2022-09-18-dyn-async-traits-part-8-the-soul-of-rust.html

## 稳定之路不远

异步工作组希望从 Rust 使用者手中得到一些有用的东西，即使 Rust 没有做到他们可能想要的一切。

这就是为什么尽管有一些限制，但当前版本的 AFIT 可能离稳定不远了[^stabilization-when]。

你可以通过查看 [跟踪问题][tracking] 来跟踪进度。

[tracking]: https://github.com/rust-lang/rust/issues/91611

[^stabilization-when]: 什么时候？可能在接下来六个月之后的某个时候。但不要强迫我：）

首先要回答两个主要问题：

* 我们是否需要先解决“从泛型 spawn”（`Send` 约束）问题？请留下关于此问题的 [反馈](https://github.com/rust-lang/rust/issues/103854)。
* 还存在哪些错误和质量问题？请为此 [提交新问题][new-issue]。你可以在这里查看 [已知的问题][known-issues]。

如果你是一个异步 Rust 爱好者，并且愿意尝试实验性的新功能，我们将非常感谢你试一试！

[new-issue]: https://github.com/rust-lang/rust/issues/new/choose
[known-issues]: https://github.com/rust-lang/rust/labels/F-async_fn_in_trait

如果使用 [`#[async_trait]`][`async-trait`]，可以尝试从一些不使用动态分发的 traits（及其 impl）中删除它。

或者，如果你正在编写新的异步代码，请尝试在那里使用它。无论哪种方式，你都可以在上面的链接中告诉我们你的经历。

## 结论

这项工作之所以成为可能，得益于许多人的努力，包括

* Michael Goulet
* Santiago Pastorino
* Oli Scherer
* Eric Holk
* Dan Johnson
* Bryan Garza
* Niko Matsakis
* Tyler Mandry

此外，它是在多年编译器工作的基础上构建的，这些工作使我们能够使用 [GATs] 以及其他基本的类型系统改进。

我们深深感谢所有贡献的人；没有你们，这项工作是不可能的。非常感谢。

要了解更多关于这个功能及其背后的挑战，请查看 [RFC 3185] 和 [为什么 AFIT 这么难][why async fn in traits are hard] 。

此外，请继续关注后续文章，我们将探讨语言扩展，使其能够在没有特殊 trait 的情况下表达“发送”界限。

感谢 Yoshua Wuyts、Nick Cameron、Dan Johnson、Santiago Pastorino、Eric Holk 和 Niko Matsakis 审阅了本文的草稿。

[`async-trait`]: https://docs.rs/async-trait
[GATs]: https://blog.rust-lang.org/2022/10/28/gats-stabilization.html
[RFC 3185]: https://rust-lang.github.io/rfcs/3185-static-async-fn-in-trait.html
[why async fn in traits are hard]: https://smallcultfollowing.com/babysteps/blog/2019/10/26/async-fn-in-traits-are-hard/
