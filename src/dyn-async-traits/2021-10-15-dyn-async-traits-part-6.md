# Dyn async traits

[原文](https://smallcultfollowing.com/babysteps/blog/2021/10/15/dyn-async-traits-part-6/) |
日期：2021-10-15 15:57 -0400

对我上一篇帖子的快速更新：

1. 一种更好的方法来做我想做的事情
2. 我想要看到的用于实验目的的库的草图

## 一种更简单的方式来编写 boxed dyn traits

在上一篇文章中，我介绍了如何创建 vtable 并将其与数据指针配对，以实现某种“自定义的动态分发”。

不过，在我发表了这篇文章后，dtolnay 给我发来了这个 playground [链接][playground]，向我展示了一种更好的方法，一种基于 [erased-serde] 库的方法。

主要的想法是，不是用一堆函数指针创建 vtable 结构体，而是创建一个反映该 vtable 内容的“shadow trait”：

[playground]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=adba43d6e056337cd8a297624a296219
[erased-serde]: https://crates.io/crates/erased-serde

```rust,ignore
// erased trait:
trait ErasedAsyncIter {
    type Item;
    fn next<'me>(&'me mut self) -> Pin<Box<dyn Future<Output = Option<Self::Item>> + 'me>>;
}
```

那么 `DynAsyncIter` 就可以是该 trait 的 boxed 形式：

```rust,ignore
pub struct DynAsyncIter<'data, Item> {
    pointer: Box<dyn ErasedAsyncIter<Item = Item> + 'data>,
}
```

通过给所有 `T: AsyncIter` 实现 `ErasedAsyncIter` 来定义填充函数 (shim functions)：

```rust,ignore
impl<T> ErasedAsyncIter for T
where
    T: AsyncIter,
{
    type Item = T::Item;
    fn next<'me>(&'me mut self) -> Pin<Box<dyn Future<Output = Option<Self::Item>> + 'me>> {
        // This code allocates a box for the result
        // and coerces into a dyn:
        Box::pin(AsyncIter::next(self))
    }
}
```

最后，为动态类型实现 `AsyncIter`：

```rust,ignore
impl<'data, Item> AsyncIter for DynAsyncIter<'data, Item> {
    type Item = Item;

    type Next<'me>
    where
        Item: 'me,
        'data: 'me,
    = Pin<Box<dyn Future<Output = Option<Item>> + 'me>>;

    fn next(&mut self) -> Self::Next<'_> {
        self.pointer.next()
    }
}
```

耶，一切正常，而且没有任何不安全的代码！

## 我想看到的是

这种“转换为 dyn”的方法并不是特定于异步的（如 erased-serde 所展示那样）。我想看到一个能把它运用到任何 trait 上的装饰器。我的想象是这样的：

```rust,ignore
// Generates the `DynAsyncIter` type shown above:
#[derive_dyn(DynAsyncIter)]
trait AsyncIter {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}
```

但这应该也适用于任何 `-> ImplTrait` 的返回类型，只要 `Trait` 是 dyn safe，并给 `Box<T>` 实现 `Trait`。所以大概是这样的：

```rust,ignore
// Generates the `DynAsyncIter` type shown above:
#[derive_dyn(DynSillyIterTools)]
trait SillyIterTools: Iterator {
    // Iterate over the iter in pairs of two items.
    fn pair_up(&mut self) -> impl Iterator<(Self::Item, Self::Item)>;
}
```

这将生成一个擦除的 trait，该 trait 返回一个 `Box<dyn Iterator<(...)>`。

类似地，你可以使用任何 `impl Foo` 并传递一个 `Box<dyn Foo>` 来实现一个技巧，这样就可以在参数位置上支持 impl Trait。

即使没有 impl Trait，`derive_dyn` 也会创建一个更符合人体工程学的 dyn。

我不认为这是一个“长期解决方案”，但我会有兴趣试试它。

## 有什么评论吗？

如果你想对这篇文章或本系列的其他文章发表评论，我已经在内部论坛上创建了一个 [帖子][post]。

[post]: https://internals.rust-lang.org/t/blog-series-dyn-async-in-traits/15449

