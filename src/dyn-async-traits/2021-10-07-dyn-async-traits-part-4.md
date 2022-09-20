---

## layout: post
title: Dyn async traits, part 4
date: 2021-10-07 12:33 -0400

## 版面：帖子标题：Dyn Async特征，第四部分日期：2021-10-07 12：33-0400

In the previous post, I talked about how we could write our own `impl Iterator for dyn Iterator` by adding a few primitives. In this post, I want to look at what it would take to extend that to an async iterator trait. As before, I am interested in exploring the “core capabilities” that would be needed to make everything work.

在上一篇文章中，我谈到了如何通过添加一些原语来编写我们自己的‘为dyn迭代器执行迭代器’。在这篇文章中，我想看看如何将其扩展到异步迭代器特性。和以前一样，我对探索一切正常运行所需的“核心能力”很感兴趣。

## Start somewhere: Just assume we want Box

## 从某个地方开始：假设我们想要Box

In the \[first post of this series\]\[post1\], we talked about how invoking an async fn through a dyn trait should to have the return type of that async fn be a `Box<dyn Future>` — but only when calling it through a dyn type, not all the time.

在[本系列的第一篇文章][post1]中，我们讨论了如何通过dyn特征调用异步fn，以使该异步fn的返回类型为`Box<dyn Future>`--但仅当通过dyn类型调用它时，而不是一直如此。

\[post1\]: {{ site.baseurl }}/blog/2021/09/30/dyn-async-traits-part-1/#conclusion-ideally-we-want-box-when-using-dyn-but-not-otherwise

\[帖子1]：{{site.base url}}/blog/2021/09/30/dyn-async-traits-part-1/#conclusion-ideally-we-want-box-when-using-dyn-but-not-otherwise

Actually, that’s a slight simplification: `Box<dyn Future>` is certainly one type we could use, but there are other types you might want:

实际上，这稍微简化了一点：`Box<dyn Future>`当然是我们可以使用的一种类型，但您可能还需要其他类型：

* `Box<dyn Future + Send>`, to indicate that the future is sendable across threads;
* Some other wrapper type besides `Box`.

To keep things simple, I’m just going to look at `Box<dyn Future>` in this post. We’ll come back to some of those extensions later.

\`Box<dyn Future+Send>`，表示未来可以跨线程发送；除了`Box`之外，还有其他一些包装类型。为了简单起见，我只在本文中查看`Box<dyn Future>`。我们稍后将回到其中的一些扩展。

## Background: Running example

## 背景：运行示例

Let’s start by recalling the `AsyncIter` trait:

让我们首先回顾一下`AsyncIter`特性：

```rust
trait AsyncIter {
    type Item;

    async fn next(&mut self) -> Option<Self::Item>;
}
```

Remember that when we “desugared” this `async fn`, we introduced a new (generic) associated type for the future returned by `next`, called `Next` here:

请记住，当我们对这个`async fn`进行去糖化时，我们为`next`返回的未来引入了一个新的(泛型)关联类型，这里称为`next`：

```rust
trait AsyncIter {
    type Item;

    type Next<'me>: Future<Output = Self::Item> + 'me;
    fn next(&mut self) -> Self::Next<'_>;
}
```

We were working with a struct `SleepyRange` that implements `AsyncIter`:

我们正在使用一个实现`AsyncIter`的结构`SleepyRange`：

```rust
struct SleepyRange { … }
impl AsyncIter for SleepyRange {
    type Item = u32;
    …
}
```

## Background: Associated types in a static vs dyn context

## 背景：静态与动态上下文中的关联类型

Using an associated type is great in a static context, because it means that when you call `sleepy_range.next()`, we are able to resolve the returned future type precisely. This helps us to allocate exactly as much stack as is needed and so forth.

在静态上下文中使用关联类型很好，因为这意味着当您调用`sleepy_range.next()`时，我们能够精确地解析返回的Future类型。这有助于我们准确地分配所需数量的堆栈，依此类推。

But in a dynamic context, i.e. if you have `some_iter: Box<dyn AsyncIter>` and you invoke `some_iter.next()`, that’s a liability. The whole point of using `dyn` is that we don’t know exactly what implementation of `AsyncIter::next` we are invoking, so we can’t know exactly what future type is returned. Really, we just want to get back a `Box<dyn Future<Output = Option<u32>>>` — or something very similar.

但在动态上下文中，例如，如果您有`Some_ITER：box<dyn AsyncIter>`，并且您调用`Some_iter.Next()`，则这是一种负担。使用`dyn`的关键在于我们不知道我们调用的是`AsyncIter：：next`的哪个实现，所以我们不能确切知道将来返回什么类型。真的，我们只是想得到一个`Box<dyn Future<Output=Option<u32>>`--或者非常类似的东西。

## How could we have a trait that boxes futures, but only when using dyn?

## 我们怎么可能有一个特征，只有在使用Dyn的时候才能包装期货呢？

If we want the trait to only box futures when using `dyn`, there are two things we need.

如果我们希望在使用‘dyn’时，这个特征只包含期货，我们需要两件事。

**First, we need to change the `impl AsyncIter for dyn AsyncIter`.** In the compiler today, it generates an impl which is generic over the value of every associated type. But we want an impl that is generic over the value of the `Item` type, but which *specifies* the value of the `Next` type to be `Box<dyn Future>`. This way, we are effectively saying that “when you call the `next` method on a `dyn AsyncIter`, you always get a `Box<dyn Future>` back” (but when you call the `next` method on a specific type, such as a `SleepyRange`, you would get back a different type — the actual future type, not a boxed version). If we were to write that dyn impl in Rust code, it might look something like this:

首先，我们需要为dyn AsyncIter`修改`impl AsyncIter`。在今天的编译器中，它生成一个在每个关联类型的值上泛型的impl。但我们希望Iml是`Item`类型的值的泛型，但它将`Next`类型的值指定为`Box<dyn Future>`。如此一来，我们实际上就是在说“当你对一个`dyn AsyncIter`调用`next`方法时，你总是得到一个`Box<dyn Future>`back”(但当你对一个特定类型调用`next`方法时，比如`SleepyRange`，你会得到一个不同的类型--实际的未来类型，而不是盒装的版本)。如果我们用Rust代码编写dyn Impll，它可能如下所示：

```rust
impl<I> AsyncIter for dyn AsyncIter<Item = I> {
    type Item = I;

    type Next<'me> = Box<dyn Future<Output = Option<I>> + ‘me>;
    fn next(&mut self) -> Self::Next<'_> {
        /* see below */
    }
}
```

The body of the `next` function is code that extracts the function pointer from the vtable and calls it. Something like this, relying on the APIs from \[RFC 2580\] along with the function `associated_fn` that I sketched in the previous post:

\`next`函数体是从vtable中提取函数指针并调用它的代码。依赖于来自[RFC 2580]的API以及我在上一篇文章中概述的函数`Associated_fn`，就像这样：

```rust
fn next(&mut self) -> Self::Next<‘_> {
    type RuntimeType = ();
    let data_pointer: *mut RuntimeType = self as *mut ();
    let vtable: DynMetadata = ptr::metadata(self);
    let fn_pointer: fn(*mut RuntimeType) -> Box<dyn Future<Output = Option<I>> + ‘_> =
        associated_fn::<AsyncIter::next>();
    fn_pointer(data)
}
```

This is still the code we want. However, there is a slight wrinkle.

这仍然是我们想要的代码。然而，这里有一个轻微的皱纹。

## Constructing the vtable: Async functions need a shim to return a `Box`

## 构造vtable：异步函数需要填充程序才能返回`Box`

In the `next` method above, the type of the function pointer that we extracted from the vtable was the following:

在上面的`next`方法中，我们从vtable提取的函数指针类型如下：

```rust
fn(*mut RuntimeType) -> Box<dyn Future<Output = Option<I>> + ‘_>
```

However, the signature of the function in the impl is different! It doesn’t return a `Box`, it returns an `impl Future`! Somehow we have to bridge this gap. What we need is a kind of “shim function”, something like this:

但是，Impl中函数的签名不同！它不返回`Box`，它返回一个`impl Future`！我们必须设法弥合这一鸿沟。我们需要的是一种“填补功能”，就像这样：

```rust
fn next_box_shim<T: AsyncIter>(this: &mut T) -> Box<dyn Future<Output = Option<I>> + ‘_> {
    let future: impl Future<Output = Option<I>> = AsyncIter::next(this);
    Box::new(future)
}
```

Now the vtable for `SleepyRange` can store `next_box_shim::<SleepyRange>` instead of storing `<SleepyRange as AsyncIter>::next` directly.

现在`SleepyRange`的vtable可以存储`Next_box_shim：：<SleepyRange>`，而不是直接存储`<SleepyRange as AsyncIter>：：next`。

## Extending the `AssociatedFn` trait

## 扩展`AssociatedFn`特性

In my previous post, I sketched out the idea of an `AssociatedFn` trait that had an associated type `FnPtr`. If we wanted to make the construction of this sort of shim automated, we would want to change that from an associated type into its own trait. I’m imagining something like this:

在我之前的帖子中，我勾勒出了一个`AssociatedFn`特征的概念，它具有一个关联的类型`FnPtr`。如果我们想使这种填充程序的构造自动化，我们会希望将其从关联类型更改为它自己的特征。我的想象是这样的：

```rust
trait AssociatedFn { }
trait Reify<F>: AssociatedFn {
    fn reify(self) -> F; 
}
```

where `A: Reify<F>` indicates that the associated function `A` can be “reified” (made into a function pointer) for a function type `F`. The compiler could implement this trait for the direct mapping where possible, but also for various kinds of shims and ABI transformations. For example, the `AsyncIter::next` method might implement`Reify<fn(*mut ()) -> Box<dyn Future<..>>>` to allow a “boxing shim” to be constructed and so forth.

其中，`A：Reify<F>`表示关联的函数`A`可以对函数类型`F`进行实例化(变成函数指针)。编译器可以在可能的情况下为直接映射实现这一特性，也可以为各种垫片和ABI转换实现这一特性。例如，`AsyncIter：：next`方法可以实现`Reify<fn(*mut())->Box<dyn Future<..>>`，以允许构造“boking shim”等等。

## Other sorts of shims

## 其他种类的垫片

There are other sorts of limitations around dyn traits that could be overcome with judicious use of shims and tweaked vtables, at least in some cases. As an example, consider this trait:

围绕dyn特征还有其他类型的限制，可以通过明智地使用垫片和微调vtable来克服，至少在某些情况下是这样。举个例子，考虑一下这个特性：

```rust
pub trait Append {
    fn append(&mut self, values: impl Iterator<Item = u32>);
}
```

This trait is not traditionally dyn-safe because the `append` function is generic and requires monomorphization for each kind of iterator — therefore, we don’t know which version to put in the vtable for `Append`, since we don’t yet know the types of iterators it will be applied to! But what if we just put *one* version, the case where the iterator type is `&mut dyn Iterator<Item = u32>`? We could then tweak the `impl Append for dyn Append` to create this `&mut dyn Iterator` and call the function from the vtable:

这个特性在传统上不是dyn安全的，因为`append`函数是泛型的，并且需要对每种迭代器进行单形化--因此，我们不知道应该将哪个版本放入`append`的vtable中，因为我们还不知道它将应用于什么类型的迭代器！但是如果我们只放一个版本，迭代器类型是`&mut dyn Iterator<Item=u32>`会怎么样？然后，我们可以调整`impl append for dyn Append`来创建这个`&mut dyn Iterator`，并从vtable调用函数：

```rust
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

## Conclusion

## 结论

So where does this leave us? The core building blocks for “dyn async traits” seem to be:

那么，这让我们处于什么境地呢？“动态异步化特征”的核心组成部分似乎是：

* The ability to customize the contents of the vtable that gets generated for a trait. 
  * For example, async fns need shim functions that box the output.
* The ability to customize the dispatch logic (`impl Foo for dyn Foo`).
* The ability to customize associated types like `Next` to be a `Box<dyn>`:
  * This requires the ability to extract the vtable, as given by \[RFC 2580\].
  * It also requires the ability to extract functions from the vtable (not presently supported).

I said at the outset that I was going to assume, for the purposes of this post, that we wanted to return a `Box<dyn>`, and I have.  It seems possible to extend these core capabilities to other sorts of return types (such as other smart pointers), but it’s not entirely trivial; we’d have to define what kinds of shims the compiler can generate. 

能够定制为特征生成的vtable的内容。例如，异步fns需要填充函数来装箱输出。能够定制分派逻辑(`impl foo for dyn Foo‘)。能够将像`Next`这样的关联类型定制为`Box<dyn>`：这需要能够提取vtable，正如[RFC 2580]所给出的那样。它还需要能够从vtable中提取函数(目前不支持)。我在一开始就说过，为了本文的目的，我将假定我们想要返回`Box<dyn>`，我已经做到了。似乎可以将这些核心功能扩展到其他类型的返回类型(如其他智能指针)，但这并不完全是微不足道的；我们必须定义编译器可以生成哪些类型的填补。

I haven’t really thought very hard about how we might allow users to specify each of those building blocks, though I sketched out some possibilities. At this point, I’m mostly trying to explore the possibilities of what kinds of capabilities may be useful or necessary to expose. 

虽然我勾勒出了一些可能性，但我并没有认真考虑过如何允许用户指定每一个构建块。在这一点上，我主要是尝试探索公开哪些类型的功能可能是有用的或必要的。