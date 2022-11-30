# Dyn Async Traits

[原文](https://smallcultfollowing.com/babysteps/blog/2021/09/30/dyn-async-traits-part-1/) |
日期：2021-09-30 10:50 -0400

在过去的几周里，[Tyler Mandry] 和我一直在努力研究如何在 trait 中实现异步函数。

根据新的 Rust Lang 团队[提案流程][initiative]，我们将在一个不断更新的网站 [Async Fundamentals Initiative]
中收集设计思想。如果你对这一领域感兴趣，你绝对应该四处看看；你可能会有兴趣阅读 [路线图][roadmap]，或者方案[评估][evaluation]，其中涵盖了仍在解决的一些挑战。

我将写一系列博客文章，重点关注我们一直在讨论的一件事：[`dyn` 和 `async fn` 的问题][`dyn` & `async fn`]。

第一个帖子介绍了问题和我们正在努力的总体目标（但还不知道最好的方法）。

[Tyler Mandry]: https://github.com/tmandry/
[initiative]: https://lang-team.rust-lang.org/initiatives.html
[Async Fundamentals Initiative]: https://rust-lang.github.io/async-fundamentals-initiative/
[roadmap]: https://rust-lang.github.io/async-fundamentals-initiative/roadmap.html
[evaluation]: https://rust-lang.github.io/async-fundamentals-initiative/evaluation.html
[`dyn` & `async fn`]: https://rust-lang.github.io/async-fundamentals-initiative/evaluation/challenges/dyn_traits.html

## 目的

我们想要的很简单。想象一下这个表示“异步迭代器”的 trait：

```rust,ignore
trait AsyncIter {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}
```

我们希望你能够编写这样的 trait，并以显而易见的方式实现它：

```rust,ignore
struct SleepyRange {
    start: u32,
    stop: u32,
}

impl AsyncIter for SleepyRange {
    type Item = u32;
    
    async fn next(&mut self) -> Option<Self::Item> {
        tokio::sleep(1000).await; // just to await something :)
        let s = self.start;
        if s < self.stop {
            self.start = s + 1;
            Some(s)
        } else {
            None
        }
    }
}
```

然后，你应该能够拥有 `Box<dyn AsyncIter<Item = u32>>`，并以与使用 `Box<dyn Iterator<Item = u32>>`
完全相同的方式使用它（当然，在每次调用 `next` 之后都带有一个`wait`）：

```rust,ignore
let b: Box<dyn AsyncIter<Item = u32>> = ...;
let i = b.next().await;
```

## 解语法糖为关联类型

考虑下面的示例：

```rust,ignore
trait AsyncIter {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}
```

这里的 `next` 方法将解糖为返回某种 Future 的函数；你可以将其视为泛型关联类型：

```rust,ignore
trait AsyncIter {
    type Item;

    type Next<'me>: Future<Output = Self::Item> + 'me;
    fn next(&mut self) -> Self::Next<'_>;
}
```

针对 impl 的解糖将使用 [type alias impl trait] (tait)：

[type alias impl trait]: https://rust-lang.github.io/impl-trait-initiative/

```rust,ignore
struct SleepyRange {
    start: u32,
    stop: u32,
}

// Type alias impl trait:
type SleepyRangeNext<'me> = impl Future<Output = u32> + 'me;

impl AsyncIter for InfinityAndBeyond {
    type Item = u32;
    
    type Next<'me> = SleepyRangeNext<'me>;
    fn next(&mut self) -> SleepyRangeNext<'me> {
        async move {
            tokio::sleep(1000).await;
            let s = self.start;
            ... // as above
        }
    }
}
```

这种解糖对于标准的泛型（或 `impl Trait`）非常有效。考虑以下函数：

```rust,ignore
async fn process<T>(t: &mut T) -> u32
where
    T: AsyncIter<Item = u32>,
{
    let mut sum = 0;
    while let Some(x) = t.next().await {
        sum += x;
        if sum > 22 {
            break;
        }
    }
    sum
}
```

这段代码将非常好地工作。例如，当你调用 `t.next()` 时，得到的 Future 将是 `T::Next` 类型。

单态化后，编译器可以将 `<SleepyRange as AsyncIter>::Next` 解析为` SleepyRangeNext` 类型，这样就可以准确地知道 future。

事实上，像 [embassy] 这样的库已经在使用这种解糖，尽管是手动的，而且只在 nightly 上使用。

[embassy]: https://github.com/akiles/embassy

## 关联类型不适用于 `dyn`

不幸的是，当你尝试使用 `dyn` 值时，这种解糖会导致问题。今天的 `dyn AsyncIter` 必须指定 `AsyncIter`
中定义的所有关联类型的值。因此，这意味着你必须编写如下内容，而不是 `dyn AsyncIter`

```rust,ignore
for<'me> dyn AsyncIter<
    Item = u32, 
    Next<'me> = SleepyRangeNext<'me>,
>
```

从人体工程学的角度来看，这显然是不可能的，但它还有一个更有害的问题。`dyn` trait 的全部意义在于，在不知道底层类型是什么的情况下有一个值。

但是将 `Next<'me>` 的值指定为 `SleepyRangeNext` 意味着这里正好有一个 impl 可以使用。这个 `dyn` 值必须是 `SleepyRange`，因为没有其他 impl 有相同的 future。

结论：要让 `dyn AsyncIter` 工作，`next()` 返回的 future 必须独立于实际的 impl。此外，它必须有固定的大小。换句话说，它应类似于 `Box<dyn Future<Output=u32>>`。

## `async-trait` 库如何解决这一问题

[`async-trait`]: https://crates.io/crates/async-trait

你可能已经使用了 [`async-trait`] 库来解决这个问题，不使用关联的类型，而是解糖到 `Box<dyn Future>` 类型：

```rust,ignore
trait AsyncIter {
    type Item;

    fn next(&mut self) -> Box<dyn Future<Output = Self::Item> + Send + 'me>;
}
```

这有几个缺点：

* 它一直强制使用 `Box`，即使在使用 `AsyncIter` 进行静态分发时也是如此。
* 上面给出的类型表示返回的 future 必须是 `Send`。对于其他异步函数，使用 auto trait 来自动分析 future 是否为 Send（如果可以的话，它是 
  `Send`，换句话说，我们不预先声明它是否必须是）。

## 结论

> 理想情况下，当使用 `dyn` 时，我们希望使用 `Box`，而其他情况下不使用 `Box`。

到目前为止，我们已经看到：

* 如果我们将异步函数解糖为关联类型，那么它对泛型情况很有效，因为可以精确地将 future 解析为正确的类型。
* 但它对 `dyn` trait 不适用，因为 Rust 的规则要求我们准确地指定关联类型的值。对于 `dyn` trait，我们非常希望返回的 future 类似于 `Box<dyn Future>`
  * 相对于静态分发，使用 `Box` 确实会有轻微的性能损失，因为我们必须动态分配 future。

理想情况下，我们希望使用 `dyn` 时只支付 `Box` 的代价：

* 当你在泛型类型中使用 `AsyncIter` 时，会得到上面所示的解糖，没有放入 Box 和静态分派
* 但是当你创建一个 `dyn AsyncIter` 时，future 的类型变成 `Box<dyn Future<Output=u32>>`
  * （也许你可以选择 `Box` 之外的另一个“智能指针”类型，但我现在会忽略它，稍后再来讨论它。）在接下来的帖子中，我将深入探讨我们可能实现这一点的一些方法。
