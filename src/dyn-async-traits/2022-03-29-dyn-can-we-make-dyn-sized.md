# `dyn*` 让 `dyn` 有大小

[原文](https://smallcultfollowing.com/babysteps/blog/2022/03/29/dyn-can-we-make-dyn-sized/) |
日期：2022-03-29 05:32 -0400[^date]

上周五，Tmanry、Cramertj 和我进行了一次激动人心的谈话。我们正在讨论将 traits 中的异步函数与 `dyn Trait` 相结合的设计，Tmanry 和我在这周五将它提交给了
Rust 语言团队的。Cramertj 在这个设计上提供了一个有洞察力的想法，我想在这里谈谈它。

请记住，这是一项新鲜出炉的、正在进行中的设计，因此可能很容易无疾而终 —— 但同时，我对它感到非常兴奋。如果它成功了，它可能会在很长一段时间内让
`dyn Trait` 在 Rust 变得易于掌握和容易上手，我认为这将是一件大事。

[^date]: 译者注：这篇文章写于 [Rust 之魂](./2022-09-18-dyn-async-traits-part-8-the-soul-of-rust.md) 之前。此外，2022 年 11 月发布的
    [RustcContributor::explore: @eholk session - dyn* and dyn async fns](https://www.youtube.com/watch?v=6mbPY4Mxzys) 视频也介绍了 `dyn*`。

## 背景：dyn 的核心问题

`dyn Trait` 是 Rust 最让人沮丧的功能之一。

一方面，`dyn Trait` 的是绝对必要的。你需要能够构建各种类型的集合，这些集合都实现了一些公共接口，以便实现系统的核心部分。

但是处理异质 (heterogeneous) 类型从根本上来说是很困难的，因为你不知道它们有多大。

这意味着你必须通过指针来操作它们，这就带来了如何管理这些指针所指向的内存的问题。这就是问题开始的地方。

## 问题：内核中没有内存分配器

分配带来一个难点。所有 Rust 程序所需的核心库 `libcore` 没有内存分配器的概念。它完全依赖于栈分配。

在大多数情况下，这很好用：你可以通过将对象从一个栈帧复制到另一个栈帧来传递对象的所有权。但如果你不知道它们占用了多少栈空间，它就没法工作。[^alloca]

[^alloca]: 但是，你会想到 alloca。但是 alloca
    并不是一个真正好的选择。首先，它不适用于所有目标平台，尤其不适用于需要固定大小栈帧的异步函数。它也不允许你将数据返回到栈上，至少不是很容易。

## 问题：`dyn Trait` 不能真正取代 `impl Trait`

在今天的 Rust 中，只要 `Trait` 是 dyn safe，类型 `dyn Trait` 就保证实现 `Trait`。

这看起来很酷，但实际上并没有那么有用。考虑一个对任何 `Debug` 类型进行操作的简单函数：

```rust,ignore
fn print_me(x: impl Debug) {
    println!("{x:?}");
}
```

尽管 `Debug` trait 是 dyn safe，但你不能只将上面的 `impl` 更改为 `dyn`：

```rust,ignore
fn print_me(x: dyn Debug) { ... }
```

这里的问题是，栈分配的参数需要有一个已知的大小，而我们不知道 `dyn` 有多大。常见的解决方案是引入某种类型的指针，例如引用：

```rust,ignore
fn print_me(x: &dyn Debug) { ... }
```

这对此函数来说还不错，但也有一些缺点。
* 首先，我们必须更改 `print_me` 已有的调用方 —— 可能以前是 `print_me(22)`，但现在写成 `print_me(&22)`。这是人体工程学的重磅炸弹。
* 其次，现在对 `dyn Debug` 的借用进行了硬编码。在其他函数中，这不一定是我们想要的。也许我们希望将 `dyn Debug` 
  存储到数据结构中并返回它。例如，函数 `print_me_later` 返回一个闭包，该闭包在被调用时将打印 `x`：

```rust,ignore
fn print_me_later(x: &dyn Debug) -> impl FnOnce() + '_ {
    move || println!("{x:?}")
}
```

假设我们想产生一个线程，该线程将调用 `print_me_later`：

```rust,ignore
fn spawn_thread(value: usize) {
   let closure = print_me_later(&value);
   std::thread::spawn(move || closure()); // <— Error, 'static bound not satisfied
}
```

此代码不会编译，因为 `closure` 引用了栈上的 `value`。但是，如果我们编写了带有 `impl Debug` 参数的 
`print_me_later`，它就可以取得其参数的所有权，一切都会正常工作。

当然，我们可以通过使用 `Box` 来编写 `print_me_later` 以解决这个问题，但这是硬编码的内存分配。

如果我们希望 `print_me_later` 出现在一个上下文中，比如 `libcore`，而该上下文甚至可能无法访问内存分配器，那么这就有问题了。

```rust,ignore
fn print_me_later(x: Box<dyn Debug>) -> impl FnOnce() + '_ {
    move || println!("{x:?}")
}
```

在这个具体的例子中，`Box `也是一种低效的做法。毕竟，`x` 只是一个 `usize` 的话，`Box` 也是一个 `usize`，所以理论上我们可以只复制整数（毕竟 `usize`
方法需要 `&usize`）。这是一种特殊的情况，但在系统的较低级别出现的情况确实比你想象的要多，此时，可能值得费力尝试将东西打包成 `usize` ——
例如，有许多 Future 实际上不需要太多状态。

## 想法：如果 `dyn` 就是指针，那会怎样？

在 Tmanry 和我提出的“async fns in traits”的提案中，我们引入了 `dynx Trait` 类型的概念。

`dynx Trait` 类型并不是使用者会输入的实际语法；相反，它们是一个实现细节。

实际上，`dynx Future` 是指一个指向实现了 `Future` 的类型的指针。它们不会硬编码该指针是一个 `Box`；相反，vtable 包含一个 drop 
函数，该函数知道如何释放指针的引用对象（对于 `Box`，它将释放内存）。

## 更好的想法：如果 `dyn` 是“某种大小已知的东西”会怎样？

在 Rust 语言团队会议之后，Tmanry 和我遇到 Cramertj，他向我们指出了一些非常有洞察力的事情。[^old]

事实上，`dynx Trait` 不必指向实现了 `Trait` 的对象的指针 —— 它只需要是指针大小的对象。Tmanry 和我实际上都知道这一点，但我们没有看到这是多么重要：

* 首先，在实践中，许多 Futures 由很少的状态组成，而且可以是指针大小的。例如，从文件描述符读取数据时只需要存储文件描述符，它是一个 32
  位整数，因为内核存储其他状态。类似地，计时器或其他内置运行时原语的未来通常只需要存储一个索引
* 第二，`dynx Trait` 允许你编写代码来操作可能被 Box 的值，而不直接涉及 Box。这对于想要出现在 libcore 或任何可能的上下文中重用的代码来说至关重要
    * 一个更容易理解的例子是，位于 libcore 中的 `Waker` 结构体实际上是一个手写的 `dynx Waker` 结构体
* 最后，我们稍后会讨论这一点，许多低级别系统代码使用了一些聪明的技巧，他们知道一些值的布局
    * 例如，你有一个包含各种类型的值的 Vec，但是 (A) 所有这些类型都具有相同的大小，并且 (B)
      它们都共享一个公共前缀。此时，你可以在不知道包含哪种数据情况下操作前缀中的字段，并使用 vtable 或判别式 (discriminatory) 来完成其余的操作。
    * 在 Rust 中，这种模式的编码很麻烦，尽管有时可以使用 `vec<S>` 来完成，其中 `S` 是包含前缀字段和枚举的结构体。枚举可以很好地工作，但如果你有一组更开放的类型，那么可能更喜欢使用 trait objects

[^old]: 此外，Cramertj 显然很久以前就有这个想法，但我们并不真正理解它。好吧，有时候事情是这样的，你必须重新创造一些东西，才能意识到最初的发明家到底有多聪明。

## 草图：dyn-star 类型

为了让你感觉到“固定大小的 dyn 类型”有多酷，我将从一个非常简单的设计草图开始。

假设引入一个新类型 `dyn* Trait`，它表示一对：

* 某个实现了 `Trait` 的类型 `T` 的指针大小的值（`*`的意思是传递“指针大小”[^cool]）
* 针对 `T: Trait` 的 vtable；vtable 中的 drop 方法用于销毁 `T` 的值。

[^cool]: 事实上，我只是觉得 `dyn-star` 听起来很酷。我一直嫉妒 `A*` 算法，并想用类似的方式命名一些东西。现在我的机会来了！哈哈!

现在，不要太纠结于特定的语法。有足够的时间来讨论和完善这个语法，我将谈一谈如何才能真正分阶段地使用像 `dyn*` 
这样的东西。现在，让我们只讨论一下使用它会是什么样子。

## 创建 `dyn*`

要将类型 `T` 的值强制转换为 `dyn* Trait`，必须满足两个约束：

* 类型 `T` 必须为指针大小或更小
* 类型 `T` 必须实现 `Trait`

## 将 `impl` 转换为 `dyn*`

`dyn*` 可以将 `impl Trait` 直接转换为 `dyn* Trait`。这很好用，因为 `dyn* Trait` 是 `Sized`。

为了真正等同于 `impl Trait`，你实际上需要一个 lifetime bound，这样 `dyn*` 也可以表示引用：

```rust,ignore
// fn print_me(x: impl Debug) {...} becomes
fn print_me(x: dyn* Debug + '_) {
    println!("{x:?}");
}

fn print_me_later(x: dyn* Debug + '_) -> impl FnOnce() + '_ {
    move || println!("{x:?}")
}
```

这两个函数可以在 `usize` 上直接调用（例如 `print_me_late(22)` 编译通过）。

更重要的是，它们可作用于引用（例如 `print_me_late(&some_type)`）或 Box `print_me_late(Box::new(some_type))`）。

它们也适合 no-std 项目中使用，因为它们不直接涉及分配器。相反，当 `dyn*` 被 drop 时，将从 vtable 调用它的析构函数，这可能会释放内存（但不是必须的）。

## `dny*` safe 的东西比 dyn safe 的东西更多

许多事情对于 `dyn Trait` 的值来说很难，但对于 `dyn* Trait` 的值来说却很容易：

* 按值的 `self` 方法可以很好地工作：`dyn* Trait` 的值是有大小的，所以只需复制它的字节就可以移动它的所有权。
* 返回 `Self`，就像在 `Clone` trait 中一样，良好地工作。
    * 同样 `trait Clone: Sized` 并不意味着 `dyn* Clone` 不能实现 `Clone`，尽管它确实意味 `dyn Clone: Clone` 不成立
* 函数参数中的 `impl ArgTrait` 类型可以转换为 `dyn* ArgTrait`，只要 `ArgTrait` 是 `dyn*` safe
* 函数返回 `impl ArgTrait` 时也可以返回 `dyn* ArgTrait`

简而言之，许多不满足 dyn safe 的障碍并不适用于 `dyn*`。当然，不是全部。

接受 `Self` 类型参数的 traits 不能工作（因为不知道两个 `dyn* Trait`
类型是否具有相同的底层类型），而且在许多情况下也不能支持泛型方法（因为不知道如何单态化）[^options]。

[^options]: 显然，我们可能会减轻这一点来适应`impl Trait` 参数。我认为可以在更多的情况下取消这一限制，但这将需要更多的设计。

## 陷阱 1：`dyn* foo` 要求 `Box<impl foo>: foo`

这当中有一个陷阱，但我更愿意将其视为一个机会。

为了从 `Box<Widget>` 这样的指针类型创建 `dyn* Trait`，你需要知道 `Box<Widget>: Trait`，而创建
`Box<dyn Trait>` 只需要知道 `Widget: Trait`（这直接源于 `Box` 现在是隐藏类型的一部分）。

目前烦人的是，当定义一个 trait 时，你不会自动获得任何“指向实现该 trait 的类型的指针”的实现。

反而，人们通常会手动定义这些指针的 trait，例如 `Iterator` trait 有这样的实现：

```rust,ignore
impl<I> Iterator for &mut I
where
    I: ?Sized + Iterator

impl<I> Iterator for Box<I>
where
    I: ?Sized + Iterator
```

然而，许多人忘记了定义这样的实现，这在实践中可能会很烦人（不仅仅是使用 dyn 时）。

我不是非常确定解决这个问题的最好方法，但我认为这是一个机会，因为如果我们能提供这样的实现，那将使 Rust 整体上更符合人体工程学。

有趣的是：你在上面看到的 `Iterator` 的实现包括 `I: ?Sized`，这使得它们适用于 `Box<dyn Iterator>`。

但是对于 `dyn* Iterator`，我们是从 `Box<impl Iterator>` 类型开始的。

换句话说，`?Sized` 约束不是必需的，因为是围绕指针创建 `dyn` 抽象的，指针是有大小的。

当然，`?Sized` 也是无害的，如果我们自动生成这样的实现，我们应该涵盖它，以便让它们应用于旧式的 `dyn` 以及像 `[u8]` 这样的切片类型。

## 陷阱 2：traits 的“共享子集”

Rust 在 `Trait` 设计方面的一个很酷的地方是，它允许你将“只读”和“修饰符” (modifier) 方法组合到一个 trait 中，如下例所示：

```rust,ignore
trait WidgetContainer {
    fn num_components(&self);
    fn add_component(&mut self, c: WidgetComponent);
}
```

我可以编写一个接收 `&mut dyn WidgetContainer` 的函数，它将能够同时调用这两个方法。如果该函数接收 `&dyn WidgetContainer`，则只能调用`num_components`。

如果我们不做其他任何事情，这种灵活性将在 `dyn*` 中丧失。假设我们希望从某个 `&impl WidgetContainer` 类型创建一个 `dyn* WidgetContainer`。

要做到这一点，我们需要为 `&T` 实现 `WidgetContainer`，但我们无法编写该代码，至少不能在不引起 panic 的情况下编写代码：

```rust,ignore
impl<W> WidgetContainer for &W
where
    W: WidgetContainer,
{
    fn num_components(&self) {
        W::num_components(self) // OK
    }

    fn add_component(&mut self, c: WidgetComponent) {
        W::add_component(self, c) // Error!
    }
}
```

这个问题并不是 `dyn` 所特有的。假设我有一些代码只调用 `num_Components`，但可以使用 `&W` 或 `Rc<W>`
或其他类似类型来调用我的代码。现在编写这样的函数对我来说有点尴尬：最简单的方法是硬编码它接收 `&w`，然后在调用者中依赖解引用强制。

Tmanry 和我一直在讨论的一个想法是，对让 traits 有“视图”。

我们的想法是，你可以编写类似 `T: &WidgetContainer` 的代码来表示“ `WidgetContainer` 的 `&self` 方法”。如果你有这个想法，那么你肯定会得到

```rust,ignore
impl<W> &WidgetContainer for &W
where
    W: WidgetContainer
```

因为你只需要定义 `num_components`（尽管我希望你不必手动编写这样的 impl）。

现在，你可以使用 `dyn &WidgetContainer`，而不是使用 `&dyn WidgetContainer`。

同样，与其使用 `&impl WidgetContainer`，你可能更好地使用 `impl &WidgetContainer`（碰巧这也有其他一些好处）。

## 陷阱 3：dyn safe 有时会对实现施加限制，而不仅仅是 trait 本身

Rust 目前的设计是假设单一的 trait 定义，从这个 trait 定义中确定是否为 dyn safe。

但有时关于 dyn safety 的限制实际上并不影响 trait，而只是影响 trait 的实现。

有种情况在“隐式 dyn safety”中并不能工作：如果你确定该 trait 是 dyn safe，则必须对其实现施加这些限制，但也许该 trait 并不意味着动态安全。

总之我认为，如果 trait 明确地宣布其意图是 dyn safe 或者不是 dyn safe，那会更好。要做到这一点，最显然的方法是使用类似 `dyn trait` 进行声明：

```rust,ignore
dyn trait Foo { }
```

一个很好的额外好处，这样的声明还可以自动生成 `impl Foo for Box<impl Foo + ?Sized>` 等实现。这也将意味着 dyn safety 成为一种语义版本保证。

我在这里主要担心的是，我怀疑大多数 traits 可能而且应该是 dyn safe 的。

我想我宁愿选择退出 dyn safe 而不是选择加入。当然，我不知道它的语法是什么，而且我们必须处理向前兼容性。

## 在一个版次内分阶段进行

如果可以重新开始，我想我会这样做：

* `dyn Trait` 语法表示 “实现了 `Trait` 的指针大小的值”。通常是 `Box` 或 `&`，但有时也可以是其他东西
* `dyn[T] Trait` 语法表示 “与实现了 `Trait` 的 T 布局兼容的值”；因此，`dyn Trait` 是 `dyn[*const ()] Trait` 的语法糖，我们可以更简洁地将其写为 `dyn* Trait`
* `dyn[T..] Trait` 语法表示 “以前缀 `T` 开头但大小未知并实现了 `Trait` 的值”
* `dyn[..] Trait` 语法表示 “实现了 `Trait` 的某个未知类型的值”

同时，我们会用一些新的功能来扩展 trait bound 语法：

* `&Trait<P...>` 之类的约束表示 “只有 `Trait` 中的 `&self` 方法”
* `&mut Trait<P...>` 之类的约束表示 “只有 `Trait` 中的 `&self` 和 `&mut self` 方法
    * 可能这里面也包括 `Pin<&mut self>`？我还没有想过这一点。
* 可能需要一种方法来编写像 `Rc<Trait<P...>>` 这样的约束来表示 `self: Rc<Self>` 等内容，但我还不知道那是什么样子。这种 traits 很少见

我想大多数人都只需要学会 `dyn Trait`。`dyn[]` 表示法的用例要专业得多，稍后介绍。

有趣的是，如果愿意，我们可以在 Rust 2024 中逐步引入这种语法。我们的想法是，在为新版次做准备时，将 `dyn` 的现有用法改为显式形式, 例如：

* `&dyn Trait` 将变为 `dyn* Trait + '_`
* `Box<dyn Trait>` 将变为 `dyn* Trait`（注意，现在 Rust 隐含了 `'static` 约束；这可能得重新考虑，但这是另一个问题）
* `dyn Trait` 的其他用法将变为 `dyn[...] Trait`

然后，在 Rust 2024 中，我们将使用 “edition idiom lint” 把 `dyn* Trait` 重新改为 `dyn Trait`。

## 结论

呼！这是一个很长的帖子。让我总结一下所谈到的内容：

* 如果 `dyn Trait` 表示实现了 `Trait` 的指针大小的值，而不是未知大小的值，那么：
    * 我们可以将 dyn safe 扩展相当多，而不需要聪明的黑客技巧：
        * 按值 self 的方法：`fn into_foo(self, ...)`
        * impl Trait 类型参数的方法（只要 `Trait` 是 dyn safe）：`fn foo(..., impl Trait, ...)`
        * 返回 impl Trait 的方法：`fn iter(&self) -> impl Iterator`
        * 返回 `Self` 类型的方法：`fn clone(&self) -> Self`
* 这会引发一些我们必须处理的问题，但所有这些都是有用的：
    * 你需要 `&dyn Trait` 和其他东西来“选择”方法集
    * 你需要更符合人体工程学的方法来确保 `Box<Trait>: Trait` 等等
* 我们可以通过引入两个语法在 Rust 2024 合理地过渡到这个模型：`dyn*`（指针大小）和 `dyn[..]`（未知大小），然后更改 `dyn` 的含义

有许多细节需要解决，但其中最突出的是：

* 我们应该明确声明 dyn-safe traits 吗？（我想是的）
    * 当我们这样做时，应该创建什么“过渡”实现？（例如涵盖 `Box<impl Trait>: Trait` 等）
* `&Trait` 约束到底如何工作 —— 你会自动获得 impls 吗？一定要这样写吗？

## 附录 A：更疯狂的任意前缀 `dyn[T]`

`dyn*` 非常有用。但我们实际上可以对其进行泛化。

你可以想象 `dyn[T]` 将表示为 “把布局读为 `T` 的值”。因此，我们所说的 `dyn* Trait` 等同于 `dyn[*const ()] Trait`。

这个更通用的版本允许我们打包更大的值，例如你可以写 `dyn[[usize; 2]] Trait` 来表示 “两个字大小的值”。

你甚至可以想象编写 `dyn[T]`，其中 `T` 表示你可以安全地将背后的值作为 `T` 实例进行访问。

这将提供对所实现的类型必须的公共字段或其他类似内容的访问。系统编程黑客经常依赖于这样的聪明东西。

如果 `T` 是像 `usize` 只是表示有多少字节的数据的类型，这就有点棘手了，因为如果你要允许将 `dyn[T]` 当作 `&mut T`
对待，使用者可能会疯狂地以肯定无效的方式覆写值。

因此，我们必须认真考虑这一点，才能使其发挥作用，这就是为什么我将其留在附录中。

## 附录 B：dyn 的其他重大问题

我认为这篇文章中的设计解决了 dyn 的许多重大问题：

* 你不能像使用 impl 一样使用它
* 许多有用的 trait 功能并不是 dyn safe
* 你必须在 impls 上写 `?Sized` 才能使它们工作

但这会留下一些问题没有解决。在我的脑海中，最大的问题之一是与 auto trais（实际上，还有生命周期）交互。

对于 `T: Debug` 这样的泛型参数，不需要明确地考虑 `T` 是否为 `Send`，或者 `T` 是否包含生命周期。

我可以只写一个泛型类型，比如 `struct MyWriter<W> where W: Write { w: W, ...}`。`MyWriter` 的使用者知道 `W`
是什么，所以他们可以根据 `Foo: Send` 来判断 `MyWriter<foo>: Send`，也能够理解 `MyWriter<&'a Foo>` 包含生命周期为 `'a` 的引用。

相反，如果我们确实写成 `struct MyWriter { w: dyn* Write, ...}`，那么`dyn*Write` 类型将隐藏底层数据。

按照目前的 Rust，它意味着 `MyWriter` 不是 `Send`，也不包含引用。

我们没有好的办法让 `MyWriter` 声明 “如果你给我的 writer 是 Send，那么 MyWriter 就是 Send”，并使用 `dyn*`。

这是一个有趣的问题！但我认为，从这篇博客文章中解决的问题来看，这是不相关的。
