# trait object 中的异步函数

> 原文：[Async Functions in Trait Objects Update](https://blog.theincredibleholk.org/blog/2022/12/19/async-fn-in-trait-object-update/)
> | by Eric Holk | 2022 年 12 月 19 日

2022 年即将结束，我想花点时间看看我们在 dyn traits 中支持异步函数的情况（AFIDT），并就如何在 2023 年取得进展提出一些建议。

我们希望使以下代码片段工作：

```rust,ignore
trait AsyncCounter {
    async fn increment_by(&mut self, amount: usize);
    async fn get_value(&self) -> usize;
}

async fn use_counter(counter: &mut dyn AsyncCounter) {
    counter.increment_by(42).await;
    counter.get_value().await
}
```

通过本文，我们将看到离实现目标有多近，还有什么可以让它到达终点线。

## 当前状态

### traits 中的静态异步函数

感谢 [Michael Goulet] 的出色工作，我们在 nightly 编译器中支持静态分发的异步函数。

这意味着我们只需一次修改就可以编译最初的示例，将 `dyn` 替换为 `impl`，[例子][example-AFIT]。

[Michael Goulet]: https://www.errs.io/
[example-AFIT]: https://play.rust-lang.org/?version=nightly&mode=release&edition=2021&gist=ac0a4a2da0bed3724b5a43c5f04cec0c

```rust
#![feature(async_fn_in_trait)]

trait AsyncCounter {
    async fn increment_by(&mut self, amount: usize);
    async fn get_value(&self) -> usize;
}

async fn use_counter(counter: &mut impl AsyncCounter) -> usize {
    counter.increment_by(42).await;
    counter.get_value().await
}
```

这个功能很强大，而且考虑到 Rust 中泛型似乎比 trait 对象更常见，我觉得这提供了人们从 trait 异步函数中寻找的大部分价值。

### `dyn*`

下一步目前正在研究的是一种叫做 `dyn* Trait` 的新型 trait 对象。

这为我们提供了一部分方案来解决异步函数返回值需要作为对象进行使用。

回想一下：
* `async fn foo() -> usize` 是 `fn foo() -> impl Future<Output = usize>` 的语法糖
* 在 trait 中， `impl Future<Output = usize>` 成为 trait 中的关联类型
* 在静态情况下，每个 impl 都可以有自己的返回值类型
* 但在动态情况下，我们需要有一个适用于所有可能的 impl 的类型

换句话说，异步函数的返回类型此时需要是动态的。

[`async-trait`]: https://crates.io/crates/async-trait 

我们可以通过把返回值放入 `Box` 来做到这一点。这就是 [`async-trait`] 库所做的，因为任何异步 trait 方法都会返回 `Box<dyn Future>`。

这种方法有一些缺点，我们希望避免这些缺点融入到语言中：这样做意味着调用异步方法总是会导致堆分配。

这可能是一个性能问题，但更重要的是，这意味着该功能在没有内存分配器的情况下将无法使用，这本身率先打破现有的 Rust 功能。

我们想要的是一种返回值的方法，这个值实现了 `Future`，但这就是你所知道的一切。

trait 对象让我们接近这一点，但你总是在指针后面使用它们。如果我们使用 `Box<dyn Future>` ，情况将与之前相同。

我们可以尝试返回 `&mut dyn Future`，但我们不知道返回值是从哪里借来的，也不知道谁负责解决它。

`dyn*` 通过隐藏它是什么类型的指针（或者即使它是指针）来解决这些问题。你所知道的是，这个对象实现了
trait，并且你需要在完成任何清理时调用一些 drop 函数。因此，使用 `dyn* Future`
作为异步方法的返回类型，可以满足我们的所有要求，并使 trait 对象安全。

总之，目标已经清楚了。这里重要的是，我们有一个 `dyn*` 的实验实现，也可以在 nightly 编译器中工作。

## 还剩什么

### 类型系统基础

在我们完全实现 trait 对象中的异步函数之前，需要对类型系统进行更多的扩展。为了更好地说明这一点，我想从概念上展示编译器如何处理异步方法。

从 `async fn` 方法开始，就像我们在文章开头所做的那样。

```rust,ignore
trait AsyncCounter {
    async fn increment_by(&mut self, amount: usize);
    async fn get_value(&self) -> usize;
}
```

下一步是去掉 `async` 关键字，并将返回值更改为 `impl Future` 。

```rust,ignore
trait AsyncCounter {
    fn increment_by(&mut self, amount: usize) -> impl Future<Output = ()>;
    fn get_value(&self) -> impl Future<Output = usize>;
}
```

编译器处理 `-> impl Trait` 方法，就像每个方法返回值的关联类型。

```rust,ignore
trait AsyncCounter {
    type IncrementBy<'a>: Future<Output = ()> + 'a
    where
        Self: 'a;
    fn increment_by<'a>(&'a mut self, amount: usize) -> Self::IncrementBy<'a>;

    type GetValue<'a>: Future<Output = usize> + 'a
    where
        Self: 'a;
    fn get_value<'a>(&'a self) -> Self::GetValue<'a>;
}
```

一旦我们达到这一步，理论上可以通过显式地将所有关联类型替换为 `dyn*` 类型来创建该 trait 的 dyn-safe 版本。结果如下：

```rust,ignore
async fn use_counter(
    counter: &mut dyn AsyncCounter<
        IncrementBy = dyn* Future<Output = ()>,
        GetValue = dyn* Future<Output = usize>,
    >,
) -> usize {
    counter.increment_by(42).await;
    counter.get_value().await
}
```

这有点啰嗦，而且也是错误的。根据编译器，我们忘记了生命周期参数。

```rust,ignore
error[E0107]: missing generics for associated type `AsyncCounter::IncrementBy`
  --> src/lib.rs:20:9
   |
20 |         IncrementBy = dyn* Future<Output = ()>,
   |         ^^^^^^^^^^^ expected 1 lifetime argument
   |
note: associated type defined here, with 1 lifetime parameter: `'a`
  --> src/lib.rs:7:10
   |
7  |     type IncrementBy<'a>: Future<Output = ()> + 'a
   |          ^^^^^^^^^^^ --
help: add missing lifetime argument
   |
20 |         IncrementBy<'_> = dyn* Future<Output = ()>,
   |         ~~~~~~~~~~~~~~~
```

（类似于 `AsyncCounter::GetValue`）

那么，我们使用什么作为生命周期参数？

事实证明，我们还不能在 Rust 中说出我们想要什么。我们想要的是这样的：

```rust,ignore
async fn use_counter(
    counter: &mut dyn AsyncCounter<
        for<'a> IncrementBy<'a> = dyn* Future<Output = ()> + 'a,
        for<'a> GetValue<'a> = dyn* Future<Output = usize> + 'a,
    >,
) -> usize {
    counter.increment_by(42).await;
    counter.get_value().await
}
```

但这甚至不能解析，更不用说在类型检查器 (type checker) 中支持它了。

最接近的解决办法是启用 [`generic_associated_types_extended`] feature，然后可以编写以下内容：

```rust,ignore
async fn use_counter(
    counter: &mut impl for<'a> AsyncCounter<
        IncrementBy<'a> = dyn* Future<Output = ()> + 'a,
        GetValue<'a> = dyn* Future<Output = usize> + 'a,
    >,
) -> usize {
    counter.increment_by(42).await;
    counter.get_value().await
}
```

但这里的 `for<'a>` 并不是我们想要的，而且我已经将其中一个 `dyn` 转换为 `impl` 。

据我所知，`generic_associated_types_extended` feature 基本上是我们将来可能添加到 GATs 中的占位符。

我不熟悉这些扩展可能是什么，所以这肯定是我很快就会深入研究的领域。

据我所知，该功能目前禁用了一些错误，但这意味着它几乎肯定是非常 unsound。这里应该有很多好玩的地方！

[`generic_associated_types_extended`]: https://github.com/rust-lang/rust/issues/95451 

### 人机工程学

理论上，如果我们有类型支持，就可以在 trait 对象中手写异步函数，但这会很痛苦。这基本上相当于一直在写上一节中的低级别的示例。

因此，要使这成为一个令人愉快的功能，需要进行大量的润色工作。我认为这里有一些开放的设计空间，所以列出一些我看到的可能性。

#### auto-dyn Traits

嗯，auto-dyn 听起来像是终结者宇宙中的 [邪恶公司][nefarious corporation]。总之，我们在上面看到带异步方法的 trait 不是 dyn-safe。

但是，如果你为所有关联类型提供值，它们就会变得 dyn-safe。这是我们在这个代码示例中所做的：

[nefarious corporation]: https://terminator.fandom.com/wiki/Cyberdyne_Systems

```rust,ignore
async fn use_counter(
    counter: &mut dyn AsyncCounter<
        IncrementBy = dyn* Future<Output = ()>,
        GetValue = dyn* Future<Output = usize>,
    >,
) -> usize {
    counter.increment_by(42).await;
    counter.get_value().await
}
```

不幸的是，如果我们使用了 `async fn increment_by(...)` 之类的方法，那么目前还无法命名该方法的返回类型。

因此，我们可以添加语法来实现这一点，它可能看起来像这样[^return-type-syntax]：

```rust,ignore
async fn use_counter(
    counter: &mut dyn AsyncCounter<
        increment_by(..) = dyn* Future<Output = ()>,
        get_value(..) = dyn* Future<Output = usize>,
    >,
) -> usize {
    counter.increment_by(42).await;
    counter.get_value().await
}
```

这个功能在其他场景中可能很有用，所以我希望我们能得到一些类似的功能，但在每个异步 trait 上都会变得冗长。

但是我们也可以让编译器为我们自动执行此操作。对于每个异步方法，编译器可以给返回类型上的隐式关联类型设置正确的 `dyn*` 类型。[^dyn-star-rpitit]

然后使用者但愿很少需要知道发生了的转换。


[^return-type-syntax]: 我不是这个所提出的语法的真正粉丝，但我也没有看到我更喜欢的语法。

[^dyn-star-rpitit]: 我们可能也希望对任何返回 `impl Trait` 的函数进行类似的转换。

如果编译器进行了这种转换，那么事情就应该正常了，从而可以编写这样的代码：

```rust,ignore
async fn use_counter(counter: &mut dyn AsyncCounter) -> usize {
    counter.increment_by(42).await;
    counter.get_value().await
}
```

这正是我们在这篇文章开头说我们想要的。

#### dyn-safe impls

到目前为止，我们只关注了一旦有人给了你一个 `dyn AsyncCounter`，如何使用它。

但是我们也需要能够创建这样的对象，否则在本文中看到的所有其他代码都将是无用的。

理想情况下，我们可以像其他东西一样编写自己的 impl：

```rust,ignore
impl AsyncCounter for MyCounter {
    async fn increment_by(&mut self, amount: usize) {
        ...
    }

    async fn get_value(&self) -> usize {
        ...
    }
}
```

然后可以这样调用 `use_counter`：

```rust,ignore
async fn call_dyn_use_counter(counter: &mut MyCounter) -> usize {
    use_counter(counter).await
}
```

在这段代码中，对 `use_counter(counter)` 的调用将获取 `&mut MyCounter`（即 `counter` ），并将其强制转换为
`&mut dyn AsyncCounter`，从而将其传递给 `use_counter`。但这样做可能会带来错误。

原因是，`use_counter` 对于 `AsyncCounter` 的所有关联类型都需要 `dyn*` 类型，而
`impl AsyncCounter for MyCounter` 具有不透明的 Future 类型。

有一个次优办法。traits 方法返回位置上的 impl Trait（Return Position Impl Trait In Traits，简写为
RPITIT）的当前实现允许你在实现 trait 时使用更具体的返回类型，因此你可以通过指定 `dyn*`
作为返回类型（或任何其他适合你的具体 Future 类型）来确保你的 impl 是 dyn-safe。这将让我们写：

```rust,ignore
impl AsyncCounter for MyCounter {
    fn increment_by(&mut self, amount: usize) -> dyn* Future<Output = ()> + '_ {
        async { ... }
    }

    fn get_value(&self) -> dyn* Future<Output = usize> + '_ {
        async { ... }
    }
}
```

如果你尝试编译这个，至少现在可以得到一个漂亮的类型检查循环。

<details>
  <summary>完整的类型检查循环</summary>

```rust,ignore
error[E0391]: cycle detected when type-checking `::increment_by`
  --> src/main.rs:19:5
   |
19 |     fn increment_by(&mut self, amount: usize) -> dyn* Future + '_ {
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: ...which requires computing layout of `[async block@src/main.rs:20:9: 20:39]`...
note: ...which requires optimizing MIR for `::increment_by::{closure#0}`...
  --> src/main.rs:20:9
   |
20 |         async move { ... }
   |         ^^^^^^^^^^^^^^^^^^
note: ...which requires elaborating drops for `::increment_by::{closure#0}`...
  --> src/main.rs:20:9
   |
20 |         async move { ... }
   |         ^^^^^^^^^^^^^^^^^^
note: ...which requires borrow-checking `::increment_by::{closure#0}`...
  --> src/main.rs:20:9
   |
20 |         async move { ... }
   |         ^^^^^^^^^^^^^^^^^^
note: ...which requires processing MIR for `::increment_by::{closure#0}`...
  --> src/main.rs:20:9
   |
20 |         async move { ... }
   |         ^^^^^^^^^^^^^^^^^^
note: ...which requires preparing `::increment_by::{closure#0}` for borrow checking...
  --> src/main.rs:20:9
   |
20 |         async move { ... }
   |         ^^^^^^^^^^^^^^^^^^
note: ...which requires unsafety-checking `::increment_by::{closure#0}`...
  --> src/main.rs:20:9
   |
20 |         async move { ... }
   |         ^^^^^^^^^^^^^^^^^^
note: ...which requires building MIR for `::increment_by::{closure#0}`...
  --> src/main.rs:20:9
   |
20 |         async move { ... }
   |         ^^^^^^^^^^^^^^^^^^
note: ...which requires building THIR for `::increment_by::{closure#0}`...
  --> src/main.rs:20:9
   |
20 |         async move { ... }
   |         ^^^^^^^^^^^^^^^^^^
note: ...which requires type-checking `::increment_by::{closure#0}`...
  --> src/main.rs:20:9
   |
20 |         async move { ... }
   |         ^^^^^^^^^^^^^^^^^^
   = note: ...which again requires type-checking `::increment_by`, completing the cycle
   = note: cycle used when type-checking all item bodies
For more information about this error, try rustc --explain E0391.

The problem is that to return a future as a dyn*, the future must be PointerSized, but the compiler can't figure that out. It might be that we could be more precise and break the cycle here3, but even if we could most interesting futures are not going to be PointerSized.

So instead we can solve this by wrapping the body in something that is PointerSized, like a Box.

impl AsyncCounter for MyCounter {
    fn increment_by(&mut self, amount: usize) -> dyn* Future<Output = ()> + '_ {
        Box::pin(async { ... })
    }

    fn get_value(&self) -> dyn* Future<Output = usize> + '_ {
        Box::pin(async { ... })
    }
}
```
 
</details>

有关此错误的详细信息，请尝试 `rustc --explain E0391`。

问题是，要将 Future 作为 `dyn*` 返回，Future 必须是 `PointerSized`，但编译器无法确定这一点。

也许我们可以更加精确，打破这一循环[^break-cycle]，但即使我们可以做到，最有趣的 Future 也不会是 `PointerSized` 。

[^break-cycle]: 我怀疑...

因此，我们可以通过用 `PointerSized` （指针大小）的东西（如 `Box` ）进行包装来解决这个问题。

```rust,ignore
impl AsyncCounter for MyCounter {
    fn increment_by(&mut self, amount: usize) -> dyn* Future<Output = ()> + '_ {
        Box::pin(async { ... })
    }

    fn get_value(&self) -> dyn* Future<Output = usize> + '_ {
        Box::pin(async { ... })
    }
}
```

这将编译（如果你添加了足够的 feature 标志，现在确实可以编译一些类似的东西），但这不是很好。

这比我想的还要冗长。这也意味着你必须决定何时执行该实现，它是否将是 dyn-safe。

最后，即使在静态分发环境中使用，dyn-safe 的 impl 也会支付该开销，这违反了 Rust 的零成本抽象原则。

#### dyn-safe 包装器

一种在编写更多代码的同时兼顾这两个方面的方法是添加 dyn-safe 包装器 (wrappers)。

其想法是编写一个结构体，该结构体包装了实现你的 trait 的任何内容，但将返回值调整为 `dyn* Future`。这样的包装器看起来像这样：

```rust,ignore
struct DynSafeAsyncCounter<T>(T)

impl<T: AsyncCounter> AsyncCounter for DynSafeAsyncCounter<T> {
    fn increment_by(&mut self, amount: usize) -> dyn* Future<Output = ()> + '_ {
        Box::pin(self.0.increment_by(usize))
    }

    fn get_value(&self) -> dyn* Future<Output = usize> + '_ {
        Box::pin(self.get_value())
    }
}
```

然后，为了将任何实现 `AsyncCounter` 的东西强制转换为 dyn-safe 版本，我们只需将其包装为 `DynSafeAsyncCounter`，如下所示：

```rust,ignore
async fn call_dyn_use_counter(counter: &mut MyCounter) -> usize {
    use_counter(DynSafeAsyncCounter(counter)).await
}
```

虽然有点冗长，特别是在包装器的 impl 实现那边，但这种方法也有一些额外的功能。

例如，我们可以将其与 [分配器 API][allocators] 或 [存储 API][storage]
相结合，以在堆分配不受欢迎或不可用的环境中启用 arena 分配或栈分配的 Futures 等功能。

[allocators]: https://github.com/rust-lang/rust/issues/32838
[storage]: https://github.com/matthieu-m/storage-poc

#### auto-dyn 包装器

上一节中的手动包装器非常适合查看幕后发生的事情，使用者可能更愿意在大多数时间不必编写所有的样板。

幸运的是，在很多情况下，这也是一种可以自动化的事情。

我们可以这样做的一种方式是使用 [`Boxing`] 适配器 (adapter)[^adapter]。

使用 `Boxing` 看起来与上面的 `DynSafeAsyncCounter` 包装器非常相似：

[^adapter]: 译者注：有时我翻译成了“转换器”。

[`Boxing`]: https://hackmd.io/RjycNB6wQbi3go5TSg-OIw#Boxing

```rust,ignore
async fn call_dyn_use_counter(counter: &mut MyCounter) -> usize {
    use_counter(Boxing::new(counter)).await
}
```

然而，重要的区别在于，我们没有为可能想要转换的每个 trait 提供不同的包装。

相反， `Boxing` 适用于所有 trait。你会问这怎么做到的？当然，编译器很神奇！

理论上，这种方法还允许像 `InlineAdapter` 这样的东西将返回的 Futures 存储在栈上，而不是堆上。

不幸的是，由于这种方法依赖于编译器的魔法，在实践中，我们将仅限于几种适配器类型，因为每个适配器类型都需要内置到编译器中。

可能还有其他方法可行，但真正强大的方法可能需要新的语言 features。

#### higher order impls

当我看到编译器或标准库可以做程序员无法用这种语言做的事情时，我常常认为这表明该语言缺少某些功能。

让我们回到使 `Boxing` 包装器工作所需的编译器魔法。

我们将想象 `Boxing::new` 是一个函数，它接受 `T` 并给你一个 `Boxing<T>` （希望不是一个不合理的假设）。

在使用 `Boxing::new(counter)` 的示例中，我们不必像使用 `DynSafeAsyncCounter` 那样在任何地方提到
`AsyncCounter`，原因是使用 `Boxing` 时，对于 `T` 实现的任何 trait，编译器都会为 `Boxing<T>` 生成该 trait 的 impl。

我们今天没有办法在 Rust 中表达这一点，但如果我们做得到，可能会是这样的：

```rust,ignore
impl<trait Trait, T: Trait> Trait for Boxing<T> {
    // What goes here???
}
```

这个示例所做的是创建一种新的泛型参数。我们已经可以使用类型参数（ `T` ）、生命周期参数（ `'a` ）和常量参数（
`const X` ），因此这将为 trait 参数（ `trait Trait` ）添加一种新的类型。然后，我们将类型参数约束为必须实现我们正在抽象的
`Trait` ，然后我们再展示如何为 `Boxing<T>` 实现 `Trait`。

不过，还不清楚 `impl` 主体内到底有什么。在一个普通的 impl 中，我们会列出所有的方法，并提供这些方法的实现。

问题是，我们对 `trait` 一无所知，只知道它是一种 trait，所以我们不能列出任何方法。

相反，我们需要告诉编译器如何根据给定的任何 trait 和另一种类型的实现生成实现。

我们可以通过转换方法找到一条很长的路，这些方法向你展示如何将 `Self` 类型转换为 `impl Trait`。

对于 dyn-safe 的异步 trait 的用例，我们还希望某种包装器在返回值上运行，将 boxing 和强制转换变为 `dyn*`。

我们可能能够将任何关联的类型和常量转发到背后的 trait （例如 `type AssociatedType = T::AssociatedType`），但这可能不是我们在所有情况下都想要的。

根据这一结论，我们可能会想办法 hook 满足特定条件的任何方法、类型和常量，并对它们应用某种转换。

如果我们将这些转换称为“建议” ，那么这个功能听起来很像面向切面编程 ([aspect-oriented programming])。

但也许我们也可以从其他地方获得灵感。例如，Haskell 有针对 newtype 的广义派生实例
([generalised derived instances for newtypes]) 和 [deriving via]，两者听起来都可能是相关的。

我怀疑这将是一个非常粗糙的功能，很难证明其合理性。但如果我们能够做到这一点，它为我们提供了一个强大的解决方案，解决了
dyn-safe 包装器问题，该问题也适用于其他领域。例如，现在像 `Box`、`Rc` 和 `Arc` 这样的智能指针并不能实现它们所指向的对象的 trait。

有很多 trait 的专属转发 impl，如 `Debug`、 `Display`、 `Fn*`、 `Read`、 `Write` 和 `Future` 等，但这些都需要手动添加。

在许多情况下，这并不明显，因为方法调用语法可以使用 `Deref` 和 `DerefMut` 获得指针，但有时确实会导致问题。

对于更高阶的 impls，我们可以添加 `impl<trait Trait, T: Trait> Trait for Box<T>` 或类似的东西。

[aspect-oriented programming]: https://en.wikipedia.org/wiki/Aspect-oriented_programming
[generalised derived instances for newtypes]: https://ghc.gitlab.haskell.org/ghc/doc/users_guide/exts/newtype_deriving.html
[deriving via]: https://ghc.gitlab.haskell.org/ghc/doc/users_guide/exts/deriving_via.html

## 结论

我真的很高兴 2022 年在 Rust 中实现异步方面取得的进展，特别是在 traits 中的异步函数方面！在写这篇文章时，我意识到还有很多有趣的工作要做。

我知道我个人很难把大局放在心上。我以前认为，“好吧，我们支持实验性的 `dyn*` 和 AFIT，所以剩下的都是细节”。

写这篇文章对我澄清一些我意识到的缺失的东西很有帮助，我希望它对其他人也有同样的帮助。

我肯定我错过了一些东西，而且肯定有几个设计选项可供选择，所以我希望得到关于缺少什么以及哪些方法最能满足使用者需求的反馈。

如果你想参与进来，请在 [Zulip] 联系！

[Zulip]: https://rust-lang.zulipchat.com/#narrow/stream/187312-wg-async

关于目前的优先事项，我认为主要有两件事。

最重要的是找出类型系统的含义并启用任何缺失的功能。这将允许我们手动或使用一些宏在 trait
对象中编写异步函数。这意味着语言具有我们想要的力量，即使它不符合人机工程学。

其次，我认为剩下的设计问题将从大量简化中受益。希望有一些最小可行的子集，可以满足真正的使用者需求，具有良好的人机工程学，而无需等待
higher order impls 等推测性的功能。

找到正确的子集需要一些时间，但我认为 Rust 项目在以前的经历中关注于找到正确的一组功能，这是 Rust 今天成为伟大语言的重要原因！

我很高兴在这个领域有这么多有趣的问题要解决，我期待着异步 Rust 的 2023 年！

