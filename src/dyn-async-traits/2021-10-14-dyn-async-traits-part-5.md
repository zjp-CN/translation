---

## layout: post
title: Dyn async traits, part 5
date: 2021-10-14 13:46 -0400

## 版面：帖子标题：Dyn Async特征，第5部分日期：2021-10-14 13：46-0400

If you’re willing to use nightly, you can already model async functions in traits by using GATs and impl Trait — this is what the [Embassy](https://github.com/embassy-rs/embassy) async runtime does, and it’s also what the [real-async-trait](https://crates.io/crates/real-async-trait) crate does. One shortcoming, though, is that your trait doesn’t support dynamic dispatch. In the previous posts of this series, I have been exploring some of the reasons for that limitation, and what kind of primitive capabilities need to be exposed in the language to overcome it. My thought was that we could try to stabilize those primitive capabilities with the plan of enabling experimentation. I am still in favor of this plan, but I realized something yesterday: **using procedural macros, you can ALMOST do this experimentation today!** Unfortunately, it doesn't quite work owing to some relatively obscure rules in the Rust type system (perhaps some clever readers will find a workaround; that said, these are rules I have wanted to change for a while).

如果您愿意每晚使用，那么您已经可以通过使用Gats和Impl特征来对特征中的异步函数进行建模--这就是大使馆异步运行时所做的事情，也是真正的异步特征箱所做的事情。然而，一个缺点是您的特征不支持动态分派。在本系列的前几篇文章中，我一直在探索造成这种限制的一些原因，以及需要在语言中公开什么样的原始功能来克服它。我的想法是，我们可以尝试通过启动实验的计划来稳定这些原始的能力。我仍然支持这个计划，但我昨天意识到了一件事：使用过程性宏，您几乎可以在今天进行这种实验！不幸的是，由于Rust类型系统中的一些相对模糊的规则(也许一些聪明的读者会找到解决办法；也就是说，这些规则是我一段时间以来一直想要更改的)，它并不能很好地工作。

**Just to be crystal clear:** Nothing in this post is intended to describe an “ideal end state” for async functions in traits. I still want to get to the point where one can write `async fn` in a trait without any further annotation and have the trait be “fully capable” (support both static dispatch and dyn mode while adhering to the tenets of zero-cost abstractions[^zca]). But there are some significant questions there, and to find the best answers for those questions, we need to enable more exploration, which is the point of this post.

明确地说：这篇文章并不是为了描述特征中的异步功能的“理想结束状态”。我仍然希望在特征中编写`async fn`而不需要任何进一步的注释，并使特征“完全有能力”(支持静态分派和动态模式，同时坚持零成本抽象的原则)。但这里有一些重要的问题，为了找到这些问题的最佳答案，我们需要进行更多的探索，这就是本文的重点。

[^zca]: In the words of Bjarne Stroustroup, “What you don’t use, you don’t pay for. And further: What you do use, you couldn’t hand code any better.”

用Bjarne Stroustroup的话说：“你不用的东西，你就不用付钱。更进一步的是：你使用的是什么，你的手工代码再好不过了。

### Code is on github

### 代码在GitHub上

The code covered in this blog post has been prototyped and is [available on github](https://github.com/nikomatsakis/ergo-dyn/blob/main/examples/async-iter-manual-desugar.rs). See the caveat at the end of the post, though!

这篇博客文章中涉及的代码已经过原型制作，可以在GitHub上获得。不过，请参见帖子末尾的警告！

### Design goal

### 设计目标

To see what I mean, let’s return to my favorite trait, `AsyncIter`:

为了理解我的意思，让我们回到我最喜欢的特征，`异步者‘：

```rust
trait AsyncIter {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}
```

The post is going to lay out how we can transform a trait declaration like the one above into a series of declarations that achieve the following:

这篇文章将介绍如何将类似上面的特征声明转换为一系列声明，以实现以下目标：

* We can use it as a generic bound (`fn foo<T: AsyncIter>()`), in which case we get static dispatch, full auto trait support, and all the other goodies that normally come with generic bounds in Rust.
* Given a `T: AsyncIter`, we can coerce it into some form of `DynAsyncIter` that uses virtual dispatch. In this case, the type doesn’t reveal the specific `T` or the specific types of the futures.
  * I wrote `DynAsyncIter`, and not `dyn AsyncIter` on purpose — we are going to create our own type that acts *like* a `dyn` type, but which manages the adaptations needed for async.
  * For simplicity, let’s assume we want to box the resulting futures. Part of the point of this design though is that it leaves room for us to generate whatever sort of wrapping types we want.

You could write the code I’m showing here by hand, but the better route would be to package it up as a kind of decorator (e.g., `#[async_trait_v2]`[^name]).

我们可以将其用作泛型绑定(`fn foo<T：AsyncIter>()`)，在这种情况下，我们将获得静态调度、全自动特征支持以及Rust中通常附带的泛型约束的所有其他好处。给定`t：AsyncIter`，我们可以将其强制为使用虚拟调度的某种形式的`dyAsyncIter`。在这种情况下，类型不会显示特定的`T‘或未来的特定类型。我编写了`dyAsyncIter`，而不是故意编写的`dyn AsyncIter`-我们将创建自己的类型，它的行为类似于`dyn`类型，但它管理异步所需的适配。为简单起见，我们假设我们想要装箱结果的未来。不过，这种设计的部分意义在于，它为我们提供了生成任何类型的包装类型的空间。您可以手动编写我在此处展示的代码，但更好的方法是将其打包为一种装饰符(例如，`#[Async_特征_v2]`)。

[^name]: Egads, I need a snazzier name than that!

天哪，我需要一个更时髦的名字！

### The basics: trait with a GAT

### 基本特征：具有GAT的特质

The first step is to transform the trait to have a GAT and a regular `fn`, in the way that we’ve seen many times:

第一步是将特征转换为具有GAT和常规的`fn‘，就像我们多次看到的那样：

```rust
trait AsyncIter {
    type Item;

    type Next<‘me>: Future<Output = Option<Self::Item>>
    where
        Self: ‘me;

    fn next(&mut self) -> Self::Next<‘_>;
}
```

### Next: define a “DynAsyncIter” struct

### 下一步：定义“dyAsyncIter”结构

The next step is to manage the virtual dispatch (dyn) version of the trait. To do this, we are going to “roll our own” object by creating a struct `DynAsyncIter`. This struct plays the role of a `Box<dyn AsyncIter>` trait object. Instances of the struct can be created by calling `DynAsyncIter::from` with some specific iterator type; the `DynAsyncIter` type implements the `AsyncIter` trait, so once you have one you can just call `next` as usual:

下一步是管理特征的虚拟分派(Dyn)版本。为此，我们将通过创建一个结构`dyAsyncIter`来“滚动我们自己的”对象。此结构充当`Box<dyn AsyncIter>`特征对象。可以通过调用带有特定迭代器类型的`dyAsyncIter：：From`来创建结构的实例；`dyAsyncIter`类型实现了`AsyncIter`特征，所以一旦有了，就可以像往常一样调用`next`：

```rust
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

### Struct definition

### 结构定义

Let’s look at how this `DynAsyncIter` struct is defined. First, we are going to “roll our own” object by creating a struct `DynAsyncIter`. This struct is going to model a `Box<dyn AsyncIter>` trait object; it will have one generic parameter for every ordinary associated type declared in the trait (not including the GATs we introduced for async fn return types). The struct itself has two fields, the data pointer (a box, but in raw form) and a vtable. We don’t know the type of the underlying value, so we’ll use `ErasedData` for that:

让我们来看看这个`dyAsyncIter`结构是如何定义的。首先，我们将通过创建一个结构`dyAsyncIter`来“滚动我们自己的”对象。此结构将对`Box<dyn AsyncIter>`特征对象建模；它将为特征中声明的每个普通关联类型(不包括我们为异步FN返回类型引入的Gats)提供一个泛型参数。该结构本身有两个字段，数据指针(框，但为原始形式)和vtable。我们不知道底层值的类型，因此我们将使用`ErasedData`：

```rust
type ErasedData = ();

pub struct DynAsyncIter<Item> {
    data: *mut ErasedData,
    vtable: &’static DynAsyncIterVtable<Item>,
}
```

For the vtable, we will make a struct that contains a `fn` for each of the methods in the trait. Unlike the builtin vtables, we will modify the return type of these functions to be a boxed future:

对于vtable，我们将为特征中的每个方法创建一个包含`fn`的结构。与内置vtable不同，我们将把这些函数的返回类型修改为盒装的未来：

```rust
struct DynAsyncIterVtable<Item> {
    drop_fn: unsafe fn(*mut ErasedData),
    next_fn: unsafe fn(&mut *mut ErasedData) -> Box<dyn Future<Output = Option<Item>> + ‘_>,
}
```

### Implementing the AsyncIter trait

### 实现AsyncIter特征

Next, we can implement the `AsyncIter` trait for the `DynAsyncIter` type. For each of the new GATs we introduced, we simply use a boxed future type. For the method bodies, we extract the function pointer from the vtable and call it:

接下来，我们可以为`dyAsyncIter`类型实现`AsyncIter`特征。对于我们引入的每个新的Gat，我们只使用一个盒装的Future类型。对于方法体，我们从vtable中提取函数指针并调用它：

```rust
impl<Item> AsyncIter for DynAsyncIter<Item> {
    type Item = Item;

    type Next<‘me> = Box<dyn Future<Output = Option<Item>> + ‘me>;

    fn next(&mut self) -> Self::Next<‘_> {
        let next_fn = self.vtable.next_fn;
        unsafe { next_fn(&mut self.data) }
   }
}
```

The unsafe keyword here is asserting that the safety conditions of `next_fn` are met. We’ll cover that in more detail later, but in short those conditions are:

这里的不安全关键字是断言满足`NEXT_fn`的安全条件。我们将在稍后更详细地讨论这一点，但简而言之，这些条件是：

* The vtable corresponds to some erased type `T: AsyncIter`…
* …and each instance of `*mut ErasedData` points to a valid `Box<T>` for that type.

### Dropping the object

### VTABLE对应于某个已擦除的类型`t：AsyncIter`……并且`*mut ErasedData`的每个实例都指向该类型的有效`Box

Speaking of Drop, we do need to implement that as well. It too will call through the vtable:

说到Drop，我们也需要实现这一点。它也将通过vtable调用：

```rust
impl Drop for DynAsyncIter {
    fn drop(&mut self) {
        let drop_fn = self.vtable.drop_fn;
        unsafe { drop_fn(self.data); }
    }
}
```

We need to call through the vtable because we don’t know what kind of data we have, so we can’t know how to drop it correctly.

我们需要通过vtable调用，因为我们不知道我们拥有什么类型的数据，所以我们不知道如何正确地删除它。

### Creating an instance of `DynAsyncIter`

### 创建`dyAsyncIter`的实例

To create one of these `DynAsyncIter` objects, we can implement the `From` trait. This allocates a box, coerces it into a raw pointer, and then combines that with the vtable:

要创建这些`dyAsyncIter`对象之一，我们可以实现`From`特征。这将分配一个框，将其强制为原始指针，然后将其与vtable组合：

```rust
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

### Creating the vtable shims

### 创建vtable垫片

Now we come to the most interesting part: how do we create the vtable for one of these objects? Recall that our vtable was a struct like so:

现在我们来看最有趣的部分：如何为这些对象之一创建vtable？回想一下我们的vtable是一个结构，如下所示：

```rust
struct DynAsyncIterVtable<Item> {
    drop_fn: unsafe fn(*mut ErasedData),
    next_fn: unsafe fn(&mut *mut ErasedData) -> Box<dyn Future<Output = Option<Item>> + ‘_>,
}
```

We are going to need to create the values for each of those fields. In an ordinary `dyn`, these would be pointers directly to the methods from the `impl`, but for us they are “wrapper functions” around the core trait functions. The role of these wrappers is to introduce some minor coercions, such as allocating a box for the resulting future, as well as to adapt from the “erased data” to the true type:

我们需要为这些字段中的每一个创建值。在普通的“dyn”中，这些是直接指向来自“impl`”的方法的指针，但对我们来说，它们是围绕核心特征函数的“包装函数”。这些包装器的作用是引入一些次要的强制，比如为产生的将来分配一个框，以及从“擦除的数据”适应到真实类型：

```rust
// Safety conditions:
//
// The `*mut ErasedData` is actually the raw form of a `Box<T>` 
// that is valid for ‘a.
unsafe fn next_wrapper<‘a, T>(
    this: &’a mut *mut ErasedData,
) -> Box<dyn Future<Output = Option<T::Item>> + ‘a
where
    T: AsyncIter,
{
    let unerased_this: &mut Box<T> = unsafe { &mut *(this as *mut Box<T>) };
    let future: T::Next<‘_> = <T as AsyncIter>::next(unerased_this);
    Box::new(future)
}
```

We’ll also need a “drop” wrapper:

我们还需要一个“Drop”包装器：

```rust
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

### Constructing the vtable

### 构造vtable

Now that we’ve defined the wrappers, we can construct the vtable itself. Recall that the `From` impl called a function `dyn_async_iter_vtable::<T>`. That function looks like this:

现在我们已经定义了包装器，我们可以构造vtable本身了。回想一下，`From`Iml调用了函数`dyn_async_iter_vtable：：<T>`。该函数如下所示：

```rust
fn dyn_async_iter_vtable<T>() -> &’static DynAsyncIterVtable<T::Item>
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

This constructs a struct with the two function pointers: this struct only contains static data, so we are allowed to return a `&’static` reference to it.

这构造了一个带有两个函数指针的结构：该结构只包含静态数据，因此我们可以返回对它的`&‘static`引用。

Done!

好了！

### And now the caveat, and a plea for help

### 现在是警告，并请求帮助

Unfortunately, this setup doesn't work quite how I described it. There are two problems:

不幸的是，这种设置并不像我所描述的那样工作。有两个问题：

* `const` functions and expressions stil lhave a lot of limitations, especially around generics like `T`, and I couldn't get them to work;
* Because of the rules introduced by [RFC 1214](https://rust-lang.github.io/rfcs/1214-projections-lifetimes-and-wf.html), the `&’static DynAsyncIterVtable<T::Item>` type requires that `T::Item: 'static`, which may not be true here. This condition perhaps shouldn't be necessary, but the compiler currently enforces it.

I wound up hacking something terrible that erased the `T::Item` type into uses and used `Box::leak` to get a `&'static` reference, just to prove out the concept. I'm almost embarassed to [show the code](https://github.com/nikomatsakis/ergo-dyn/blob/3503770e08177a6d59e202f88cb7227863331685/examples/async-iter-manual-desugar.rs#L107-L118), but there it is. 

\`const`函数和表达式仍然有很多限制，特别是在`T`这样的泛型方面，我无法让它们工作；由于RFC1214引入的规则，`&‘静态dyAsyncIterVtable<T：：Item>`类型要求`t：：Item：’Static`，这里可能不是这样的。这个条件可能不是必需的，但编译器现在强制执行它。我最终破解了一些可怕的东西，将`t：：Item`类型擦除到Use中，并使用`Box：：Leak`获得`&‘static`引用，只是为了证明这个概念。我几乎不好意思展示代码，但它就在那里。

Anyway, I know people have done some pretty clever tricks, so I'd be curious to know if I'm missing something and there *is* a way to build this vtable on Rust today. Regardless, it seems like extending `const` and a few other things to support this case is a relatively light lift, if we wanted to do that.

无论如何，我知道人们做了一些非常聪明的把戏，所以我很好奇我是否遗漏了什么，今天有办法在Rust上构建这个vtable。无论如何，扩展`const`和其他一些东西来支持这种情况似乎是一个相对较轻的提升，如果我们想这样做的话。

### Conclusion

### 结论

This blog post presented a way to implement the dyn dispatch ideas I've been talking using only features that currently exist and are generally en route to stabilization. That's exiting to me, because it means that we can start to do measurements and experimentation. For example, I would really like to know the performance impact of transitiong from `async-trait` to a scheme that uses a combination of static dispatch and boxed dynamic dispatch as described here. I would also like to explore whether there are other ways to wrap futures (e.g., with task-local allocators or other smart pointers) that might perform better. This would help inform what kind of capabilities we ultimately need.

这篇博客文章提供了一种方法来实现我一直在谈论的Dyn调度想法，只使用当前存在的功能，通常正在走向稳定的道路上。这对我来说是令人兴奋的，因为这意味着我们可以开始进行测量和实验。例如，我真的很想知道从`async-trait`过渡到这里描述的静态调度和盒式动态调度相结合的方案对性能的影响。我还想探索是否有其他方法来包装期货(例如，使用任务本地分配器或其他智能指针)，可能会执行得更好。这将有助于告知我们最终需要什么样的能力。

Looking beyond async, I'm interested in tinkering with different models for `dyn` in general. As an obvious example, the "always boxed" version I implemented here has some runtime cost (an allocation!) and isn't applicable in all environments, but it would be far more ergonomic. Trait objects would be Sized and would transparently work in far more contexts. We can also prototype different kinds of vtable adaptation.

除了异步机，我还对“dyn”的不同机型感兴趣。作为一个明显的例子，我在这里实现的“始终装箱”版本有一些运行时成本(分配！)并不是适用于所有环境，但它会更符合人体工程学。特征对象的大小将被调整，并且将在更多的上下文中透明地工作。我们还可以制作不同类型的vtable适应的原型。