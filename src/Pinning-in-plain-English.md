# 用通俗易懂的英语解释 pinning

![](https://blog.schichler.dev/_next/image?url=https%3A%2F%2Fcdn.hashnode.com%2Fres%2Fhashnode%2Fimage%2Fupload%2Fv1637755591169%2FOMsBWv8Bc.png%3Fw%3D1600%26h%3D840%26fit%3Dcrop%26crop%3Dentropy%26auto%3Dcompress%2Cformat%26format%3Dwebp&w=3840&q=75)

由 [Tamme Schichler](https://hashnode.com/@Tamme) 所写，发布于 2021 年 11 月 25 日，预计阅读完需要 15 分钟。
[点击查看原文][original-post]，以及原文的[讨论][discussion]。

[original-post]: https://blog.schichler.dev/pinning-in-plain-english-ckwdq3pd0065zwks10raohh85
[discussion]: https://users.rust-lang.org/t/pinning-in-plain-english/67992

 > 标题图像显示在木质砧板上放着一个橙色的甜椒。
 > 
 > 该甜椒已经镜像映射到了图像的右半部分，那显然仍然是同一种水果，
 > 但镜像在很大程度上只能与根本不同的生物和谐共生，
 > 因为它的许多复杂一些的分子性质都是手性的[^chiral]。
 >
 > 这听起来令人耳目一新。

[^chiral]: 
as many of its more complex molecules are chiral \
译者注：“手性”指一个物体不能与其镜像相重合。如我们的双手，左手与互成镜像的右手不重合。

Rust 中的 pinning 是一种强大而非常方便的模式，但在我看来，它没有在更广泛的生态系统中得到足够的支持。

一种普遍的看法是它很难理解，并且 [pin 模块的文档](https://doc.rust-lang.org/stable/core/pin/index.html) 
令人困惑。（就我个人而言，我认为它可以很好地作为参考资料来回答有关极端例子 (edge cases) 
的问题，但它难读懂，不一定是很好的入门文本。）

我通过写这篇文章试着让这一功能更易于使用，希望能激励更多的开发人员让他们的 crate 知道 pinning 在哪里会有帮助。

---

**许可和翻译**

请参阅本帖末尾之后的内容。简而言之，这篇文章除了明确标记为这样的代码引用，
整体上是根据 [CC by-NC-SA2.0](https://creativecommons.org/licenses/by-nc-sa/2.0/)
授权的。未标记为引用的代码片段和代码块也在 
[CC01.0](https://creativecommons.org/publicdomain/zero/1.0/) 下获得许可。

---

好了，让我们来看看实际内容吧！

请注意，我在不同位置添加了 Rust 文档的链接。
这些都是“进一步的阅读材料”，所以我建议在研究它们之前阅读完这整篇文章。
通过这种方式，（希望）它们会更容易理解。

# 问题

在 Rust 中，只要任何实例 (instance) 的大小在编译时是已知的，那么就认为该实例是一般情况下都是可移动的。
这意味着任何人拥有实例或对实例 `&mut`
进行独占引用之后，就可以将其非结构化数据（即数据直接包含的字节）复制到不同的内存地址，
然后以其他方式重新使用旧位置或使用移动后的实例时，不会有任何中断。

与 C# 或 JavaScript 不同，Rust 的这一点很重要：引用、指针和地址可以非常容易地在彼此之间进行转换，并且
**地址（即在某种程度上的指针）是可以用于计算偏移量的普通数字**。
例如，这使得创建非常高效的泛型集合成为可能，因为它们总是可以在同一次分配中“按值” (by-value) 
存储实例。如果空间不足，无法添加新项，则 Rust 会隐式重新分配存储空间，可能会将内容移动到内存中的新位置。

然而，与 C++ (
[1](https://en.cppreference.com/w/cpp/language/copy_constructor), 
[2](https://en.cppreference.com/w/cpp/language/move_constructor), 
[3](https://en.cppreference.com/w/cpp/language/copy_assignment), 
[4](https://en.cppreference.com/w/cpp/language/move_assignment)
)
不同的是，Rust **没有内置机制来更新实例所有者不直接知道的指针和地址，或者完全阻止特定类型的移动**：
你不能重载或删除普通赋值操作符 `=`，也不存在当实例被隐式移动时可以调用的
[move](https://en.cppreference.com/w/cpp/language/move_constructor) 或
[copy constructor](https://en.cppreference.com/w/cpp/language/copy_constructor) 的概念。

Rust 的引用 （`&` 或 `&mut`） 是轻量级指针，虽然有一些别名限制 (aliasing restrictions)，
但它们始终有效[^valid]，而且不能从其他地方自动更新。这反过来又会阻止借用期间**完全**重新分配其目标（但独占引用 
`&mut T` 的代码仍然可以自由地 [`swap`](https://doc.rust-lang.org/stable/core/mem/fn.swap.html) 实例）。

[^valid]: 更具体地说：Rust 的引用总是保证是可解引用的 (dereferenceable)
，这允许编译器在许多情况下提前加载和缓存它们指向的值。C++ 提供了相同的可解引用保证，但默认情况下允许可变别名。

所有这些都使得编写依赖于两次调用实例之间的已知位置的内存安全 API
变得棘手，因为**通过所有权或借用来持续拥有它将是不灵活的**，通常也很不方便，并且**使用间接句柄 (handle)
对于许多低级组件来说效率太低**。

# pinning 摘要

Rust 选择显式更改引用的可见类型，（而且显式使用 API）以防止意外移出“不安全”代码。

虽然有一些例外，但是 pinning 的**默认**断言是：如果 `Pin<&T>` 是可访问的，那么 `T`
的实例在 droped 之前，其地址一直可用。

可以在一定程度上削弱这一点，但除了在 `T: Unpin` 的时候，只能以某种方式使用 `unsafe` 代码才能做到。

然而当一个类型实现了 [`Unpin`](https://doc.rust-lang.org/stable/core/marker/trait.Unpin.html)
，pin 它的实例是[没有意义的](https://doc.rust-lang.org/stable/core/pin/struct.Pin.html#impl-DerefMut)。

**为简单起见，除非另有说明，否则本文中的 `T` 隐含为 `!Unpin` （pinned 类型）**。这意味着，如果你试图 pin 一个
`f64` 或 所有成员都是 `Unpin` 的结构体，那么有些句子就不再正确的。往下阅读时请牢记这一点。

# pin vs pinned

每当在 Rust 中发生 pinning 时，都会涉及两个部分[^two-components]：

* pin 类型表明 “值不能（通过 safe Rust）被移走”。这通常被称为 “pinning pointer” 。
  
  pin 类型通常是用 `Unpin` 描述的，但这与它们的功能无关。相反，此规则仅通过 pin API 的约束来实施。[^Unpin-API]

* 被 pinned 的值不能被 API 的使用者移动。
  
  这通常说的是一个具体的值，而不是泛指某个类型的所有成员，因为 pinned 不是任何类型的固有属性 
  (inherent property) 。

[^two-components]: 译者注：pin type （或者说 pins） 和 pinned value。

[^Unpin-API]: 译者注：这句话指涉及 pin 的 API 通常使用 `Unpin` trait 来描述（因为只有 
[`Unpin` trait](https://doc.rust-lang.org/std/marker/trait.Unpin.html) 和
[`Pin` struct](https://doc.rust-lang.org/std/pin/struct.Pin.html)）。

# pin 类型通常是复合形式

例如，看看 [`Box::pin`](https://doc.rust-lang.org/stable/alloc/boxed/struct.Box.html#method.pin) 的签名：

```rust, ignore
// Citation.
pub fn pin(x: T) -> Pin<Box<T>> { … }
```

此函数将 `T` 类型的值 `x` pin 在一个 `Pin<Box<_>>` 类型的智能指针后面。

可以把 `Pin<Box<_>` 理解为 pinning `Box` （pin 住值的 Box），而不是放入 `Pin` 内的 `Box`。 而且 `Pin<Box<_>`
**不能安全**取出值（[除非 `Box<T>` 中 `T` 满足 `T: Unpin` 时使用 `into_inner` 取出][unless `T: Unpin`]）。

[unless `T: Unpin`]: https://doc.rust-lang.org/stable/core/pin/struct.Pin.html#method.into_inner

作者附注：普通的 `Pin<Box<_>>` 是 pinn**ing** 而不是 pinn**ed** [^pinning-pinned]。 实际上， `Pin<Box<_>`
总是 `Unpin` （因为 `Box<_>` 总是 `Unpin`）[^Pin<Box<_>>-not-pinned]，所以把 Box 和 `Pin<Box<_>`
自己 pin 住没有意义。

[^pinning-pinned]: 译者注：我的理解是 `Pin<Box<_>>` 是把值/实例 pin 住 ——
主动语态补全为 Box is pinning the stuff inside of it，而不是把 
`Box`/`Pin<Box<_>>` pin 住 —— 如果被动语态补全，即 `Pin<Box<_>>` 不是描述 Box itself is pinned。

[^Pin<Box<_>>-not-pinned]: 译者注：由
[`Pin` 实现的 `Unpin`](https://doc.rust-lang.org/std/pin/struct.Pin.html#impl-Unpin)
`impl<P: Unpin> Unpin for Pin<P>` 和 
[`Box` 实现的 `Unpin`](https://doc.rust-lang.org/std/boxed/struct.Box.html#impl-Unpin) 
`impl<T: ?Sized, A: Allocator + 'static> Unpin for Box<T, A>` 即可知。

对于所有标准的智能指针和 Rust 引用类型，包括在 pin 住 `Unpin` 类型的情况下，都适用于下面的描述：

|Rust|English| （译者注） |
|----|-------|--|
|`Pin<Box<_>>`|"pinning `Box`"|把值/实例 pin 住的 `Box`|
|`Pin<Rc<_>>`|"pinning 	`Rc`"|把值/实例 pin 住的 `Rc`|
|`Pin<Arc<_>>`|"pinning `Arc`"|把值/实例 pin 住的 `Arc` |
|`Pin<&_>`|"pinning (shared) reference"|把值/实例 pin 住的共享引用|
|`Pin<&mut _>`|"pinning (exclusive) reference"|把值/实例 pin 住的独占引用|

我经常在默读的时候把 "pinning `Box`" 精简为 "`Pin`-`Box`"，这样大声朗读的时候你也应该听懂。

# `Unpin` 是一种 auto trait

实际上 Rust 中很少有类型是 `!Unpin` 的。由于 
[`Unpin`](https://doc.rust-lang.org/stable/core/marker/trait.Unpin.html) 是
[`auto trait`][`auto trait`]，所以所有成员已经是 `Unpin` 的复合类型（结构体、枚举体、unions）都会实现
`Unpin`。它对于几乎所有的内置类型都是自动实现的，**对于[指针][pointers]也是显式实现的**！这意味着像
[`NonNull`][`NonNull`] 这样的指针包装器 (pointer wrappers) 也是 `Unpin` 的！

[`auto trait`]: https://doc.rust-lang.org/beta/unstable-book/language-features/auto-traits.html
[pointers]: https://doc.rust-lang.org/stable/std/primitive.pointer.html
[`NonNull`]: https://doc.rust-lang.org/stable/core/ptr/struct.NonNull.html

In fact, the only type that is explicitly `!Unpin` as of stable Rust 1.56, including in custom crates, is [`core::marker::PhantomPinned`](https://doc.rust-lang.org/stable/core/marker/struct.PhantomPinned.html), a marker you can use as member type to make your custom type `!Unpin`.

事实上，从 stable Rust 1.56 开始，包括自定义 crates 在内，唯一显式为 `!Unpin` 的类型是
[`core::marker::PhantomPinned`]，这是一个可以作为成员类型的标记 (marker)，用于使你的自定义类型为
`!Unpin` 的[^!Unpin-PhantomPinned]。

[`core::marker::PhantomPinned`]: https://doc.rust-lang.org/stable/core/marker/struct.PhantomPinned.html
[^!Unpin-PhantomPinned]: 译者注：也就是说，目前 Rust 的 `!Unpin` 类型只有 PhantomPinned 类型及其衍生类型。

你可以在此处查看实现了 `Unpin` 的完整列表：
[`Unpin`\#implementors](https://doc.rust-lang.org/stable/core/marker/trait.Unpin.html#implementors)。

# （几乎所有的）值一开始都不会被 pinned

即使对于不是 `Unpin` 的 `T`，在 stack 上的 `T` 或通过 `&mut T` 直接访问的 `T` 的一般实例也不是 pinned 
。这也意味着它可以在不运行析构函数 (destructor) 的情况下被丢弃，例如可以把它作为参数来调用
[`mem::forget`](https://doc.rust-lang.org/stable/core/mem/fn.forget.html)。

`T` 的实例仅在传递给 `Box::pin` 这样的函数时才会被 pinned
，该函数提供[这些保证][guarantees]。（而且理想情况下，根据需要以某种方式公开 `Pin<&T>`）。

[guarantees]: https://doc.rust-lang.org/stable/core/pin/index.html#drop-guarantee

# `Pin<_>` 的功能

`Box<T>` 和 `Pin<Box<T>` 的唯一区别是：

* `Pin<Box<T>>` 从不暴露 `&mut T` 或直接的 `T`，因此，将值移动 (move) 到其他地方是不可能的。

* `Pin<Box<T>>` 暴露 `Pin<&T>` 和 `Pin<&mut T>`。这称为针对存储的值的 “pin projection” （pin 投影）。
  
  通过访问这些 pinning 引用，你可以在需要 `self: Pin<&Self>` 或 `self: Pin<&mut Self>` 
  的值上调用方法，还可以调用具有类似参数类型的关联函数。

普通的 `&T` 值引用可以像以前一样访问，也可以通过 `Pin<&T>` 的解引用访问，而无需考虑 `T` 是否为 `Unpin`。

所有实现 [`Deref<Target = T>`](https://doc.rust-lang.org/stable/core/ops/trait.Deref.html)
的智能指针和引用（以及对于可变访问的 
[`DerefMut`](https://doc.rust-lang.org/stable/core/ops/trait.DerefMut.html)
）在 pinning 时的解引用功能别无二致。

为了让文章剩余的部分保持易读性：

定义以下内容（仅在本文中有效）：

|Shorthand|implied constraint|English|（译者注）|
|---------|------------------|-------|--|
|`T`|**not** `T: Unpin`|"\[arbitrary\] type" / "\[arbitrary\] value"| 任意类型 / 任意值|
|a pinned `T`|**not** `T: Unpin`|"a pinned value"|被 pinned 住的值|
|`P`|`P: Deref<Target = T>` 或 `P: DerefMut`|"pointer"| 指针|
|`Pin<P>`|`P: Deref<Target = T>` 或 `P: DerefMut`|"pinning pointer"| pin 住值的指针|

# pinning 是一个仅存在于编译时的概念

`Pin<P>` 是对其单个成员 [`#[repr(transparent)]`][repr-transparent] 的包装，是 `P` 类型的私有字段。

[repr-transparent]: https://doc.rust-lang.org/stable/reference/type-layout.html#the-transparent-representation

换句话说：**`Pin<Box<T>>` 与 `Box<T>` 具有完全相同的运行时表现。**

反过来，由于 `Box<T>` 又具有与其派生的 `&T` 有相同的运行时表现，因此得到 `&T` 的 `Pin<Box<T>` 
解引用是一个标识函数，这个函数返回与它进入函数运行后得到完全相同的值，这意味着（但只有优化之后！）根本不需要运行时操作。

将指针或容器转换为 pinning 版本之后操作同样自由 (free action) 。

这种自由是可能发生的，因为一旦涉及到引用，Rust 中的移动 (move)
就已经相当显式：底层赋值可能隐藏在另一个方法中，但是 Rust 有告知之后才移动 heap 上的实例的机制。与 
C# 不同，在 C# 中，pinning 是[集成到 GC API 中][C#-GC]的运行时操作。

[C#-GC]: https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.gchandletype?view=net-6.0

唯一的例外是 [`Copy`][`Copy`] 类型，它的 `&` 引用的实例可以隐式地使用新地址创建副本。`Copy` trait 不是
[`auto`][`auto`] 的，这意味着它必须显式 derive 给每种隐式可创建副本的类型。

作者附注：对于具有重要地位的类型，请不要实现 `Copy` 。实现 [`Clone`][`Clone`] 通常就足够了。
`Copy` 非常方便于经常通过值传递的不可变实例，因此即使底层类型实现了 `Copy` 的
[`Cell`][`Cell`]，也不会实现 `Copy`。

[`Copy`]: https://doc.rust-lang.org/stable/core/marker/trait.Copy.html
[`auto`]: https://doc.rust-lang.org/unstable-book/language-features/auto-traits.html
[`Clone`]: https://doc.rust-lang.org/stable/core/clone/trait.Clone.html
[`Cell`]: https://doc.rust-lang.org/stable/core/cell/struct.Cell.html

# pinning 是一个视角问题

pin 住值是通过如下方式做到的：使 safe Rust 在没有 drop 实例的情况下不能移动实例或释放其内存。给
`Pin<&T>` 或 `Pin<&mut T>` 提供 safe Rust 访问权限的 pin 对这种做法做出正式而明确的宣告，尤其对 `T` 的无关实现。

但是，由于被 pinned 的实例本身的类型不变，它在最初实现 pin 类型的模块中可能仍看起来是 unpinned 的样子。

**`Pin<_>` 只能通过封装来隐藏正常的可变接口 (mutable API) ，但不能将这个可变接口完全擦除。**

对于具有[内部可变][interior mutability]的 pin 类型或包含 non-pinning 字段所组成的 pin 
类型来说尤其如此，因为它们通常会在 non-pinning API 和 pinning API 之间共享大部分实现。

[interior mutability]: https://doc.rust-lang.org/core/cell/index.html 

可能在此之前已经向外呈现衍生的 `Pin<&T>`，但此类模块内的安全代码（目前）经常处理 `&mut T` 
，在这种情况下必须格外小心，以避免不安全的移动。随着有更多的 pinning 指针和集合 (collections) 可用，如果
Rust 使包装在 `Pin<_>` 中的自定义类型更容易添加方法，上述情况将来就可能有所改观，

# 集合可以 pin 住其元素

这在 `Box<T>` 或 `Pin<Box<T>>` 中表现得最为明显，其中 `Box` 充当 `T` 的单元素 (1-item)
集合。这对于 `Arc` 和 `Rc` 也是如此。

因此，还要注意是各元素为类型 `T` 的集合 `C` 。

同样可以使用 `Pin` 创建一个新类型的 `Pin<C>`，通过既不暴露 `&mut T` 也不暴露 `T`
，`Pin<C>` 和用于 pinning 智能指针作用相似。

定义以下内容（仅在本文中有效）：

|Shorthand|implied constraint|English| （译者注）|
|---------|------------------|-------|--|
|`C`|owns instances of type `T`|"collection"|拥有 `T` 实例的所有权的集合|
|`Pin<C>`|owns instances of type `T`|"pinning collection"|拥有 `T` 实例的所有权、pin 住其元素的集合|

标准库对此没有提供太多的功能，因为常规的集合比智能指针种类和功能要丰富得多。你如果决定编写一个支持 pinning
的集合，那么将不得不自己实现大部分 API，并且从 Rust 1.56 开始，可能必须通过 traits 提供扩展方法来无缝地调用它。

如果集合 `C` 允许其 pinned 形式（`&Pin<C>` 或 `&mut Pin<C>`）投射到 `Pin<&T>` 或 `Pin<&mut T>`
，那么这个集合 `C` “可以 pin 住其元素”。

集合也可能**固有地** (inherently) pin 住元素，在这种情况下，它的功能类似于 `Pin<C>`
，但类型上没有出现 `Pin` 。我们不会在此直接看到这种集合。

# `Pin<P>` vs. `Pin<C>` vs. `T`

普通的 non-pinning 指针和集合的功能应该足够清楚了，所以我只在此比较它们和 `T` 在 pinning 和 pinned 时 API
的不同之处：

||`Pin<P>`|`Pin<C>`|`T` behind `Pin<&T>` or `Pin<&mut T>` |
|--|--------|--------|---|
|English|"pinning pointer"|"pinning collection"|"pinned value"|
|（译者注）|pin 住值的指针|pin 住元素的集合|被 pinned 住的值|
|`: Unpin`|几乎总是|很多时候是|实践中几乎不是，因为 pin 住 `T: Unpin` 是没意义的|
|无 pinning 的 APIs 可以访问| 可以访问 `Pin<&T>` 或者 `Pin<&mut T>`|视情况而定，但通常与 `Pin<P>` 类似|由参数为 `Pin<&T>` 或 `Pin<&mut T>` 的函数决定|
|pinning 之后 APIs 不能访问|不能访问 `&mut T`、unwrap `T` |视情况而定，但通常不能访问 `&mut T`、remove T、任何涉及重新分配 `T` 的事|由参数为 `&mut T` 的函数决定|
|未变的 APIs （例子）|访问 `&T`|访问`&T`、drop `T` in place|由参数为 `&T` 的函数决定|
|`: Clone`|通常是 |有可能是 |当 `T: Clone`[^Clone-T] 是|视情况而定|

[^Clone-T]: 根据此表格，则可以实现这样的功能 `pub fn clone_unpinning(this: &Pin<Self>) -> Self { … }`
。但如果 `T: Clone`，那么很可能 `T` 也是 `Unpin` 的，这使得 pin 住 `T`
几乎毫无用处。有关更有用的、可以 clone 被 pinned 实例的、有意义的实现，请参阅本文末尾的内容。

# 哪些函数需要 `Pin<&T>`

`Pin<&T>` 和 `Pin<&mut T>` 的用法各不相同，但大多数情况下有三大类：

## 避免引用计数

> Avoiding reference-counting

如果指向实例的智能指针经常被复制，但很少被访问，并且因为其 lifetime 不能被静态约束
(statically constrained) 而无法使用引用，那么将运行时成本从 clone 指针转移到检查访问的有效性上是有意义的。在这种情况下，智能指针被替换成了可复制的句柄 (handles) 。

句柄怎么知道他们的目标什么时候消失了？pin 住 `T` 将明确宣布**该特定实例**将运行 `<T as drop>::drop` 
，因此这将有机会通知寿命更长的注册表 (longer-lived registry) 。

这也让句柄不能显式删除成为可能，比如它们可直接通过 [`bumpalo`][`bumpalo`]
这样的 arena allocator 存储下来。你可以在我的 [`lignin`][`lignin`] crate
中看到这个模式的示例，它以此支持 (V)DOM 回调。

[`bumpalo`]: https://crates.io/crates/bumpalo
[`lignin`]: https://docs.rs/lignin/0.1/lignin

## 嵌入外部管理的数据

> Embedding externally-managed data

我的 `tiptoe` crate 将宿主值实例 (hosted value instances)
的智能指针的引用计数直接存储在其内部。pinning 让它们仍然可以将独占引用公开为 `Pin<&mut T>`。

[`tiptoe`]: https://lib.rs/crates/tiptoe 

您可以在[这篇较早的帖子][intrusive rc]中阅读更多关于侵入式引用计数 (intrusive reference-counting)
及其启用的 heap-only 模式的内容。

[intrusive rc]: https://blog.schichler.dev/intrusive-smart-pointers-heap-only-types-ckvzj2thw0caoz2s1gpmi1xm8

## 维持自我引用

> Persisting self-references

思考以下 [`async`](https://rust-lang.github.io/async-book/03_async_await/01_chapter.html) 代码：

```rust, ignore
async {
    let a = "Hello, future!";
    let b = &a;
    yield_once().await;
    println!("{}", b);
}
```

这段代码创建了一个不透明类型 (opaquely-typed) [`Future`][`Future`] 
实例，在第一次轮询 ([polled][polled]) 之后，该实例将至少在形式上包含对其另一个字段的引用。

[`Future`]: https://doc.rust-lang.org/stable/core/future/trait.Future.html 
[polled]: https://doc.rust-lang.org/stable/core/future/trait.Future.html#the-poll-method 

此时该实例将不再被外部借用，但移动它会将使 `a` 的私有引用中断，因此 `Future::Poll(…)` 需要
`self: Pin<&mut Self>` 作为接收方 (receiver) 。

这保证了 `iml Future` 的实例只有在已经保证不会再次移动的情况下才会进入这种状态。如果执行者确实需要移动 
`Future`，可以要求 `Future + Unpin`，[这允许直接将 `&mut_` 转换为 `Pin<&mut_>`][Pin-new]。

[Pin-new]: https://doc.rust-lang.org/stable/core/pin/struct.Pin.html#method.new

作者附注：pinning 对于安全的“异步”性能来说是一件大事！即使是最终自引用的 `impl Future` 
的实例一开始也是未固定的，但是它们可以直接组合，而不需要像按需将其状态 (state)
提升到堆上这样的变通方法。这会导致生成的状态机 (generated state machines)
中的碎片更少、控制流更少、代码更小（至少在内联之前），所有这些都使快速计算它们变得容易得多。\
或者更确切地说：它提高了异步运行时可能合理实现的上限，因为它不会被它无法控制的生成代码所阻碍。虽然用 Rust
编写简单的异步运行时相当容易，但优秀的异步运行时是以非常“接近金属” (close to the metal)
的方式所运行的调度器，因此经常受到硬件怪癖 (quirks) 的强烈影响。（跟随 Rust 异步开发进展相当有趣。）

# 削弱不可移动性

对于这篇文章的最后部分，让我们再次回到 pinning 最初的保证。

虽然被 pinned 住的实例不能被其他**不相关**的代码安全地移动，但通常仍可以提供如下功能：

```rust, ignore
/// Swaps two pinned instances, making adjustments as necessary.
pub fn swap(
    a: Pin<&mut Self>,
    b: Pin<&mut Self>,
) { … }
```

由于 `swap` 可以访问 `Self` 的私有字段（并且可能依赖一些内部实现的细节，这些细节涉及 `Self`
如何准确使用 pinning），因此它可以在交换过程中根据需要修补 (patch) 任何自引用 (self-referential)
或全局实例注册表指针 (global instance registry pointers)。

类似地，也有可能重新实现 C++ 的涉及地址值移动基础设施 (address-aware value movement infrastructure)
，正如 Miguel Young 在 4 月份的
[*Move Constructors in Rust: Is it possible?*](https://mcyoung.xyz/2021/04/26/move-ctors/#towards-move-constructors-copy-constructors)
一文指出的那样，并在 [`moveit`](https://docs.rs/moveit/0.3.0/moveit/) crate 中实现了。

除了 Rust 程序以这种方式更好地与 C++ 交互之外，pinning collections 还可以使用 `moveit` 的 
[`MoveNew`](https://docs.rs/moveit/0.3.0/moveit/trait.MoveNew.html) 和 
[`CopyNew`](https://docs.rs/moveit/0.3.0/moveit/trait.CopyNew.html) trait，以更接近于
Rust 的方式将其部分 non-pinning API 移植到它们的固定接口：

```rust, ignore
impl<T> C<T> {
    pub fn push_pin(
        this: &mut Pin<Self>,
        value: T,
    ) -> Pin<&mut T>
    where T: MoveNew {
        todo!("Potentially reallocate existing items.")
    }

    pub fn push_pinned(
        this: &mut Pin<Self>,
        value: Pin<MoveRef<'_, T>>,
    ) -> Pin<&mut T>
    where T: MoveNew {
        todo!("Potentially reallocate existing items, move new item.")
    }
}

// Careful: This also enables `Clone` for `Pin<C<T>>`!
impl<T: CopyNew> Clone for C<T> {
    pub fn clone(self) -> Self {
        todo!("Clone items in an address-aware way.")
    }
}
```

具有稳定后备存储 (stable backing storage) 的集合通常可以接受新值，无论它们当前是否是 pin
住了元素 (pinning) 。但是，类似于 [`Vec`](https://doc.rust-lang.org/stable/alloc/vec/struct.Vec.html) 的
pinning 集合有时必须重新分配，因此必须允许其值修补指针以任意扩展功能。

与标准的 [`Clone`][`Clone`] 相比， `CopyNew` 的实现范围更广，`Clone` 
不能用于可能存在内部指针或存在反向引用 (back-reference) 的某些类型上（例如，指向所述实例的
non-owning "multicast" 式引用）。

---

# 感谢

感谢 [Robin Pederson][Robin Pederson]和 [telios][telios] 
的校对和建议，他们在理清文章上提出了各种建议；感谢 [Milou][Milou] 从 C++ 的角度提出了提出的批评和建议。

[Robin Pederson]: https://hashnode.com/@TheBerkin 
[telios]: https://hashnode.com/@telios 
[Milou]: https://github.com/jimkoen 

# 许可和翻译

本文整体上受 [CC BY-NC-SA 2.0](https://creativecommons.org/licenses/by-nc-sa/2.0/)
许可（引文除外）。所有的代码示例（即格式为像 `this` 的代码块和代码片段）都是在
[CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/)
下额外授权的，引文除外。因此你可以在任何许可或无许可的情况下，在你的项目中自由使用。

来自 Rust 官方项目的引文保留了其原始的 `MIT OR Apache-2.0` 许可，并在 
[https://www.rust-lang.org/policies/licenses](https://www.rust-lang.org/policies/licenses)
允许的情况下使用。很抱歉这里的许可复杂，不幸的是我（原作者）的国家几乎不承认合理使用。

如果你翻译了这篇文章，请让我知道，这样我就可以链接到这里。我应该很快就能自己贴出德文翻译了。

我建议对翻译中的代码片段使用与这里相同的许可结构，尽管这我无法强制你执行。
如果翻译使用不同的许可证，你很可能仍然可以从 CC0 许可下的原始版本中获取所需的代码。

（译者注：译文许可遵照原文。）
