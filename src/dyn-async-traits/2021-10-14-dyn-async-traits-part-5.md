# Dyn async traits

[原文](https://smallcultfollowing.com/babysteps/blog/2021/10/14/dyn-async-traits-part-5/) |
日期：2021-10-14 13:46 -0400

如果你愿意使用 nightly，那么已经可以通过 GATs 和 impl Trait 来对 traits 中的异步函数进行建模 ——
这就是 [Embassy] 异步运行时所做的事情，也是 [real-async-trait] crate 所做的事情。然而，一个缺点是你的 trait 不支持动态分发。

在本系列的前几篇文章中，我一直在探索造成这种限制的一些原因，以及需要在语言中公开什么样的原语功能 (primitive capabilities) 来克服它。

以前我的想法是，可以尝试通过启动实验的计划来稳定这些原语功能。现在我仍然支持这个计划，但昨天意识到了一件事：使用过程宏，你几乎可以在今天进行这种实验！

不幸的是，由于 Rust 类型系统中的一些相对模糊的规则，它并不能很好地工作（也许一些聪明的读者会找到变通的办法；而那些模糊的规则是我一段时间以来一直想要更改的）。

明确地说：这篇文章并不是为了描述 trait 中的异步函数的“理想最终状态”。

我仍然希望在 trait 中编写 `async fn` 而不需要任何进一步的宏注释，并使 trait 完全有能力支持静态分发和 dyn 模式，同时坚持零成本抽象[^zca]的原则。

但这里有一些重要的问题，为了找到这些问题的最佳答案，我们需要进行更多的探索，这就是本文的重点。

[^zca]: 用 Bjarne Stroustroup 的话说：“你不用支付不使用的东西。更进一步的是：对真正所使用的东西，你手工编写的代码不会比零成本抽象更好。

[Embassy]: https://github.com/embassy-rs/embassy
[real-async-trait]: https://crates.io/crates/real-async-trait

## 代码在 GitHub 上

这篇博客文章中涉及的代码有原型，可以[在 GitHub 上获得][prototyped]。不过，见本文末尾的警告！

[prototyped]: https://github.com/nikomatsakis/ergo-dyn/blob/main/examples/async-iter-manual-desugar.rs

## 设计目标

为了理解我的想法，让我们回到我最喜欢的 trait `AsyncIter`：

```rust,ignore
trait AsyncIter {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}
```

这篇文章将介绍如何将类似上面的 trait 声明转换为另一系列声明，以实现以下目标：

* 我们可以将其用作泛型约束，如 `fn foo<T: AsyncIter>()`，此时将获得静态分发、完整的 auto trait 支持以及 Rust 中通常附带的泛型约束的所有其他好处
* 给定 `T: AsyncIter`，我们可以将其强制为使用虚拟分发的某种形式 `DynAsyncIter`。此时，类型不会显示为具体的 `T` 或具体的  Future 类型
  * 我故意写了 `DynAsyncIter`，而不是 `dyn AsyncIter`，因为我们将创建自己的类型，它的行为类似于 `dyn` 类型，但它管理异步所需的改写。
  * 为简单起见，我们假设想要把 Future 放入 Box 。不过，这种设计的部分意义在于，它为我们提供了生成任何包装类型的空间。

你可以自己编写我在此处展示的代码，但更好的方法是将其打包为一种装饰器（例如 `#[async_trait_v2]`[^name]）。

[^name]: 天哪，我需要一个更吸引人的名字！

## 基础：带 GAT 的 trait

第一步是转换 trait，让其具有 GAT 和常规的 `fn`，就像我们多次看到的那样：

```rust,ignore
trait AsyncIter {
    type Item;

    type Next<'me>: Future<Output = Option<Self::Item>>
    where
        Self: 'me;

    fn next(&mut self) -> Self::Next<'_>;
}
```

## 定义 `DynAsyncIter`

下一步是管理 trait 的动态分发版本。为此，我们将创建 `DynAsyncIter`。它充当 `Box<dyn AsyncIter>` trait object。

可通过调用带有特定迭代器类型的 `DynAsyncIter::from` 来创建它的实例；`DynAsyncIter` 实现了 `AsyncIter` trait，所以你就可以像往常一样调用 `next`：

```rust,ignore
let the_iter: DynAsyncIter<u32> = DynAsyncIter::from(some_iterator);
process_items(&mut the_iter);

async fn sum_items(iter: &mut impl AsyncIter<Item = u32>) -> u32 {
    let mut s = 0;
    while let Some(v) = the_iter.next().await {
        s += v;
    }
    s
}
```

### 结构体定义

`DynAsyncIter` 以 `Box<dyn AsyncIter>` trait object 为模型，它将为 trait 中声明的每个普通关联类型（不包括为异步函数返回类型的 GATs）提供一个泛型参数。

它有两个字段，数据指针（一个裸指针的 Box）和 vtable。我们不知道底层值的类型，因此使用 `ErasedData`：

```rust,ignore
type ErasedData = ();

pub struct DynAsyncIter<Item> {
    data: *mut ErasedData,
    vtable: &'static DynAsyncIterVtable<Item>,
}
```

对于 vtable，我们再创建一个结构体 `DynAsyncIterVtable`，来为 trait 中的每个方法创建相应的 `fn`。

与内置 vtable 不同的是，我们把这些函数的返回类型修改为 boxed Future：

```rust,ignore
struct DynAsyncIterVtable<Item> {
    drop_fn: unsafe fn(*mut ErasedData),
    next_fn: unsafe fn(&mut *mut ErasedData) -> Box<dyn Future<Output = Option<Item>> + ‘_>,
}
```

### 实现 `AsyncIter` trait 

接下来，为 `DynAsyncIter` 实现 `AsyncIter` trait。对于引入的每个新的 GAT，我们只使用一个 boxed Future 类型。在方法内部，从 vtable 中提取函数指针并调用它：

```rust,ignore
impl<Item> AsyncIter for DynAsyncIter<Item> {
    type Item = Item;

    type Next<'me> = Box<dyn Future<Output = Option<Item>> + 'me>;

    fn next(&mut self) -> Self::Next<'_> {
        let next_fn = self.vtable.next_fn;
        unsafe { next_fn(&mut self.data) }
   }
}
```

这里的 `unsafe` 关键字是断言满足 `next_fn` 的安全条件。我们将在稍后更详细地讨论这一点，但简而言之，这些条件是：

* vtable 对应于某个已擦除的类型 `T: AsyncIter`
* `*mut ErasedData` 的每个实例都指向该有效的 `Box<T>`

### 实现 `Drop` trait

```rust,ignore
impl Drop for DynAsyncIter {
    fn drop(&mut self) {
        let drop_fn = self.vtable.drop_fn;
        unsafe { drop_fn(self.data); }
    }
}
```

我们需要通过 vtable 调用，因为不知道拥有什么类型的数据，所以不知道如何正确地 drop 它。

## 创建 `DynAsyncIter` 实例

要创建这些`dyAsyncIter`对象之一，我们可以实现`From` trait 。这将分配一个框，将其强制为原始指针，然后将其与vtable组合：

```rust,ignore
impl<Item, T> From<T> for DynAsyncIter<Item>
where
    T: AsyncIter<Item = Item>,
{
    fn from(value: T) -> DynAsyncIter {
        let boxed_value = Box::new(value);
        DynAsyncIter {
            data: Box::into_raw(boxed_value) as *mut (),
            vtable: dyn_async_iter_vtable::<T>(), // we’ll cover this fn later
        }
    }
}
```

## 填充 vtable

现在我们来看最有趣的部分：如何为对象创建 vtable？回想一下我们的 vtable 是一个结构体，如下所示：

```rust,ignore
struct DynAsyncIterVtable<Item> {
    drop_fn: unsafe fn(*mut ErasedData),
    next_fn: unsafe fn(&mut *mut ErasedData) -> Box<dyn Future<Output = Option<Item>> + '_>,
}
```

我们需要为每一个字段创建值。在普通的 `dyn` 中，这些是直接指向来自 `impl` 方法的指针，但对我们来说，它们是围绕核心 trait  函数的“包装函数”。

这些包装器的作用是引入一些次要的强制转化，比如给产生的 Future 分配一个 Box，以及从“擦除的数据”转化到真正的类型：

```rust,ignore
// Safety conditions:
//
// The `*mut ErasedData` is actually the raw form of a `Box<T>` 
// that is valid for 'a.
unsafe fn next_wrapper<'a, T>(
    this: &'a mut *mut ErasedData,
) -> Box<dyn Future<Output = Option<T::Item>> + 'a
where
    T: AsyncIter,
{
    let unerased_this: &mut Box<T> = unsafe { &mut *(this as *mut Box<T>) };
    let future: T::Next<'_> = <T as AsyncIter>::next(unerased_this);
    Box::new(future)
}
```

我们还需要一个 drop 包装器：

```rust,ignore
// Safety conditions:
//
// The `*mut ErasedData` is actually the raw form of a `Box<T>` 
// and this function is being given ownership of it.
fn drop_wrapper<T>(
    this: *mut ErasedData,
)
where
    T: AsyncIter,
{
    let unerased_this = Box::from_raw(this as *mut T);
    drop(unerased_this); // Execute destructor as normal
}
```

## 构造 vtable

回想一下， `From` impl 调用了 `dyn_async_iter_vtable::<T>` 函数。该函数如下所示：

```rust,ignore
fn dyn_async_iter_vtable<T>() -> &'static DynAsyncIterVtable<T::Item>
where
    T: AsyncIter,
{
    const {
        &DynAsyncIterVtable {
            drop_fn: drop_wrapper::<T>,
            next_fn: next_wrapper::<T>,
        }
    }
}
```

这构造了一个带有两个函数指针的结构体，里面包含只静态数据，因此我们可以返回对它的 `&'static` 引用。

完成！

## 警告并请求帮助

不幸的是，这种设置并不像我所描述的那样工作。有两个问题：

* `const` 函数和表达式仍然有很多限制，特别是在 `T` 这样的泛型方面，我无法让它们工作
* 由于 [RFC 1214] 引入的规则，`&'static DynAsyncIterVtable<T: Item>` 类型要求 `T::Item: 'static`，这里可能不是这样的。这个条件可能不是必需的，但编译器现在强制执行它。

[RFC 1214]: https://rust-lang.github.io/rfcs/1214-projections-lifetimes-and-wf.html

我最终用了一些可怕的东西，使用时将 `T:: Item` 类型擦除，并使用 `Box::leak` 获得 `&'static` 引用，只是为了证明这个概念。

我几乎不好意思展示代码，但代码在
[这里](https://github.com/nikomatsakis/ergo-dyn/blob/3503770e08177a6d59e202f88cb7227863331685/examples/async-iter-manual-desugar.rs#L107-L118)。

我知道有人运用了一些非常聪明的技巧，所以我很好奇我是否遗漏了什么，有没有办法在现在的 Rust 上构建这个 vtable。

无论如何，如果我们想这样做的话，扩展 `const` 和其他一些东西来支持这种情况似乎是一个相对较轻的提升。

## 结论

这篇博客文章提供了一种方法来实现我一直在谈论的动态分发的想法，只使用当前存在的功能，并总地来说正在走向稳定的道路上。

这让我兴奋，因为这意味着我们可以开始进行测量和实验。

例如，我真的很想知道从 `async-trait` 过渡到这里描述的静态分发和 Box 方式的动态分发相结合的方案对性能的影响。

我还想探索是否有其他方法来包装 Futures （例如使用本地任务分配器或其他智能指针）来让执行效率更高。这将有助于告知我们最终需要什么样的功能。

除了 async，我还对 `dyn` 的一般模型感兴趣。

一个明显的例子是，我在这里实现的“始终 Box”的版本有一些运行时成本（需要一次分配！）并不是适用于所有环境，但它会更符合人体工程学。

trait object 可能是 Sized，并且将在更多的上下文中透明地工作。我们还可以制作不同类型的 vtable 形式。
