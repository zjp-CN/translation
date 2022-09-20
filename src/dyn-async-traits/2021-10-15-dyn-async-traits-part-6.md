---

## layout: post
title: Dyn async traits, part 6
date: 2021-10-15 15:57 -0400

## 版面：帖子标题：Dyn Async特征，第六部分日期：2021-10-15 15：57-0400

A quick update to my last post: first, a better way to do what I was trying to do, and second, a sketch of the crate I'd like to see for experimental purposes.

对我上一篇帖子的快速更新：第一，一种更好的方法来做我想做的事情，第二，我想要看到的用于实验目的的板条箱的草图。

## An easier way to roll our own boxed dyn traits

## 一种更简单的方式来描述我们自己的盒装Dyn特征

In the previous post I covered how you could create vtables and pair the up with a data pointer to kind of "roll your own dyn". After I published the post, though, dtolnay sent me [this Rust playground link](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=adba43d6e056337cd8a297624a296219) to show me a much better approach, one based on the [erased-serde](https://crates.io/crates/erased-serde) crate. The idea is that instead of make a "vtable struct" with a bunch of fn pointers, we create a "shadow trait" that reflects the contents of that vtable:

在上一篇文章中，我介绍了如何创建vtable并将其与数据指针配对，以实现某种“自己的动态滚动”。不过，在我发表了这篇文章后，dtolnay给我发来了这个铁锈游乐场的链接，向我展示了一种更好的方法，一种基于擦除-serde板条箱的方法。我们的想法是，不是用一堆fn指针创建“vtable struct”，而是创建一个反映该vtable内容的“影子特征”：

```rust
// erased trait:
trait ErasedAsyncIter {
    type Item;
    fn next<'me>(&'me mut self) -> Pin<Box<dyn Future<Output = Option<Self::Item>> + 'me>>;
}
```

Then the `DynAsyncIter` struct can just be a boxed form of this trait:

那么，`dyAsyncIter`结构就可以是该特征的盒装形式：

```rust
pub struct DynAsyncIter<'data, Item> {
    pointer: Box<dyn ErasedAsyncIter<Item = Item> + 'data>,
}
```

We define the "shim functions" by implementing `ErasedAsyncIter` for all `T: AsyncIter`:

我们通过为所有`t：AsyncIter`实现`ErasedAsyncIter`来定义填充函数：

```rust
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

And finally we can implement the `AsyncIter` trait for the dynamic type:

最后，我们可以为动态类型实现`AsyncIter`特征：

```rust
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

Yay, it all works, and without *any* unsafe code!

耶，一切正常，而且没有任何不安全的代码！

## What I'd like to see

## 我想看到的是

This "convert to dyn" approach isn't really specific to async (as erased-serde shows). I'd like to see a decorator that applies it to any trait. I imagine something like:

这种“转换为dyn”的方法并不是真正特定于异步的(如erasted-serde所示)。我想看到一位能把它运用到任何特征上的装饰师。我的想象是这样的：

```rust
// Generates the `DynAsyncIter` type shown above:
#[derive_dyn(DynAsyncIter)]
trait AsyncIter {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}
```

But this ought to work with any `-> impl Trait` return type, too, so long as `Trait` is dyn safe and implemented for `Box<T>`. So something like this:

但这应该也适用于任何`->ImplTrait`返回类型，只要`Trait`是dyn安全的，并为`Box<T>`实现。所以大概是这样的：

```rust
// Generates the `DynAsyncIter` type shown above:
#[derive_dyn(DynSillyIterTools)]
trait SillyIterTools: Iterator {
    // Iterate over the iter in pairs of two items.
    fn pair_up(&mut self) -> impl Iterator<(Self::Item, Self::Item)>;
}
```

would generate an erased trait that returns a `Box<dyn Iterator<(...)>>`. Similarly, you could do a trick with taking any `impl Foo` and passing in a `Box<dyn Foo>`, so you can support impl Trait in argument position.

将生成一个擦除的特征，该特征返回一个`Box<dyn Iterator<(...)>`。类似地，您可以使用任何`impl Foo`并传递一个`Box<dyn Foo>`来实现一个技巧，这样您就可以在参数位置支持Iml特征。

Even without impl trait, `derive_dyn` would create a more ergonomic dyn to play with.

即使没有Iml特征，`duced_dyn‘也会创建一个更符合人体工程学的dyn来玩。

I don't really see this as a "long term solution", but I would be interested to play with it.

我真的不认为这是一个“长期解决方案”，但我会有兴趣玩它。

## Comments?

## 有什么评论吗？

I've created a [thread on internals](https://internals.rust-lang.org/t/blog-series-dyn-async-in-traits/15449) if you'd like to comment on this post, or others in this series.

如果你想对这篇文章或本系列的其他文章发表评论，我已经创建了一个关于内部结构的帖子。