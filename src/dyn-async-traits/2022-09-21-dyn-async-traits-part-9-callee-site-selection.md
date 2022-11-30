# Dyn async traits

[原文](https://smallcultfollowing.com/babysteps/blog/2022/09/21/dyn-async-traits-part-9-callee-site-selection/) |
日期：2022-09-21 17:35 -0400

在我 [上一篇] 关于 Dyn Async Trait 的帖子之后，一些人指出，我忽略了一个看似显而易见的可能性。

为什么不在调用处做出管理 Future 的选择？这是真的，我在很大程度上排除了这种选择，但它值得考虑。

这篇文章将探讨如何使基于调用处进行分发的工作方式，以及人体工程学可能是什么样子。我认为它实际上是相当有吸引力的，尽管它有一些局限性。

[上一篇]: ./2022-09-18-dyn-async-traits-part-8-the-soul-of-rust.md

## 如果添加对未知大小的返回值的支持

[RFC 2884]: https://github.com/rust-lang/rfcs/pull/2884

这个想法建立在 [RFC 2884] 提出的机制之上。根据该 RFC，你将能够拥有返回 `dyn Future` 的函数：

```rust,ignore
fn return_dyn() -> dyn Future<Output = ()> {
    async move { }
}
```

通常，当调用函数时，Rust 可以在栈上分配空间来存储返回值。但是当你调用 `return_dyn` 时，Rust 不知道在编译时需要多少空间，所以我们不能这样做。[^alloca]

这意味着你不能只编写 `let x = return_dyn()`。相反，你必须选择如何分配该内存。

使用 RFC 2884 提出的 API，最常见的选择是将其存储在堆上。`Box` 将增加一个新方法 `Box::new_with`，它的作用类似于 
`new`，但它接受闭包，闭包可以返回任何类型的值，包括 `dyn` 值：

```rust,ignore
let result = Box::new_with(|| return_dyn());
// result has type `Box<dyn Future<Output = ()>>`
```

调用 `new_with` 在人体工程学上会令人不快，因此我们还可以添加一个 `.box` 运算符。

Rust 一直有一个不稳定的 `box` 运算符，这次可能最终提供了足够的动力，使它值得添加：

```rust,ignore
let result = return_dyn().box;
// result has type `Box<dyn Future<Output = ()>>`
```

当然，你不必使用 `Box`。假设我们有足够的 API 可用，人们可以编写自己的方法，比如做一些基于区域的内存管理 (arena) 分配的事情...

```rust,ignore
let arena = Arena::new();
let result = arena.new_with(|| return_dyn());
```

...或者可能是一个假设的 `maybe_box`，如果足够大，它将使用缓冲区，否则使用 Box：

```rust,ignore
let mut big_buf = [0; 1024];
let result = maybe_box(&mut big_buf, || return_dyn()).await;
```

如果添加 [后缀宏][postfix macros]，那么甚至可能支持 `return_dyn.maybe_box!(&mut big_buf)` 之类的内容，尽管我不确定当前的提议是否支持这一点。

[postfix macros]: https://github.com/rust-lang/rfcs/pull/2442

[^alloca]: 我知道你现在会说：“但是栈分配 (alloca) 呢！”我会谈到它。

## 未知大小的返回值是什么？

这种返回 `dyn Future` 的想法有时被称为“未知大小的返回值” (unsized return values)，因为函数现在可以返回“未知大小”类型的值（即大小不是静态已知的类型）。

它们是由 [Olivier Faure] 在 [RFC 2884] 中提出的，我相信也有一些更早的RFC。

[Olivier Faure]: https://github.com/PoignardAzur

与此同时，`.box` 操作符几乎一直是 nightly Rust 的一部分，尽管它目前是以前缀的形式书写的，即 `box foo`[^prefix-box]。

未知大小返回值和 `.box` 的主要动机一直以来是效率：它们允许在今天不可能的情况下进行原地初始化。

例如，如果我今天写 `Box::new([0; 1024])`，从技术上讲，这是在栈上分配一个 `[0; 1024]` 大小的缓冲区，然后将其复制到 Box 中：

```rust,ignore
// First evaluate the argument, creating the temporary:
let temp: [u8; 1024] = ...;

// Then invoke `Box::new`, which allocates a Box...
let box: *const T = allocate_memory();

// ...and copies the memory in.
std::ptr::write(box, temp);
```

优化器也许能够解决这个问题，但这并不是一件小事。如果你查看操作的顺序，就会发现它需要在参数被分配之前进行分配。

LLVM 认为对已知分配器的调用是“无副作用的”，但提升它们仍然是有风险的，因为这意味着更早地分配更多的内存，这可能会导致内存耗尽。

重点不是看 LLVM 在实践中到底会做什么优化，而是说优化掉临时变量并不是一件小事：它需要一些深思熟虑的启发式方法。

[^prefix-box]: 当前没有办法稳定编译器支持的 `box foo` 运算符。有更早的计划（见 [RFC 809] 和 [RFC 1228]），但我们最终放弃了这些努力。
    事实上，这个问题的部分原因是 `box foo` 的顺序不符合人体工程学：`foo.box` 要好得多。

[RFC 809]: https://github.com/rust-lang/rfcs/pull/809
[RFC 1228]: https://rust-lang.github.io/rfcs/1228-placement-left-arrow.html

## 未知大小返回值将如何工作？

这值得一篇单独的博客文章，我不会深入探讨细节。

对于我们这里的目的，关键是，当被调用方返回其最终值时，它可以使用调用方喜欢的任何策略来得到达返回点，并将返回值直接写入其中。

RFC 2884 提出了一种基于生成器 (generators) 的解决方案，但我希望在我们确定某一种解决方案之前，花时间仔细考虑所有的替代方案。

## 在 traits 的异步函数中使用动态返回类型

因此，问题是，我们是否可以使用 `dyn` 返回类型来帮助实现 traits 中的异步函数？继续上一篇文章的例子，如果有一个 `AsyncIterator` trait...

```rust,ignore
trait AsyncIterator {
    type Item;
    
    async fn next(&mut self) -> Option<Self::Item>;
}
```

...我们想在 `dyn AsyncIterator` 类型上调用 `next` 后将产生 `dyn Future<Out = Option<Self::Item>>`。因此，你可以编写如下代码：

```rust,ignore
fn use_dyn(di: &mut dyn AsyncIterator) {
    di.next().box.await;
    //       ^^^^
}
```

表达式 `di.next()` 本身会产生一个 `dyn Future`。此类型没有大小，因此它不会自行编译。添加 `.box` 将生成一个 `Box<dyn AsyncIterator>`，然后你可以等待
(await) 它[^pin]。

[^pin]: 如果你今天尝试等待 `Box<dyn Future>`，那么[会得到一个错误][playground-pinned]，说它需要被固定。我认为可通过给 `Box<dyn Future>` 实现
    `IntoFuture` 并将其转换为 `Pin<Box<dyn Future>>` 来解决这个问题。

[playground-pinned]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=b981b7eafee70cc39f70176f6b135023

与我之前讨论的 Boxing 转换器相比，这相对容易解释。我不太确定在实践中使用哪一个更方便：这取决于你创建了多少个 `dyn`
值以及在它们上调用了多少个方法。当然，你可以通过包装器类型或辅助性方法为你解决在每个调用点编写 `.box` 的问题。

## 复杂性：`dyn AsyncIterator` 不实现 `AsyncIterator`

有一个复杂的问题。今天，在 Rust 中，每个 `dyn Trait` 类型也实现了 `Trait`。但是 `dyn AsyncIterator` 可以实现 `AsyncIterator`吗？

事实上，它不能！问题在于，`AsyncIterator` trait 将 `next` 定义为返回 `impl Future<...>`，实际上它是 `impl Future<...> + Sized` 的缩写，但我们说
`next` 将返回 `dyn Future<...>`，这是 `?Sized`。因此，`dyn AsyncIterator` 类型不符合 trait 所要求的约束。

## 但 `dyn AsyncIterator` 必须实现 `AsyncIterator` 吗？

并没有“硬性和固定”的理由让 `dyn Trait` 类型必须实现 `Trait`，而且有几个很好的理由不这样做。

dyn safety 的替代方案是这样的设计：你总是可以为任何 `Trait` 创建一个 `dyn Trait` 值，但你可能无法使用它所有的成员。

例如，给定一个 `dyn Iterator`，你可以调用 `next`，但不能调用像 `map` 这样的泛型方法。

事实上，我们在实践中已经实现了这种设计，这要归功于 `where Self: Sized` [黑客技巧][obj-safety-sized]，它允许我们排除某些方法，使这些方法不被用于 `dyn` 值。

[obj-safety-sized]: https://rust-lang.github.io/rfcs/0255-object-safety.html#adding-a-where-clause

首先为什么我们要采用 object safety？如果你回顾一下 [RFC 255]，这个规则的主要动机是人体工程学：更清晰的规则和更好的错误消息。

尽管我当时支持 RFC 255，但我现在认为这些动机并没有变得那么成熟。例如，现在如果你有一个带有泛型方法的 trait，当你尝试创建一个
`dyn Trait` 值时会得到一个错误，它告诉你不能从带有泛型方法的 trait 创建 `dyn Trait`。这可能比在调用泛型方法时得到一个错误，告诉你不能通过
`dyn Trait` 调用泛型方法会更清楚。

[RFC 255]: (https://rust-lang.github.io/rfcs/0255-object-safety.html

让 `dyn Trait` 实现 `Trait` 的另一个动机是，你可以用 `T: Trait` 编写一个泛型函数，并让它同样适用于对象类型。

这一功能很有用，但因为你必须写 `T: ?Sized` 来利用它，所以只有在你仔细计划的情况下，它才能真正发挥作用。

在实践中，我发现更有效的方法是让 `&dyn Trait` 实现 `Trait`。

## 删除 `dyn AsyncIterator: AsyncIterator` 意味着什么？

我认为新的系统应该是这样的：

* 你始终[^almost] 可以创建一个 `dyn Foo` 值。`dyn Foo` 类型将基于使用动态分发的 `Foo` trait 来定义固有方法，但有一些变化：
    * 异步函数和使用 `-> impl Trait` 定义的其他方法将返回 `-> dyn Trait`
    * 泛型方法、引用 `Self` 的方法以及其他此类情况被排除在外：这些不能通过虚拟分发处理
* 如果 `Foo` 使用当前规则是 [object safe]，则 `dyn Foo: foo` 成立
    * 在一个不太相关的补充，我想创建一个声明 dyn safety 所需的 `dyn` 关键字

[^MIR]: 在编译器内部，这将需要修改 MIR 的定义，使动态分发更加具有一流的支持

[^almost]: 还是几乎总是？我可能忽略了一些边缘案例。

[object safe]: https://doc.rust-lang.org/reference/items/traits.html#object-safety

## 删除上述这条规则的影响

这意味着 `dyn AsyncIterator` （或任何带有异步函数的 trait/RPITIT[^rpitit]）不会再实现 `AsyncIterator`。所以如果我写这个函数

```rust,ignore
fn use_any<I>(x: &mut I)
where
    I: ?Sized + AsyncIterator,
{
    x.next().await
}
```

那么我不能将其与 `I = dyn AsyncIterator` 一起使用。

原因不言而喻：它调用 `next` 并假设结果为 `Sized`（正如 trait 所承诺的那样），因此它不会添加任何类型的 `.box` 指令（它也不应该添加）。

你能做的是给封装 Box 的包装器实现 `AsyncIterator`：

```rust,ignore
struct BoxingAsyncIterator<'i, I> {
    iter: &'i mut dyn AsyncIterator<Item = I>
}

impl<I> AsyncIterator for BoxingAsyncIterator<'i, I> {
    type Item = I;
    
    async fn next(&mut self) -> Option<Self::Item> {
        self.iter.next().box.await
    }
}
```

然后你就可以调用 `use_any(BoxingAsyncIterator::new(ai))`[^boxing]。

[^rpitit]: 不知道 [RPITIT] (return position impl trait in traits) 代表什么？！它指的是在 traits 中函数的返回值位置上支持 impl trait 。

[RPITIT]: https://rust-lang.github.io/impl-trait-initiative/explainer/rpit.html

[^boxing]: 这基本上就是“神奇的” `Boxing::new` 在旧方案中为你做的事情。

## 限制：如果你想要进行栈分配怎么办？

[上一篇] 文章提议的一个目标是允许你编写使用 `dyn AsyncIterator` 的代码，该代码在 std 和 no-std 环境中同样有效。我觉得这个目标已经部分实现了。

核心想法是调用方选择分配 Future 的策略，因此它可以选择使用内联分配（因此支持 no-std）或使用 Box（因此很简单）。

在本文基于的这个提案中，调用处必须进行选择。然后，可能你会认为只能选择在调用处使用栈分配，从而支持 no-std。

但是如何选择堆分配呢？这实际上相当棘手！部分问题是，异步栈帧存储在结构体中，因此我们不能支持类似 `alloca` 的东西（至少不支持跨 await
存活的值，这包括正在 await 的任何 Future[^expl]）。

事实上，即使在异步之外，使用 alloca 也相当困难！问题在于，栈就是栈。

理想情况下，你应该在被调用方返回之前进行分配，但这是你知道需要多少内存的时候。而此时，被调用方仍在使用栈，因此你的分配位置错误。[^ada]

我个人认为，我们应该排除使用 alloca 进行栈分配的想法。

[^expl]: [这里][async-alloca] 简单解释一下为什么 async 和 alloca 不能在这里混合。

[async-alloca]: https://internals.rust-lang.org/t/blog-series-dyn-async-in-traits-continues/17403/52

[^ada]: 有人告诉我，Ada 编译器将在栈顶部配内存，将其复制到函数区的开始处，然后弹出剩余的内存。理论上是可能的！

如果我们不能使用 alloca，那还能做什么？我们有几个选择。

在一开始，我就谈到了 `maybe_box` 函数，它使用一个缓冲区并仅将其用于非常大的值的。这是一种巧妙的做法，但它仍然依赖于回退到 
Box，所以它对 no-std 没用[^abort]。不过， [stackfuture] 可能是的一个不错的选择[^size]。

[stackfuture]: https://twitter.com/theinedibleholk/status/1557802452069388288

[^abort]: 你可以想象这样一个版本，如果 size 错误，它也会中止 (abort) 代码，这将使它不安全，但不是以一种可靠的方式（中止令人讨厌）。

[^size]: 可以想象一下，你将大小设置为 `size_of(SomeOtherType)` 以自动确定需要多少空间。

你也可以通过编写包装器类型来实现内联（这是 Tmandry 和 我 [之前设计的原型][prototype]），但接下来的挑战是，被调用方不接受
`&mut dyn AsyncIterator`，它接受 `&mut DynAsyncIter` 之类的内容，其中 `DynAsyncIter` 是你定义的、用于包装的结构体。

[prototype]: https://github.com/nikomatsakis/dyner/blob/8086d4a16f68a2216ddff5c03c8c5b3d94ed93a2/src/dyn_async_iter.rs#L4-L6

**总而言之，我认为现实中的答案是：如果想让你的代码在 no-std 环境中被使用，就不要在你的公共接口中使用 `dyn`；只需使用
`impl AsyncIterator` 即可。如果你确实想要动态分发，则可以在内部使用包装器类型之类的奇淫技巧。**

## 问：编译器还有多大的空间可以变得更聪明？

我在考虑这项提议时的另一个担忧是，它似乎被夸大了。

也就是说，在该方案中，绝大多数调用处将使用 `.box`，因此意味着它们应该分配一个 Box 来存储结果。

但是像跨调用缓存 Box 或者“尽量”栈分配之类的想法又如何呢？它们适合放在哪里？

据我所知，这些优化仍然是可能的，只要将要分配的 Box 不脱离函数（这与我们之前的前提条件相同）。

这样想的话：使用者写 `foo().box.await`，告诉我们使用 boxing  分配器来把返回值 `foo` 放进 Box。但是我们可以看到，这个结果被传递给
await，这会取得所有权，然后释放它。因此，我们可以决定替换一个不同的分配器，也许是在调用之间重新使用 
Box，或者尝试使用栈内存；只要我们修改释放内存的代码来配合，这就是可以的。这样做依赖于要知道分配的值立即返回给我们，并且它永远不会离开我们的控制。

## 结论

总而言之，我认为对于大多数使用者来说，这个设计的工作原理是这样的：

* 可以将 `dyn` 用于具有异步函数的 traits，但每次调用方法时都必须写 `.box`
* 在其他地方也可以使用 `.box`，我们至少获得了某些对未知大小返回值的支持[^limit]
* 如果你想编写有时使用 `dyn`，有时使用静态分发的代码，则必须编写一些笨拙的包装器类型[^fornow]
* 如果你正在编写 no-std 代码，请使用 `impl Trait`，而不是 `dyn Trait`；如果你必须使用 `dyn`，则将需要包装器类型

最初，我反对调用处分配，因为它违反了 `dyn Trait: Trait`，并且它不允许使用同时支持 std 和 no-std 的 `dyn` 所编写的代码。

但我现在认为违反 `dyn Trait: Trait` 实际上可能是好的，我不确定后一个约束到底有多重要。

此外，我认为 `Boxing::new` 和各种 “dyn 转换器” 可能会让使用者相当困惑，但在调用处上编写 `.box` 相对容易解释（“我们不知道你需要什么
Future，所以你必须将放入 Box”）。

所以现在它对我来说似乎更有吸引力了，我很感谢 [Olivier Faure] 再次提出这个问题。

[^limit]: 我说了“至少某些”，因为我怀疑，在获得更多经验之前，更一般情况的许多细节将仍不稳定。

[^fornow]: 无论如何，现在你必须编写笨拙的包装器类型。我对如何让它更自动化的想法很感兴趣，但我认为这超出了本文的范围。

一种可能的扩展是允许使用者以某种方式指定每个返回的 Future 的类型。

当我写完这篇文章时，我看到 Matthieum 在内部帖子上贴出了[一个有趣的想法][matthieum]。

总的来说，我确实看到了需要某种 “trait 转换器” (adapters)，这样你就可以获取一个像 `Iterator` 这样的 base 
trait，并以各种方式“转换”它，例如生成一个使用异步方法的版本，或者是 const-safe 的版本。

这与整个 [关键字泛型][keyword generics] (keyword generics) 计划也有一些相当大的重叠。

我认为这是一个值得考虑的扩展，但它不会是我们最先发布的 MVP 的一部分。

[matthieum]: https://internals.rust-lang.org/t/blog-series-dyn-async-in-traits-continues/17403/50
[keyword generics]: https://blog.yoshuawuyts.com/announcing-the-keyword-generics-initiative/

## 此处留言

请在 [这个帖子][post] 里留言，谢谢！

[post]: https://internals.rust-lang.org/t/blog-series-dyn-async-in-traits-continues/17403/40

## 附录：`FnOnce` 的关联类型 `Output`

[`Output`]: https://doc.rust-lang.org/std/ops/trait.FnOnce.html#associatedtype.Output

这是一件有趣的事情！由所有可调用事物实现的 `FnOnce` trait 将其关联类型 [`Output`] 定义为 `Sized`！如果我们想允许未知大小的返回值，就必须改变这一点。

从理论上讲，这可能是一个巨大的向前兼容的风险。根据 `FnOnce` trait，编写 `F::Output` 的代码可以假定返回值是 Sized，所以如果我们去掉这个约束，代码将不再编译！

幸运的是，我认为这是可以的。我们故意限制了 fn 类型，所以你只能使用 `()` 表示法，例如 `where F: FnOnce()` 或 `where F: FnOnce() -> ()`。

这两种形式都扩展为显式指定 `Output` 的内容，如 `F: FnOnce<(), Output = ()>`。这意味着即使你真的编写泛型代码...

```rust,ignore
fn foo<F, R>(f: F)
where
    F: FnOnce<Output = R>
{
    let value: F::Output = f();
    ...
}
```

...当你写 `F::Output` 时，它实际上被化为 `R`，类型 `R` 有自己的（隐式的） `Sized` 约束。

（实际上最近有一个与这个约束相关的不健全问题，[被这个 PR 关闭了][unsoundness]，我们在 Zulip 上 [讨论了][zulip] 这个向后兼容的问题。）

[unsoundness]: https://github.com/rust-lang/rust/pull/100096
[zulip]: https://rust-lang.zulipchat.com/#narrow/stream/326866-t-types.2Fnominated/topic/.23100096.3A.20a.20fn.20pointer.20doesn't.20implement.20.60Fn.60.2F.60FnMut.60.2F.60FnOnc.E2.80.A6/near/297797248


