# Dyn Async Traits

[原文](https://smallcultfollowing.com/babysteps/blog/2021/10/07/dyn-async-traits-part-4/) |
日期：2021-10-07 12:33 -0400

在上一篇文章中，我谈到了如何通过添加一些原语来编写自己的 `impl Iterator for dyn Iterator`。

在这篇文章中，我想看看如何将其扩展到异步迭代器。和以前一样，我对探索一切正常运行所需的“核心能力”很感兴趣。

## 假设我们想要 Box

在 [本系列的第一篇文章][post1] 中，我们讨论了如何通过 `dyn Trait` 调用异步函数，使该异步函数的返回类型为
`Box<dyn Future>`，但仅当通过 dyn 类型调用它时这样做，而不要一直如此。

[post1]: ./2021-09-30-dyn-async-traits-part-1.html#结论

实际上，这稍微简化了一下： `Box<dyn Future>` 当然是我们可以使用的一种类型，但你可能还需要其他类型：

* `Box<dyn Future + Send>` 表示 Future 可以跨线程发送
* 除了 `Box` 之外，还有其他一些包装类型

为了简单起见，本文中出现 `Box<dyn Future>`。稍后将回到这些扩展类型。

## 背景：`AsyncIter` 示例

首先回顾一下 `AsyncIter` trait：

```rust,ignore
trait AsyncIter {
    type Item;

    async fn next(&mut self) -> Option<Self::Item>;
}
```

请记住，当我们对这个 `async fn` 语法去糖时，我们为 `next` 返回的 Future 引入了一个新的（泛型）关联类型，这里称为 `Next`：

```rust,ignore
trait AsyncIter {
    type Item;

    type Next<'me>: Future<Output = Self::Item> + 'me;
    fn next(&mut self) -> Self::Next<'_>;
}
```

我们正在使用一个实现了 `AsyncIter` trait 的 `SleepyRange` 结构体：

```rust,ignore
struct SleepyRange { ... }
impl AsyncIter for SleepyRange {
    type Item = u32;
    ...
}
```

## 背景：静态与动态上下文中的关联类型

在静态上下文中使用关联类型很好，因为这意味着当调用 `sleepy_range.next()` 时，我们能够精确地解析返回的 Future
类型。这有助于我们准确地分配所需数量的栈，等等。

但在动态上下文中，例如，如果对 `some_iter: Box<dyn AsyncIter>`，调用 `some_iter.next()`，是一种负担。

使用 `dyn` 的关键在于我们不知道调用的是 `AsyncIter::next` 的哪个实现，所以无法确切知道返回什么 Future 类型。

我们实际上只想得到一个类似于 `Box<dyn Future<Output = Option<u32>>` 的东西。

## 如何让 trait 只在使用 dyn 的时候才把 Futures 放入 `Box`

如果我们希望在使用 `dyn` 时，这个 trait 只包含 Future ，则需要做两件事。

首先，修改 `impl AsyncIter for dyn AsyncIter`。 在今天的编译器中，它生成一个 impl ，对每个关联类型的值进行泛化。

但我们希望 impl 对 `Item` 类型的值进行泛化，但把 `Next` 类型的值指定为 `Box<dyn Future>`。

如此一来，实际上就是“当你对一个 `dyn AsyncIter` 调用 `next` 方法时，你总是得到 `Box<dyn Future>`”。

（但当你对一个特定类型调用 `next` 方法时，比如 `SleepyRange`，你会得到一个不同的类型 —— 真实的 Future 类型，而不是 boxed 的版本）。

如果我们用 Rust 代码编写 dyn Impl，它可能如下所示：

```rust,ignore
impl<I> AsyncIter for dyn AsyncIter<Item = I> {
    type Item = I;

    type Next<'me> = Box<dyn Future<Output = Option<I>> + 'me>;

    fn next(&mut self) -> Self::Next<'_> {
        /* see below */
    }
}
```

`next` 函数体是从 vtable 中提取函数指针来调用。背后依赖于 [RFC 2580] 的 APIs 以及我在上一篇文章中描述的函数 `associated_fn`，就像这样：

[RFC 2580]: https://rust-lang.github.io/rfcs/2580-ptr-meta.html

```rust,ignore
fn next(&mut self) -> Self::Next<'_> {
    type RuntimeType = ();
    let data_pointer: *mut RuntimeType = self as *mut ();
    let vtable: DynMetadata = ptr::metadata(self);
    let fn_pointer: fn(*mut RuntimeType) -> Box<dyn Future<Output = Option<I>> + '_> =
        associated_fn::<AsyncIter::next>();
    fn_pointer(data)
}
```

这仍然是我们想要的代码，但略有不同。

## 构造 vtable：异步函数需要填充才能返回 `Box`

在上面的`next`方法中，我们从 vtable 提取的函数指针类型如下：

```rust,ignore
fn(*mut RuntimeType) -> Box<dyn Future<Output = Option<I>> + '_>
```

但是， impl 中的函数签名不同！它不返回 `Box` 而是返回一个 `impl Future`！我们必须设法弥合这一鸿沟。我们需要的是一种“填充功能” (shim)，就像这样：

```rust,ignore
fn next_box_shim<T: AsyncIter>(this: &mut T) -> Box<dyn Future<Output = Option<I>> + '_> {
    let future: impl Future<Output = Option<I>> = AsyncIter::next(this);
    Box::new(future)
}
```

现在 `SleepyRange` 的 vtable 可以存储 `next_box_shim::<SleepyRange>`，而不是直接存储 `<SleepyRange as AsyncIter>::next`。

## 扩展 `AssociatedFn` trait

在我之前的帖子中，我勾勒出了一个 `AssociatedFn` trait 的概念，它具有一个关联的类型 `FnPtr`。

如果我们想使这种填充构造自动化，则需要把其关联类型改为它自己的特征。例如这样：

```rust,ignore
trait AssociatedFn { }
trait Reify<F>: AssociatedFn {
    fn reify(self) -> F; 
}
```

其中，`A: Reify<F>` 表示关联函数 `A` 可以对函数类型 `F` 进行实例化（变成函数指针）。

编译器可以在可能的情况下直接实现这一 trait，也可以为各种填充和 ABI 转换实现这一特性。

例如，`AsyncIter::next` 方法可以实现 `Reify<fn(*mut ()) -> Box<dyn Future<...>>`，以允许构造“boxing shim”等等。

## 其他种类的填充

关于 dyn Trait，还有其他各种限制，可通过明智地使用填充和微调 vtable 来克服，至少在某些情况下是这样。举个例子，考虑以下 trait：

```rust,ignore
pub trait Append {
    fn append(&mut self, values: impl Iterator<Item = u32>);
}
```

这个 trait 在传统意义上不是 dyn-safe 的，因为 `append` 函数是泛型的，并且需要对每种迭代器进行单态化。

因为我们还不知道它将应用于什么类型的迭代器，所以不知道应该将哪个版本放入 `Append` 的 vtable 中！

但是如果我们只放一个版本，迭代器类型是 `&mut dyn Iterator<Item = u32>` 会怎么样？

然后，调整 `impl Append for dyn Append` 来创建这个 `&mut dyn Iterator`，并从 vtable 调用函数：

```rust,ignore
impl Append for dyn Append {
    fn append(&mut self, values: impl Iterator<Item = u32>) {
        let values_dyn: &mut dyn Iterator<Item = u32> = &values;
        type RuntimeType = ();
        let data_pointer: *mut RuntimeType = self as *mut ();
        let vtable: DynMetadata = ptr::metadata(self);
        let f = associated_fn::<Append::append>(vtable);
        f(data_pointer, values_dyn);
    }
}
```

## 结论

dyn async Trait 的核心组成部分似乎是：

* 能够为了某个 trait 生成 vtable，自定义 vtable 的内容
  * 例如，异步函数需要填充函数来把返回值放入 `Box`
* 能够自定义派发的逻辑 （`impl foo for dyn Foo`）
* 能够将对 `Next` 这样的关联类型自定义为 `Box<dyn>`：
  * 这需要提取 vtable，正如 [RFC 2580] 描述的那样
  * 还需要从 vtable 中提取函数（目前不支持）

我在一开始就说过，为了简单起见，将假定我们想要返回 `Box<dyn>`，现在已经做到了。

似乎可以将这些核心功能扩展到其他类型的返回类型（如其他智能指针），但这并不够；我们必须定义编译器可以给哪些类型生成填充。

虽然我勾勒出了一些可能性，但我并没有认真考虑过如何允许使用者指定每一个构建块。

在这一点上，我主要是尝试探索公开哪些类型的功能可能是有用的或必要的。
