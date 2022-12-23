# AFIT 如何在 rustc 中工作

原文：[How Async Functions in Traits could Work in Rustc](raw) | by Eric Holk | 2022 年 4 月 18 日

[raw]: https://blog.theincredibleholk.org/blog/2022/04/18/how-async-functions-in-traits-could-work-in-rustc/

[wg-async]: https://rust-lang.github.io/wg-async/vision/roadmap/async_fn.html

Rust Async 工作组的主要目标之一是 [允许在任何允许使用 `fn` 的地方使用 `async fn`][wg-async]，特别是在 trait 方面。

在这篇文章中，我想提炼一些已被提议的设计，并展示如何实现 trait 中的异步函数。

我们将研究一种可能的工作方式，尽管我想强调这不是唯一的方式，最终将采用的设计的许多细节仍在制定之中。

# 效果

我们希望以下程序能够运行。

```rust
// 点击右上角按钮允许这段代码
use std::sync::Arc;

trait AsyncTrait {
    async fn get_number(&self) -> i32;
}

impl AsyncTrait for i32 {
    async fn get_number(&self) -> i32 {
        *self
    }
}

async fn print_the_number(from: Arc<dyn AsyncTrait>) {
    let number = from.get_number().await;
    println!("The number is {number}");
}

#[tokio::main]
async fn main() {
    let number_getter = Arc::new(42);
    print_the_number(number_getter).await;
} 
```

这个简短的程序演示了 trait 中的异步函数的几乎所有功能。

特别从 `AsyncTrait` 开始，这是一个只有一个异步方法的 trait。此 trait 可以在静态上下文 (static context) 中使用，例如：

```rust,ignore
async fn print_the_number_static(from: impl AsyncTrait) {
    let number = from.get_number().await;
    println!("The number is {number}");
} 
```

我们将在 [静态 trait][static-traits] 一节中了解如何做到这一点。

在本例中，`print_the_number` 函数动态调用 `get_number`。我们将在 [动态 trait][dynamic-traits] 那节探讨这一点。

[static traits]: #static-traits

[dynamic traits]: #dynamic-traits

# 现状

以上示例的 [同步版本][sync-version] 在今天 Rust 中可以工作（即如果我们擦除程序中所有的 `async` 和 `.await`）：

```rust
use std::sync::Arc;

trait Trait {
     fn get_number(&self) -> i32;
}

impl Trait for i32 {
     fn get_number(&self) -> i32 {
        *self
    }
}

 fn print_the_number(from: Arc<dyn Trait>) {
    let number = from.get_number();
    println!("The number is {number}");
}

 fn main() {
    let number_getter = Arc::new(42);
    print_the_number(number_getter);
} 
```

[sync-version]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=f463d4aece0f23c5d7f45026143e6705

[async-version]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=48145a43fe9b942611d465d3e4ba817f

[`async_trait`]: https://crates.io/crates/async-trait

我们也可以在 Rust 中做到 [异步版本][async-version]，但需要使用 [`async_trait`] 库：

```rust
use std::sync::Arc;
use async_trait::async_trait;

#[async_trait]
trait AsyncTrait {
    async fn get_number(&self) -> i32;
}

#[async_trait]
impl AsyncTrait for i32 {
    async fn get_number(&self) -> i32 {
        *self
    }
}

async fn print_the_number(from: Arc<dyn AsyncTrait>) {
    let number = from.get_number().await;
    println!("The number is {number}");
}

#[tokio::main]
async fn main() {
    let number_getter = Arc::new(42);
    print_the_number(number_getter).await;
} 
```

我们希望不借助任何额外的库做到异步。此外，`async_trait` 库的工作原理是将来自任何异步方法的返回的 Future
放入 Box，这意味着产生额外的分配。如果能够避免这种情况，那就太好了。

在这篇文章的其余部分，我们将从 [静态 trait][static traits] 中的异步函数开始，看看这可能是如何发生的。

<a name="static-traits"></a>

# 静态 traits 中的异步函数

## 第 1 步：使用 TAIT 解语法糖

（译者注： TAIT 是 Type Alias Impl Trait 的缩写，见 [RFC 2515]）

[RFC 2515]: https://rust-lang.github.io/rfcs/2515-type_alias_impl_trait.html

目前，对于顶层的异步函数，编译器按如下方式对该函数进行解糖：

```rust
#use std::future::Future;
// 之前
async fn foo() -> i32 {
    42
} 

// 之后
fn foo_after() -> impl Future<Output = i32> {
    async {
        42
    }
} 
```

从根本上说，异步函数是返回 Future 的函数。

那个 Future 有一个具体的类型，但它是不可命名的，因为它隐藏在 `impl Future<Output = i32>` 后面。我们所知道的是，它是实现 `Future` trait 的东西。

现在，让我们试着为 trait 做同样的事情。

```rust
trait AsyncTrait {
    async fn get_number(&self) -> i32;
}

impl AsyncTrait for i32 {
    async fn get_number(&self) -> i32 {
        *self
    }
} 
```

我们可以尝试执行与之前相同的转换：

```rust
#use std::future::Future;
trait AsyncTrait {
    fn get_number(&self) -> impl Future<Output = i32>;
}

impl AsyncTrait for i32 {
    fn get_number(&self) -> impl Future<Output = i32> {
        async {
            *self
        }
    }
} 
```

不过，这存在一些问题。

首先，`get_number` 的返回类型 (return type) 应该使用什么类型？编译器需要为它找到某个具体的类型，即使我们看不到该类型。

可以想象 `AsyncTrait::get_number` 有一个具体的返回类型。如果我们这样做，解糖效果将如下所示：

```rust
#use std::future::Future;
type SecretGetNumberReturn = impl Future<Output = i32>;

trait AsyncTrait {
    fn get_number(&self) -> SecretGetNumberReturn;
}

impl AsyncTrait for i32 {
    fn get_number(&self) -> SecretGetNumberReturn {
        async {
            *self
        }
    }
} 
```

这里，我们为类型名称添加了前缀 `Secret`，以表明这不是一个可命名的类型；它将在编译器内部生成。

一个明显的问题是，目前还不允许使用 `type SecretGetNumberReturn = impl Future<Output = i32>`。

实现这个功能需要还在开发中 `#![feature(type_alias_impl_trait)]` 或简称
TAIT，尽管它不稳定，但此时似乎工作得相当好，我们可以在编译器内部使用它来对异步函数进行解糖。

更微妙的问题是，每个 `async { ... }` 块都有一个不同的、无法命名的类型。

因此，为整个程序选择一个具体的 `impl Future` 类型，导致只能实现一次 `AsyncTrait`。这违背了 traits 的意图！

因此，我们需要将 `SecretGetNumberReturn` 作为 trait 的关联类型，如下所示[^fn:secret-prefix]：

[^fn:secret-prefix]: 我在类型名称前加上了前缀 `Secret`，以表示这些类型将是由编译器生成的不可命名类型。从本质上讲，编译器已经对所有返回
`-> impl Trait` 的函数执行了相同的操作。

```rust
#use std::future::Future;
trait AsyncTrait {
    type SecretGetNumberReturn: Future<Output = i32>;

    fn get_number(&self) -> Self::SecretGetNumberReturn;
}

impl AsyncTrait for i32 {
    type SecretGetNumberReturn: Future<Output = i32> = impl Future<Output = i32>;

    fn get_number(&self) -> Self::SecretGetNumberReturn {
        async {
            *self
        }
    }
} 
```

因此，我们已经基本实现了这一目标，但仍然存在一个问题。

目前能做到在给定 trait 的每一个类型实现中，返回一个特定的 Future 。问题是，返回的 Future 通常需要向 `self`
借用（将在 [下一节](#gat-desugaring) 中解释原因），这意味着返回的 Future 的类型将需要生命周期。要实现这一点，还需要泛型关联类型 ([GATs])。

[GATs]: https://rust-lang.github.io/generic-associated-types-initiative/

<a name="gat-desugaring"></a>

## 第 2 步：使用 GATs 解语法糖

我们在上一节忽略的一个细节是在 `foo` 函数体中的 `async { *self } ` 。

与编译闭包非常相似，编译器将该 async 块转换为一个 `struct` 和该 `struct` 的一个 `Future` 的 impl。它看起来像这样：

```rust
#use std::future::Future;
#use std::pin::Pin;
#use std::task::{Context, Poll};
trait AsyncTrait {
    type SecretGetNumberReturn: Future<Output = i32>;

    fn get_number(&self) -> Self::SecretGetNumberReturn;
}

struct SecretGetNumberForI32Future {
    this: &i32,
}

impl Future for SecretGetNumberForI32Future {
    type Output = i32;

    fn poll(self: Pin<&mut Self>, _context: &mut Context<'_>) -> Poll<i32> {
        Poll::Ready(*self.as_ref().this)
    }
}

impl AsyncTrait for i32 {
    type SecretGetNumberReturn = SecretGetNumberForI32Future;

    fn get_number(&self) -> Self::SecretGetNumberReturn {
        SecretGetNumberForI32Future {
            this: self,
        }
    }
} 
```

基本上是将 `self` 参数复制到 `SecretGetNumberForI32Future` 结构体中，但将其重命名为 `this`，因此它是一个合法的字段名称。

然后，生成一个 `poll` 函数，该函数取消对 `this` 的引用并返回结果整数。

但是有一个很大的问题，那就是在结构体中有一个引用，从而结构体需要一个生命周期。

因此，为了正确执行此操作，需要在 `SecretGetNumberForI32Future` 中添加一个生命周期参数，并将其延伸到所有其他使用它的位置：

```rust
#use std::future::Future;
#use std::pin::Pin;
#use std::task::{Context, Poll};
trait AsyncTrait {
    type SecretGetNumberReturn<'a>: Future<Output = i32> + 'a where Self: 'a;

    fn get_number(&self) -> Self::SecretGetNumberReturn<'_>;
}

struct SecretGetNumberForI32Future<'a> {
    this: &'a i32,
}

impl<'a> Future for SecretGetNumberForI32Future<'a> {
    type Output = i32;

    fn poll(self: Pin<&mut Self>, _context: &mut Context<'_>) -> Poll<i32> {
        Poll::Ready(*self.as_ref().this)
    }
}

impl AsyncTrait for i32 {
    type SecretGetNumberReturn<'a> = SecretGetNumberForI32Future<'a>;

    fn get_number(&self) -> Self::SecretGetNumberReturn<'_> {
        SecretGetNumberForI32Future {
            this: self,
        }
    }
} 
```

GATs 在 Rust 1.65 中已经稳定，以上代码可以工作。

<a name="dynamic-traits"></a>

# 使用 `dyn*` 的动态 traits

## 第 3 步：`dyn*`

到目前为止，我们所拥有的足以在静态分发上下文中的 trait 中使用异步函数。例如，可以编写以下代码：

```rust,ignore
async fn print_the_number_static_dispatch(from: impl AsyncTrait) {
    let number = from.get_number().await;
    println!("The number is {number}");
} 
```

然而，这要求我们在编译时知道 `from` 的具体类型。在例子中有 `from: Arc<dyn AsyncTrait>`，它允许在运行时更改 `dyn AsyncTrait` 背后的具体类型。

其中一个主要挑战是如何存储 `get_number()` 返回的 Future。

在静态分发的情况下，编译器只是将 Future 存储在 `print_the_number_static_dispatch` 的栈帧中。

但这需要编译器知道 Future 的大小，这在 `dyn AsyncTrait` 的情况下是不可能知道的。

这个问题普遍存在于 `dyn Trait` 中。

在非 `async` 的情况下，解决这个问题的方法是使代码不直接指向 `dyn Trait` 对象。

而是让代码通过 `&dyn Trait` 和 `Box<dyn Trait>` 等指针与 `dyn Trait` 交互。

这些指针是胖指针，由一对指向对象本身的指针和一个指向 `Trait` 对象的 vtable 的指针组成。

这里的关键是使用指针而不是原始的 trait 对象来赋予类型已知的大小。

因此，对于 trait 中的异步函数，我们需要一种将返回的 Future 强制转换为指针的方法。可以尝试像这样返回 `&dyn Future`：

```rust,ignore
trait AsyncTrait {
    type SecretGetNumberReturn<'a>: Future<Output = i32> + 'a;

    // return value is a reference to a SecretGetNumberReturn
    fn get_number(&self) -> &dyn Future<Output = i32>;
} 
```

当这样实现时，会得到如下结果：

```rust,ignore
impl AsyncTrait for i32 {
    type SecretGetNumberReturn<'a> = SecretGetNumberForI32Future<'a>;

    fn get_number(&self) -> &dyn Future<Output = i32> {
        &SecretGetNumberForI32Future {
            this: self,
        }
    }
} 
```

正如你可能预料到的那样，这段代码有问题。最重要的是，它非常不健全（尽管如果你尝试编写这段代码，编译器会给你一个错误）。

问题在于返回临时变量的引用。当 `get_number` 函数返回时，临时引用将被丢弃，这意味着引用已超过其引用对象的生命周期。

另一个问题是，现在 trait 需要知道它返回的是哪种指针。对于现有的 dyn-safe trait，你所编写 trait 的 impl，无论它是用作
`&dyn Trait`、`Box<dyn Trait>`、`Arc<dyn trait >`，还是一些尚未发明的指针类型，该 impl 都可以工作。

我们的计划是通过引入一个新的 `dyn*` 语法来解决这个问题，它是一个指向与其底层指针类型无关的 `dyn Trait` 的指针。[^fn:unsized-rvalues]

[^fn:unsized-rvalues]: 解决此问题的另一种可能的方法是使用未知大小的右值 ([unsized Rvalues])。未知大小的右值允许使用 `alloca`
   将一些未知大小的类型的值存储在堆栈上。遗憾的是，在 `async fn` 中支持 `alloca` 往好了说不是小事，往坏了说是不可能的。

[unsized Rvalues]: https://github.com/rust-lang/rfcs/blob/master/text/1909-unsized-rvalues.md

我将在这里总结 `dyn*` 的工作方式，但要了解更多信息，请参考 Niko 的帖子 *[dyn*：Can we make dyn Size?][niko-dyn]* [^niko-dyn-zh]。

[niko-dyn]: https://smallcultfollowing.com/babysteps//blog/2022/03/29/dyn-can-we-make-dyn-sized/

[^niko-dyn-zh]: 译者注：我翻译了这篇（和这个系列），见
[dyn async traits 系列之“dyn* 让 dyn 有大小”](https://zjp-cn.github.io/translation/dyn-async-traits/2022-03-29-dyn-can-we-make-dyn-sized.html)

虽然现在必须明确是否有 `&dyn Trait`、`Box<dyn Trait>` 或其他东西，但 `dyn* Trait` 将能够表示其中的任何一个。

`dyn* Trait` 仍然是指针大小的对象和指向 vtable 的指针。指针大小的对象通常实际上是指向 trait
对象数据的指针，但在对象本身是指针大小的情况下，我们可以选择内联存储对象本身。

我们可以控制如何通过辅助 traits 创建 `dyn* Trait` 对象，以便将对象强制转换为指针并返回。

trait 将如下所示，我们将使用它作为示例：

```rust
#use std::pin::Pin;
trait IntoDynPointer {
    type Raw: PointerSized;

    fn into_dyn(self) -> Self::Raw;
    fn from_dyn(raw: Self::Raw) -> Self;
    fn from_dyn_pin_mut(raw: Pin<Self::Raw>) -> Pin<&'static mut Self>;
}

unsafe trait PointerSized {}
unsafe impl PointerSized for usize {}
unsafe impl<T> PointerSized for *const T {}
unsafe impl<T> PointerSized for Box<T> {}
// etc. 
```

当编译器看到 `x as dyn* Future<Output = ()>` 强制转换时，它会被解糖成类似 `(mem::transmute<_, usize>(x.into_raw()), VTABLE_FOR_X_AS_FUTURE)` 的形式。

如果 `x` 实现了 `Future<Output = ()>` 和 `IntoDynPointer`，则可以这样做。

编译器还将生成如下所示的 vtable：

```rust,ignore
const VTABLE_FOR_SECRET_NUMBER_RETURN_AS_FUTURE = FutureVtable {
    poll_fn: |this, cx| {
        let this: <SecretNumberReturn as IntoDynPointer>::Raw =
            unsafe { mem::transmute(this) };
        let this = <SecretNumberReturn as IntoDynPointer>::from_dyn_pin_mut(this);
        this.poll(cx)
    },
    drop_fn: |this| {
        let this: <SecretNumberReturn as IntoDynPointer>::Raw =
            unsafe { mem::transmute(this) };
        SecretNumberReturn::from_dyn(this);
    }
} 
```

`IntoDynPointer` 的实现将决定 Future 是被放入 Box 还是使用其他分配策略。以下是默认 Box 策略下的实现：

```rust,ignore
impl IntoDynPointer for SecretNumberReturn {
    type Raw = *const ();

    fn into_dyn(self) -> Self::Raw {
        Box::into_raw(Box::new(self)) as *const _
    }

    fn from_dyn(raw: Self::Raw) -> Self {
        unsafe { *Box::from_raw(raw as *mut _) }
    }

    fn from_dyn_pin_mut(raw: Pin<Self::Raw>) -> Pin<&'static mut Self> {
        unsafe { mem::transmute(raw) }
    }
} 
```

到目前为止，有几件事需要注意。

首先，我们添加了一个新的 `unsafe trait PointerSized`。虽然它被标记为 unsafe，但并不是真的不安全。

由于编译器知道实际类型，因此它可以检查声称为指针大小的类型是否实际为指针大小。（但无法推断指针大小，因为类型布局在编译中计算得太晚。）

其次，我们还需要 `from_dyn` 的其他版本，如 `from_ref`、`from_ref_mut`、`from_pin` 等，以支持 trait 中不同方法可能具有的不同 self 类型。

最后，编译器将应该能够给编译器所生成的 Future（即 `async` 块的结果和 `async fn` 的返回类型）生成一个 `IntoDynPointer` 
impl，就像上面展示的那样，尽管在不可能或不希望 boxing 的情况下将需要一个备用方案。

接下来，我们需要创建 `dyn*` 对象，并为 `dyn*` 生成一个 `Future` 的 impl。

编译器将把一个 `dyn* Future` 表示为某些数据和一个 vtable 指针。[^fn:vtable-inline]

[^fn:vtable-inline]: 以内联方式存储 vtable 有意义吗？虽然它是大小已知的，我们将避免间接性，但是，太大 trait 的 `dyn*` 类型会变得更大，而且由于
   `dyn*` 是按值传递的，这将导致花费大量时间来复制内存。

我们将使用下面的结构体来表示：

```rust,ignore
struct DynStarFuture {
    data: *const (),
    vtable: &'static FutureVtable,
} 
```

现在我们需要为 `DynStarFuture` 生成一个 `Future` 的 impl，从而有 `dyn* Future: Future`：

```rust,ignore
impl Future for DynStarFuture {
    type Output = i32;

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        (self.vtable.poll_fn)(self.data, cx)
    }
} 
```

我们还需要实现 `Drop`：

```rust,ignore
impl Drop for DynStarFuture {
    fn drop(&mut self) {
        (self.vtable.drop_fn)(self.data)
    }
} 
```

最后，生成用于 `dyn AsyncTrait` 的转换器 (adapter) trait。转换器 trait （我们将其称为 `DynAsyncTrait`）与 `AsyncTrait`
相同，但将 `impl Future` 类型替换为 `dyn* Future`：[^fn:trait-alias]

[^fn:trait-alias]: 有趣的是，这实际上并不是一个新的 trait；它等同于 `AsyncTrait<SecretGetNumberReturn = dyn* Future<Output = i32>`。

```rust,ignore
trait DynAsyncTrait {
    fn get_number(&self) -> dyn* Future<Output = i32>;
} 
```

我们可以为任何实现了 `AsyncTrait` 的 `T` 编写一个 blanket impl，以允许在 `dyn` 上下文中使用任何实现：

```rust,ignore
impl<T: AsyncTrait> DynAsyncTrait for T {
    fn get_number(&self) -> dyn* Future<Output = i32> {
        let future = <Self as Future>::get_number(self);
        DynStarFuture {
            data: mem::transmute(Self::into_raw()),
            vtable: &VTABLE_FOR_SECRET_NUMBER_RETURN_AS_FUTURE,
        }
    }
} 
```

从而现在我们终于能够重写 `print_the_number` 以动态分发到一个异步函数。作为提醒，以下是原始函数：

```rust,ignore
async fn print_the_number(from: Arc<dyn AsyncTrait>) {
    let number = from.get_number().await;
    println!("The number is {number}");
} 
```

然后，编译器会将其重写为类似以下内容：

```rust,ignore
async fn print_the_number(from: Arc<dyn DynAsyncTrait>) {
    let number_future = from.get_number();
    let number = number_future.await;
    println!("The number is {number}");
} 
```

这两个函数之间唯一的实质性变化是 `from` 的类型从 `Arc<dyn AsyncTrait>` 变为 `Arc<dyn DynAsyncTrait>`。

神奇之处实际上发生在调用处。因为 `DynAsyncTrait` 现在是 dyn-safe，并且对任何实现了 `AsyncTrait` 的类型都有一个 blanket 的
`DynAsyncTrait` 实现，所以编译器可以强制将实现了 `AsyncTrait` 的东西（在我们的例子中是 `i32`）变成一个 `DynAsyncTrait`，就像任何其他 dyn trait 一样。

# 结论

刚刚介绍了 rustc 在 dyn trait 对象中实现异步函数所需要进行的一组相当长的转换。

首先展示了如何使用泛型关联类型支持静态分发 trait 中的异步函数。

接下来，展示了如何使用 `dyn*` 概念使这些 trait dyn-safe。

如果你想看到整个转换后的程序，请查看这个 Rust [playground] 链接。

[playground]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=8f015eed4068a6cc6aabb77ed82ef3b0

请注意，这些想法仍在开发中，我在这篇文章中可能写了一些错误的东西。

我的目标是在一个地方总结所有挑战，并展示一些已被提议的解决方案如何应对这些挑战。

关于 `dyn*` 的设计的后期部分仍然是相当新的，可能会有重大的变化，但我们似乎已经有了足够丰富的东西，可以开始试验实现了。

感谢 [Nick Cameron](http://ncameron.org/) 对这篇文章早期草稿的反馈。
