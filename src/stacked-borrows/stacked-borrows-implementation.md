# 实现栈借用

> 原文：[Stacked Borrows Implemented](https://www.ralfj.de/blog/2018/11/16/stacked-borrows-implementation.html)
> | 作者：Ralf Jung | 2018 年 11 月 16 日 | [IRLO]

[IRLO]: https://internals.rust-lang.org/t/stacked-borrows-implemented/8847

三个月前，我提出栈借用 (Stacked Borrows)，一种定义了 Rust 中允许哪些别名的模型，并提出了有效性不变量
[validity invariant] 的概念，它始终必须由所有代码维护。

从那时起，我一直忙于实现这两个功能，并在此过程中进一步开发了栈借用。

这篇文章描述了栈借用的最新版本，并报告了我在实施阶段的发现：什么可以工作，什么不工作，还有什么需要做。你也会有机会参与进来！

这篇文章是对栈借用的一个独立的介绍。

除非你想知道提出它的历史或者与之前提出的 [Types as Contracts] 的区别，没有理由阅读 [上一篇帖子][old-post]。

栈借用所做的是，它定义了 Rust 程序的语义，使得有关引用的某些内容对于每个有效的执行总是成立的（意味着不会出现 [UB]）：
* `&mut` 引用是唯一的（我们可以相信其他函数不会访问它们所指向的内存）
* 而 `&` 引用是不可变的（我们可以相信不会对它们所指向的内存进行写入，除非有 `UnsafeCell` ）

通常，借用检查器会保护我们，它防止引用类型的保证被恶意违反，但是，唉，当我们编写不安全的代码时，借用检查器无法帮助我们。

我们必须定义一组即使对于不安全的代码也有意义的规则。

我将在这篇文章中再次解释这些规则。解释不会和 [上次][old-post] 
相同，这不仅是因为它有一点变化，还因为我认为我现在更好地理解了这个模型，所以可以更好地解释它。

准备好了吗?我们开始吧。我希望你拿出一些时间，因为这是一个相当长的帖子。

如果你对栈借用的详细描述不感兴趣，可以跳过这篇文章的大部分内容，直接转到第 4 节。如果你只想知道如何提供帮助，请跳到第 6 节。

[validity invariant]: https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html
[Types as Contracts]: https://www.ralfj.de/blog/2017/07/17/types-as-contracts.html
[old-post]: ./stacked-borrows.md
[UB]: https://www.ralfj.de/blog/2017/07/14/undefined-behavior.html

## 1. 强制执行唯一性

首先忽略关于 `&` 引用是不可变的部分，而关注可变引用的唯一性。

也就是说，我们希望以这样一种方式定义我们的模型：调用以下函数将触发未定义的行为：

```rust
fn demo0() {
  let x = &mut 1u8;
  let y = &mut *x;
  *y = 5;
  // Write through a pointer aliasing `y`
  *x = 3;
  // Use `y` again, asserting it is still exclusive
  let _val = *y;
}
```

我们希望这个函数不被允许，因为在两次使用 `y` 之间，同一位置使用了另一个指针，这违反了 `y` 应该是唯一的。

请注意，此函数不能编译，借用检查器不允许它通过。太好了!毕竟，这是一种 UB。

但此例的整个要点是解释为什么在这里有未定义的行为，而不是利用借用检查器，因为我们希望有也适用于不安全代码的规则。

事实上，你可以说，这些规则可以回过头来解释借用检查器为什么会这样工作：假装模型先出现，而借用检查器只是执行编译时检查，以确保我们遵循模型的规则。

要做到这一点，我们必须假装机器有两个真实的 CPU 所没有的东西。这是一个用“虚拟机”添加“影子状态” (shadow state)
或“检测状态” (instrumented state) 来 [指定 Rust] 的例子。它并不是一种罕见的方法，源语言经常会做出实际硬件中不会出现的事情。一个相关的例子是
valgrind 的 [memcheck]，它跟踪哪个内存被初始化来检测内存错误：在正常执行期间，未初始化的内存看起来就像所有其他内存一样，但要确定程序是否违反了
C 语言的内存规则，我们必须跟踪一些额外的状态。

[指定 Rust]: https://www.ralfj.de/blog/2017/06/06/MIR-semantics.html
[memcheck]: http://valgrind.org/docs/manual/mc-manual.html

对于栈借用，额外状态如下所示：

1. 对于每个指针，跟踪记录该指针何时以及如何创建的额外的“标签”。
2. 对于每个内存中的位置，跟踪“项” (items) 的栈，这些项表明指针必须有哪种标签 (tag) 来被允许访问该位置。

这些单独存在，也就是说，当指针存储在内存中时，都有一个标签作为了该指针值的一部分（记住，[字节不仅仅有 `u8`]
），指针占用的每个字节都有一个栈来管理对该位置的访问。这两者也不交互，也就是说，当从内存加载指针时，只加载作为该指针的一部分存储的标签。
位置的栈和存储在某个位置的指针的标签彼此之间没有任何影响。

[字节不仅仅有 `u8`]: https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html

在我们的示例中，有两个指针（ `x` 和 `y`）和一个感兴趣的位置（这两个指针都指向的位置，初始化为 `1u8`）。

当最初创建 `x` 时，它被标签为 `Uniq(0)` ，表示它是唯一的引用，位置的栈的顶部有 `Uniq(0)` ，表示这是允许访问该位置的最新引用。

当创建 `y` 时，它会有一个新的标签 `Uniq(1)` ，这样我们就可以将它与 `x` 区分开来。我们还将 `Uniq(1)` 推送到栈上，不仅表明
`Uniq(1)` 是允许访问的最新引用，而且还表明它是从 `Uniq(0)` 派生的：栈中更高的标签是更低的标签的后代。

所以在创建这两个引用后，有：`x` 被标签为 `Uniq(0)`，`y` 被标签为 `Uniq(1)`，栈有 `[Uniq(0), Uniq(1)]`。（栈的顶部在右侧。）

当使用 `y` 访问位置时，我们确保它的标签在栈的顶部：检查，这里没有问题。当使用 `x`
时，我们做同样的事情：因为它还不在顶部，我们弹出栈，直到它在顶部，这很容易。现在栈只是 `[uniq(0)]`。然后再用
`y` ，然后..嘣！其标签不在栈上。我们有 UB。

如果你没跟上，以下是源代码，带有注释，展示我们感兴趣的一个位置的标签和栈：

```rust
fn demo0() {
  let x = &mut 1u8; // tag: `Uniq(0)`
  // stack: [Uniq(0)]

  let y = &mut *x; // tag: `Uniq(1)`
  // stack: [Uniq(0), Uniq(1)]

  // Pop until `Uniq(1)`, the tag of `y`, is on top of the stack:
  // Nothing changes.
  *y = 5;
  // stack: [Uniq(0), Uniq(1)]

  // Pop until `Uniq(0)`, the tag of `x`, is on top of the stack:
  // We pop `Uniq(1)`.
  *x = 3;
  // stack: [Uniq(0)]

  // Pop until `Uniq(1)`, the tag of `y`, is on top of the stack:
  // That is not possible, hence we have undefined behavior.
  let _val = *y;
}
```

实际上，在这里有 UB 是个好消息，因为这是我们从一开始就想要的！由于在 [Miri] 中实现了该模型，你可以自己尝试一下：
@shepmaster 已将 Miri 集成到 playground 中，因此你可以将 [示例] 放在那里（略微调整了它以绕过借用检查器），然后选择
“Tools - Miri”，它会发出警告：

[Miri]: https://github.com/solson/miri/
[示例]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=d15868687f79072688a0d0dd1e053721

```rust,ignore
error: Undefined Behavior: attempting a read access using <3121> at alloc1656[0x0], but that tag does not exist in the borrow stack for this location
 --> src/main.rs:6:14
  |
6 |   let _val = *y;
  |              ^^
  |              |
  |              attempting a read access using <3121> at alloc1656[0x0], but that tag does not exist in the borrow stack for this location
  |              this error occurs as part of an access at alloc1656[0x0..0x1]
  |
  = help: this indicates a potential bug in the program: it performed an invalid operation, but the Stacked Borrows rules it violated are still experimental
  = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md for further information
help: <3121> was created by a Unique retag at offsets [0x0..0x1]
 --> src/main.rs:3:20
  |
3 |   let y = unsafe { &mut *(x as *mut u8) };
  |                    ^^^^^^^^^^^^^^^^^^^^
help: <3121> was later invalidated at offsets [0x0..0x1] by a write access
 --> src/main.rs:5:3
  |
5 |   *x = 3;
  |   ^^^^^^
  = note: BACKTRACE (of the first span):
  = note: inside `main` at src/main.rs:6:14: 6:16
```

## 2. 启用共享

如果我们只有独占指针，Rust 将是一门相当枯燥的语言。

幸运的是，对某个位置的共享访问也有两种方式：通过（安全的）共享引用和通过（不安全的）裸指针。

此外，共享引用有时断言一个额外的保证：它们的目的地是不可变的。（但当它们指向 `UnsafeCell` 时不会）

例如，我们希望允许以下代码 —— 尤其因为这实际上是借用检查器接受的安全代码，所以最好确保这不是 UB：

```rust
fn demo1() {
  let x = &mut 1u8;
  // Create several shared references, and we can also still read from `x`
  let y1 = &*x;
  let _val = *x;
  let y2 = &*x;
  let _val = *y1;
  let _val = *y2;
}
```

但是，以下代码不允许：

```rust
fn demo2() {
  let x = &mut 1u8;
  let y = &*x;
  // Create raw reference aliasing `y` and write through it
  let z = x as *const u8 as *mut u8;
  unsafe { *z = 3; }
  // Use `y` again, asserting it still points to the same value
  let _val = *y;
}
```

如果你在 Miri 上尝试 [运行][playground2]，你会看到它抱怨：

[playground2]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=1bc8c2f432941d02246fea0808e2e4f4

```rust,ignore
 --> src/main.rs:6:14
  |
6 |   let _val = *y;
  |              ^^ Location is not frozen long enough
  |

// 上面是原文的、古老的错误；下面是当前翻译时看到的错误
error: Undefined Behavior: attempting a write access using <3120> at alloc1656[0x0], but that tag only grants SharedReadOnly permission for this location
 --> src/main.rs:5:12
  |
5 |   unsafe { *z = 3; }
  |            ^^^^^^
  |            |
  |            attempting a write access using <3120> at alloc1656[0x0], but that tag only grants SharedReadOnly permission for this location
  |            this error occurs as part of an access at alloc1656[0x0..0x1]
  |
  = help: this indicates a potential bug in the program: it performed an invalid operation, but the Stacked Borrows rules it violated are still experimental
  = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md for further information
help: <3120> was created by a SharedReadOnly retag at offsets [0x0..0x1]
 --> src/main.rs:4:11
  |
4 |   let z = x as *const u8 as *mut u8;
  |           ^
  = note: BACKTRACE (of the first span):
  = note: inside `main` at src/main.rs:5:12: 5:18
```

它是如何做到这一点的，什么是“冻结” (frozen) 的地点？

要解释这一点，我们必须稍微扩展一下“虚拟机”的“影子状态”。

首先，我们介绍了一种指针可以携带的新标签：共享标签 (shared tag)。下面的 Rust 类型描述了指针的可能标签：

```rust
pub type Timestamp = u64;
pub enum Borrow {
    Uniq(Timestamp),
    Shr(Option<Timestamp>),
}
```

你可以将时间戳视为唯一的 ID，但正如我们将看到的，对于共享引用，能够确定这些 ID 中的哪个 ID 是首先创建的也很重要。

时间戳在共享标签中是可选的，因为该标签也被裸指针使用，并且对于裸指针，我们通常无法跟踪它们是何时以及如何创建的（例如，裸指针何时被转换为整数并被转换回来）。

我们对栈上的项使用单独的类型，因为在那里我们不需要共享指针的时间戳：

```rust
pub enum BorStackItem {
    Uniq(Timestamp),
    Shr,
}
```

最后，“借用栈”由一个 `BorStackItem` 栈组成，并表示该栈（及其管理的位置）当前是否处于冻结状态，这意味着它或许只能读而不能写：

```rust
pub struct Stack {
    borrows: Vec<BorStackItem>, // used as a stack; never empty
    frozen_since: Option<Timestamp>, // virtual frozen "item" on top of the stack
}
```

### 2.1 执行示例

现在让我们来看看当执行两个示例程序时会发生什么。为此，我将在源代码中嵌入注释。

这里只有一个感兴趣的位置，所以每当我谈到“栈”时，我指的是该位置的栈。

```rust
fn demo1() {
  let x = &mut 1u8; // tag: `Uniq(0)`
  // stack: [Uniq(0)]; not frozen

  let y1 = &*x; // tag: `Shr(Some(1))`
  // stack: [Uniq(0), Shr]; frozen since 1

  // Access through `x`.  We first check whether its tag `Uniq(0)` is in the
  // stack (it is).  Next, we make sure that either our item *or* `Shr` is on
  // top *or* the location is frozen.  The latter is the case, so we go on.
  let _val = *x;
  // stack: [Uniq(0), Shr]; frozen since 1

  // This is not an access, but we still dereference `x`, so we do the same
  // actions as on a read.  Just like in the previous line, nothing happens.
  let y2 = &*x; // tag: `Shr(Some(2))`
  // stack: [Uniq(0), Shr]; frozen since 1

  // Access through `y1`.  Since the shared tag has a timestamp (1) and the type
  // (`u8`) does not allow interior mutability (no `UnsafeCell`), we check that
  // the location is frozen since (at least) that timestamp.  It is.
  let _val = *y1;
  // stack: [Uniq(0), Shr]; frozen since 1

  // Same as with `y2`: The location is frozen at least since 2 (actually, it
  // is frozen since 1), so we are good.
  let _val = *y2;
  // stack: [Uniq(0), Shr]; frozen since 1
}
```

这个例子展示了一些新的方面。

* 首先，在这个模型中，实际上（到目前为止）有两个操作执行与标签相关的检查：解引用指针（任何存在
  `*` 的时候，也包括隐式的情况下）和实际的内存访问。像 `&*x` 这样的操作是在不访问内存的情况下解引用指针的操作的示例。
* 其次，读取可变引用实际上是可以的，即使该引用不是独占的。它只是通过一个可变的引用来“重新断言”它的排他性。

我稍后会再谈这些问题，但让我们先来看看另一个例子。

```rust
fn demo2() {
  let x = &mut 1u8; // tag: `Uniq(0)`
  // stack: [Uniq(0)]; not frozen
  
  let y = &*x; // tag: `Shr(Some(1))`
  // stack: [Uniq(0), Shr]; frozen since 1

  // The `x` here really is a `&*x`, but we have already seen above what
  // happens: `Uniq(0)` must be in the stack, but we leave it unchanged.
  let z = x as *const u8 as *mut u8; // tag erased: `Shr(None)`
  // stack: [Uniq(0), Shr]; frozen since 1

  // A write access through a raw pointer: Unfreeze the location and make sure
  // that `Shr` is at the top of the stack.
  unsafe { *z = 3; }
  // stack: [Uniq(0), Shr]; not frozen

  // Access through `y`.  There is a timestamp in the `Shr` tag, and the type
  // `u8` does not allow interior mutability, but the location is not frozen.
  // This is undefined behavior.
  let _val = *y;
}
```

### 2.2 解引用指针

正如我们已经看到的，在任何内存访问发生之前，在对指针解引用时 (dereference) 已经考虑了指针的标签 (tags)。

解引用操作永远不会改变栈，但它会执行一些基本检查，这些检查可能会表明程序有 UB。

这样做的原因有两个：

* 首先，我认为即使在不访问内存的情况下对被解引用的指针要求一些基本的有效性。
* 其次，在 Miri 中实现有一个实际问题：当解引用一个指针时，我们保证有可用的类型信息（这对于依赖于
  `UnsafeCell` 的东西来说至关重要），而在 Miri 中拥有每次内存访问的类型信息是很难实现的。

注意，在解引用时，指针和指针的类型上都有一个标签，这两者可能不一致，我们并不总是排除这种可能（例如，在
`transmute` 之后，它们可能具有具有 unique 标签的裸指针或共享指针)。

对于指针涉及的每个位置，在每个指针解引用时都会执行以下检查（ `size_of_val` 告诉我们指针涉及的字节数）：

1. 如果这是一个裸指针 (raw pointer)，则什么都不做。尽可能少地检查原始访问 (raw access)。
2. 如果这是 unique 引用并且标签是 `Shr(Some(_))`，则这是一个错误。
3. 如果标签是 `Uniq`，则确保栈上存在具有相同 ID 的相匹配的 `Uniq` 项。
4. 如果标签是 `Shr(None)`，则确保位置被冻结 (frozen) 或者栈上有 `Shr` 项。
5. 如果标签是 `Shr(Some(T))`，则后面的检查取决于该位置是否在 `UnsafeCell` 内，根据引用的类型：
    * `UnsafeCell` 之外的位置必须将 `freeze_since` 设置为 `t` 或更早的时间戳
    * `UnsafeCell` 位置必须被冻结或在其栈中有一个 `Shr` 项（与标签没有时间戳的检查相同）

### 2.3 访问内存

在实际的内存访问中，我们知道用于访问的指针的标签（其实，我们总是使用实际的标签，而忽略指针的类型），
并且我们知道是从当前位置读取还是向当前位置写入。在受访问影响的所有位置上执行以下操作：

1. 如果位置被冻结，并且这是读访问，则不会发生任何事情（即使标签是 `Uniq`）。
2. 否则，如果这是写访问，则解冻 (unfreeze) 位置（将 `frozen_since` 设置为 `None`）。
   （如果这是读访问，而我们到了这里，则该位置已解冻。）
3. 弹出栈，直到顶部的项与指针的标签匹配：
   * `Uniq` 项匹配具有相同 ID 的 `Uniq` 标签
   * `Shr` 项匹配任何 `Shr` 标签（有或没有时间戳）
   * 当读取时，`Shr` 项匹配 `Uniq` 标签
   * 如果弹出整个栈但没有找到匹配，那么就有未定义的行为 (UB)

为了更好地理解这些规则，请你尝试回顾到目前为止看到的三个示例，并应用这些规则来解引用指针和访问内存，以了解它们是如何交互的。

这里最微妙的一点是，我们让 `Uniq` 标签匹配 `Shr` 项，并接受冻结位置的 `Uniq` 读取。这是使
`demo1` 能工作所必需的：Rust 允许通过可变引用进行读访问，即使它们当前实际上不是唯一的。因此，我们的模型也必须这样做。

## 3. 重新标签和创建裸指针

我们已经谈了相当多关于 **使用** 指针时会发生什么。现在是时候仔细看看指针是如何创建的。

然而，在谈到这一点之前，再考虑一个例子：

```rust,ignore
fn demo3(x: &mut u8) -> u8 {
  some_function();
  *x
}
```

问题是：我们可以将 `x` 的工作 (load) 移到函数调用之前吗？

请记住，栈借用的全部意义是在使用引用时强制执行特定规则，特别是强制可变引用的唯一性。

因此，我们应该希望这个问题的答案是“是的”（这反过来来说是好的，因为可以将其用于优化）。不幸的是，事情并不是那么容易。

可变引用的唯一性完全取决于这样一个事实：指针具有 unique 标签。

如果一个标签位于栈的顶部（并且位置没有被冻结），则对另一个标签的任何访问都将从栈中弹出项（或者导致未定义的行为）。这是由内存访问检查确保的。

因此，如果在发生了一些其他访问之后，有一个标签仍然在栈上（并且我们知道，按照上面描述的解引用检查，
每次解引用指针时，它仍然在栈上），那么我们就能知道不可能通过具有不同标签的指针产生访问。

### 3.1 保证引用全新

但是，如果 `some_function` 具有与 `x` 完全相同的副本，该怎么办？

我们从（我们不信任的）调用者那里获得了 `x`，也许他们将相同的标签用于另一个引用（比如使用 `transmute_copy`
之类的复制它），并将其提供给 `some_function`？

有一个简单的方法可以绕过这个问题：为 `x` 生成一个新的标签。

如果是我们生成标签（并且知道不会生成两次相同的标签，这很容易），那么就能确保此标签不用于任何其他引用。

因此，让我们通过将 `Retag` 指令放入生成新标签的代码中来明确这一点：

```rust,ignore
fn demo3(x: &mut u8) -> u8 {
  Retag(x);
  some_function();
  *x
}
```

几乎在复制引用的任何时候，编译器都会插入这些 `Retage` 指令：
* 在每个函数的开头，所有引用类型的输入都会被重新标签
* 在每次赋值时，如果赋值类型为引用类型，也会重新标签该值
* 此外，即使当引用值位于 `struct` 或 `enum` 的字段内时，也会重新标签，以确保真的覆盖了所有引用（这种递归下降已经实现，但实现尚未实现）

最后，`Box` 被视为可变引用，以编码它访问唯一的断言。然而，我们不会通过引用递归下降：对于 `&mut &mut u8` 只会重新标签最外层的引用。

重新标签是 **唯一** 生成新标签的操作。使用一个引用只是变相使用基于引用的指针的标签。

以下是显式重新标签的第一个示例：

```rust,ignore
fn demo0() {
  let x = &mut 1u8; // nothing interesting happens here
  Retag(x); // tag of `x` gets changed to `Uniq(0)`
  // stack: [Uniq(0)]; not frozen
  
  let y = &mut *x; // nothing interesting happens here
  Retag(y); // tag of `y` gets changed to `Uniq(1)`
  // stack: [Uniq(0), Uniq(1)]; not frozen

  // Check that `Uniq(1)` is on the stack, then pop to bring it to the top.
  *y = 5;
  // stack: [Uniq(0), Uniq(1)]; not frozen


  // Check that `Uniq(0)` is on the stack, then pop to bring it to the top.
  *x = 3;
  // stack: [Uniq(0)]; not frozen

  // Check that `Uniq(1)` is on the stack -- it is not, hence UB.
  let _val = *y;
}
```

对于每个引用和 `Box`，`Retag` 对引用覆盖的所有位置（同样，根据 `Size_of_val`）执行以下操作（我们稍后将略微细化这些指令）：

1. 计算一个新的标签：可变引用和 `Box` 使用表示共享引用，共享引用使用 `Shr(Some(_))`。
2. 解引用时也会执行检查。
3. 通过该引用进行实际访问时也会发生相同的操作（对于共享引用是读访问，对于可变引用是写访问）。
4. 检查新标签是否为 `Shr(Some(t))` 以及位置是否在 `UnsafeCell` 内：
  * 如果这两个条件都满足，则使用时间戳 `t` 冻结位置；如果已经冻结，则不执行任何操作
  * 否则，将一个新项推送到栈上：如果标签是 `Shr(_)`，则推 `Shr`；如果标签是 `Uniq(id)`，则推 `Uniq(id)`

思考重新标签 (retag) 的一个高层级方式是 retag 其实是计算一个新标签，然后使用新标签重新借用旧引用。

### 3.2 当指针逃逸时

创建共享引用并不是共享一个位置的唯一方法：我们还可以创建裸指针，并且如果足够小心，那么可以从不同的别名指针
(aliasing pointers) 访问位置。

（当然，“足够小心”并不是非常准确，但准确的答案就是我在这里描述的模型。）

为了说明这一点，我们将转换为裸指针的行为视为创建引用的一种特殊方式（所谓的“创建原始引用” ([raw reference](https://github.com/rust-lang/rfcs/pull/2582))）。

像通常创建新引用一样，该操作之后重新标签 (retag)。然而，这种重新标签是特殊的，因为与普通的重新标签不同，它作用于裸指针。

考虑以下 [示例](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=253868e96b7eba85ef28e1eabd557f66)：

```rust,ignore
fn demo4() {
  let x = &mut 1u8;
  Retag(x); // tag of `x` gets changed to `Uniq(0)`
  // stack: [Uniq(0)]; not frozen

  let y1 = x as *mut u8;
  // Make sure what `x` points to is accessible through raw pointers.
  Retag([raw] y1) // tag of `y` gets erased to `Shr(None)`
  // stack: [Uniq(0), Shr]; not frozen

  let y2 = y1;
  unsafe {
    // All of these first dereference a raw pointer (no checks, tag gets
    // ignored) and then perform a read or write access with `Shr(None)` as
    // the tag, which is already the top of the stack so nothing changes.
    *y1 = 3;
    *y2 = 5;
    *y2 = *y1;
  }

  // Writing to `x` again pops `Shr` off the stack, as per the rules for
  // write accesses.
  *x = 7;
  // stack: [Uniq(0)]; not frozen

  // Any further access through the raw pointers is undefined behavior, even
  // reads: The write to `x` re-asserted that `x` is the unique reference for
  // this memory.
  let _val = unsafe { *y1 };
}
```

`Retag([raw])` 的作用与普通的 retag 几乎相同，但不同之处在于它不会忽略裸指针，而是将它们标签为 `Shr(None)`，并将 `Shr` 推入栈。

这样，即使程序将指针转换为一个整数后在转换回指针（此时我们不能总是跟踪标签，因此它可能会被重置为 `Shr(None)`），栈上也会有一个匹配的
`Shr`，以确保裸指针实际可以被使用。

思考这一点的一种方式是考虑在将引用转换为裸指针时，引用“逃逸” (escape) ，栈上的 `Shr` 项反映了这一点。

了解了 `Retag` 如何与裸指针交互后，你现在可以返回到 `demo2`，并且应该能够完整地解释为什么那个示例中栈改变了方式。

### 3.3 别名引用的情况

到目前为止，我所描述的一切都是大约一周前的样子。然而，有一个棘手的问题是我很晚才发现的，像往常一样，它最好通过一个示例来演示
—— 在完全安全代码中：

```rust,ignore
fn demo_refcell() {
  let rc: &mut RefCell<u8> = &mut RefCell::new(23u8);

  Retag(rc); // tag gets changed to `Uniq(0)`
  // We will consider the stack of the location where `23` is stored; the
  // `RefCell` bookkeeping counters are not of interest.

  // stack: [Uniq(0)]

  // Taking a shared reference shares the location but does not freeze, due
  // to the `UnsafeCell`.
  let rc_shr: &RefCell<u8> = &*rc;
  Retag(rc_shr); // tag gets changed to `Shr(Some(1))`
  // stack: [Uniq(0), Shr]; not frozen

  // Lots of stuff happens here but it does not matter for this example.
  let mut bmut: RefMut<u8> = rc_shr.borrow_mut();
  
  // Obtain a mutable reference into the `RefCell`.
  let mut_ref: &mut u8 = &mut *bmut;
  Retag(mut_ref); // tag gets changed to `Uniq(2)`
  // stack: [Uniq(0), Shr, Uniq(2)]; not frozen
  
  // And at the same time, a fresh shared reference to its outside!
  // This counts as a read access through `rc`, so we have to pop until
  // at least a `Shr` is at the top of the stack.
  let shr_ref: &RefCell<u8> = &*rc; // tag gets changed to `Shr(Some(3))`
  Retag(shr_ref);
  // stack: [Uniq(0), Shr]; not frozen


  // Now using `mut_ref` is UB because its tag is no longer on the stack.  But
  // that is bad, because it is usable in safe code.
  *mut_ref += 19;
}
```

注意 `mut_ref` 和 `shr_ref` 是别名！然而，在唯一的 `mut_ref` 已经覆盖的内存情况下创建共享引用无法使 `mut_ref` 无效。

如果按照上面的说明操作，当在创建 `shr_ref` 后重新标签它时，我们别无选择，只能将匹配 `mut_ref` 的项从栈中弹出。唉。

这让我意识到，在 `UnsafeCell` 中创建共享引用肯定非常弱。事实上，它完全等同于 
`Retag([raw])`：我们只需要确保某种类型的共享访问是可能的，但必须接受可能存在假定独占访问相同位置的存活的可变引用。

不过，单靠这一点是不够的。

我还在重新标签过程中添加了一个新的检查：在采取任何操作之前（即，在步骤 3 之前，在那一步可能会将项从栈中弹出），检查重新借用是否是多余的：
如果想要创建的新引用已经是可解引用的（因为它的项已经在栈上，并且如果条件满足，栈就已经冻结），并且如果证明这一点的项也是“派生自”对应于旧引用的项，那么我们就不做任何事情。

在这里，“派生自” (derived from) 的意思是“在栈中更靠上”。基本上，重新借用已经发生，新引用也已经准备好可以使用；由于“派生自”检查，我们知道使用新引用不会将对应于旧引用的项从栈中弹出。

此时，我们避免了弹出任何内容，以保持其他引用有效。

这条规则似乎永远不满足，因为新标签如何与栈中已有的标签匹配？这对于 `Uniq` 标签来说确实是不可能的，但是对于 `Shr` 标签来说，匹配更自由。

例如，在上面的示例中，从 `mut_ref` 中创建 `shr_ref` 时，此规则满足。我们不需要冻结（因为有一个 `UnsafeCell`），而栈上已经有一个
`Shr`（所以新的引用是可解引用的），并且匹配旧引用的项 `Uniq(0)` 在那个 `Shr` 的下面（所以在使用新引用之后，旧的引用仍然是可解引用的）。

因此，我们什么都不做，将 `Uniq(2)` 保留在栈上，以便让末尾通过 `mut_ref` 进行的访问保持有效。

这可能听起来像是一个奇怪的规则，事实的确如此。如果 `RefCell`  不强迫我们处理这里的情况，我肯定不会想到这一点。

然而，正如我们将在 [第 5 节](#5-key-properties) 中看到的，它也不会破坏模型的任何重要属性（可变引用是唯一的，共享引用是不变的，但
`UnsafeCell` 除外）。

此外，在将项推送到栈时（在重新标签操作结束时），我们此时可以确保栈尚未冻结：如果它被冻结，重新借用将是多余的。

有了这个规则扩展，`Retag` 指令现在如下所示（根据 `size_of_val`，在引用覆盖的所有位置上执行）：

2. 解引用时也会执行检查。记住栈里面与标签相匹配的项的位置。
3. 额外的检查：
    * 如果新标签通过了对解引用执行的检查，并且如果使此检查成功的项在步骤 2 中记住的项 **之上**，则停止。（在步骤
      2 中，“冻结”状态被认为处于栈中每个项之上）。此位置的操作已完成。
4. 通过该引用进行实际访问时也会发生相同的操作（对于共享引用是读访问，对于可变引用是写访问）。
    * 现在不能再冻结这个位置了：如果新标签是 `Uniq`，就解冻；如果新标签是 `Shr`，并且位置已经被冻结，那么额外校验（步骤 3）就开始了。
5. 检查新标签是否为 `Shr(Some(t))` 以及位置是否在 `UnsafeCell` 内：
  * 如果这两个条件都满足，则使用时间戳 `t` 冻结位置；如果已经冻结，则不执行任何操作
  * 否则，将一个新项推送到栈上：如果标签是 `Shr(_)`，则推 `Shr`；如果标签是 `Uniq(id)`，则推 `Uniq(id)`

我对额外检查有些不满意的是，在进行读取访问时，`Shr` 项与 `Uniq`
标签匹配的规则有些重叠。这两者都支持以只读方式使用已经共享的可变引用；我更希望有一个单独的条件来实现这一点，而不是两个一起工作。

尽管如此，但总的来说，我认为这是一个令人愉快的干净模型；当然比我去年提出的要干净得多，同时也更兼容现有代码。

## 4. 与原提案的不同之处

与原提案的关键区别在于，对解引用执行的检查和对访问执行的检查不是相同的检查。

这意味着模型中有更多的“灵活的部分”，但这也意味着不再需要像最初的提议那样为 `demo1` 提供一个奇怪的特殊例外（关于从冻结位置读取数据）。

然而，这一变化的主要原因是，在进行访问时，我们只是不知道是否在 `UnsafeCell` 中，所以不能执行我们想要执行的所有检查。

因此，我也稍微重新整理了一下术语。不再有一个“重新激活” (reactivation) 操作，而是有一个“解引用”检查和“访问”操作，如上文第
[2.2](#22-解引用指针) 和 [2.3](#23-访问内存) 节所述。

除此之外，我还使共享引用和裸指针的行为更加统一。这有助于修复关于 slice 上 `iter_mut`
的失败的测试，它首先创建一个裸指针，然后创建一个共享引用：在原来的模型中，创建共享引用会使先前创建的裸指针失效。

由于更统一带来的好处，这种情况不再发生。巧合的是，我进行此更改并不是为了修复 `iter_mut`，而是因为想减少模型中不同情况的数量。
然后我意识到，在启用了完整模型的情况下，相关测试也突然通过了，调查了发生什么之后，才意识到我意外地有了一个很棒的想法。:D)

该标签现在是“类型化的” (`Uniq` vs `Shr` )，以便能够支持引用和共享指针之间的 `transmute`。这种 `transmute`
在最初的模型中是一个悬而未决的问题，在随后的讨论中，一些人提出了对它的担忧。

我鼓励所有人想出一些奇怪的、应该能够 `transmute` 的东西 ，并把它们扔给 Miri，这样就可以看看你们的用例是否被覆盖了。:)

重新标签期间的额外检查可以被视为改进了原始模型在创建新引用时所做的类似检查（如果新借用已经处于活动状态，则不会更改状态）。

最后，栈借用原提案中的“函数屏障” (function barriers) 尚未实现。这是我待办事项清单上的下一项。

<a name="5-key-properties"></a>

## 5. 两个关键属性

让我们来看看我列出的两个关键属性，它们是设计的目标，并看看模型如何保证它们在所有有效（无 UB）的执行中都是正确的。

### 5.1 可变引用是唯一的

我想在这里建立的属性是：在创建（实际上是重新标签） `&mut` 
之后，如果我们运行一些未传递引用的未知代码，然后再次使用该引用（读取或写入），则可以确定该未知代码根本没有访问可变引用背后的内存（或者有
UB）。例如：

```rust,ignore
fn demo_mut_unique(our: &mut i32) -> i32 {
  Retag(our); // So we can be sure the tag is unique

  *our = 5;

  unknown_code();

  // We know this will return 5, and moreover if `unknown_code` does not panic
  // we know we could do the write after calling `unknown_code` (because it
  // cannot even read from `our`).
  *our
}
```

验证草图如下：重新标签引用后，我们知道它位于栈的顶部，并且位置没有冻结。（额外的再借用规则不适用，因为新的
`Uniq` 标签永远不能是多余的。）对于未知代码执行的任何访问，我们知道访问不能使用这个引用的标签，因为这些标签是唯一的且不可伪造。
因此，如果未知代码访问此位置，将从栈中弹出标签。当我们再次使用这个引用时，我们知道它在栈上，因此没有被弹出。因此，不可能有来自未知代码的访问。

实际上，这个定理适用于 **任何时候**，只要我们有一个引用，就可以确保它的标签没有泄露给其他任何人，并且它指向将这个标签放在（未冻结的）栈顶部的位置。

这不仅仅是重新标签后立即出现的情况。我们知道引用在写入后位于栈的顶部，所以在下面的示例中，`unknown_code_2` 无法访问 `our`：

```rust,ignore
fn demo_mut_advanced_unique(our: &mut u8) -> u8 {
  Retag(our); // So we can be sure the tag is unique

  unknown_code_1(&*our);


  // This "re-asserts" uniqueness of the reference: After writing, we know
  // our tag is at the top of the stack.
  *our = 5;

  unknown_code_2();

  // We know this will return 5
  *our
}
```

### 5.2 （没有 `UnSafeCell` 的）共享引用是不可变的

共享引用的关键属性是：在创建（实际上是重新标签）共享引用之后，如果运行一些未知代码（如果需要的话，它甚至可以得到这个引用），
然后再次使用该引用，那么我们知道该引用指向的值没有更改。例如：

```rust,ignore
fn demo_shr_frozen(our: &u8) -> u8 {

  Retag(our); // So we can be sure the tag actually carries a timestamp

  // See what's in there.
  let val = *our;
  
  unknown_code(our);

  // We know this will return `val`
  *our
}
```

证明草图如下：在重新标签引用之后，我们知道该位置是冻结的（即使运用“额外的再借用”规则，情况也是如此）。
如果未知代码执行任何写入操作，我们知道这将解冻该位置。该位置可能会重新冻结，但仅限于当时的时间戳。当从未知代码返回后进行读取时，
这将检查位置是否至少自其标签中给出的时间戳以来被冻结，因此如果位置被未知代码解冻或重新冻结，则检查将失败。因此，未知代码不可能已写入该位置。

这里对这两个证明的一个有趣的观察是，当执行未知代码时，我们所依赖的是在每次内存访问时执行的操作。

当指针被解引用时发生的额外检查只在我们的代码中进行，而不是在外部代码中进行。因此，如果我们通过 FFI
调用某些代码，而这些代码是用一种没有“解引用”概念的语言编写的，那么在推理时就没有问题了，因为我们所关心的只是该外部代码执行的实际内存访问。

这也表明，我们可以将对指针解引用的检查视为 `Retag` 旁的另一个“影子状态操作” (shadow state operation) 
，然后这两个操作加上对内存访问的操作就是栈借用的全部内容。这在 Miri 中很难实现，因为在路径求值的任何时候都可能发生解引用，但它仍然很有趣，
并且在不允许在路径中解引用的“低级 MIR”中可能很有用。

## 6. 评价模型，以及你可以如何提供帮助

我已经在 Miri 中实现了有效性不变量 (validity invariant) 和上述模型。这揭示了标准库中的两个问题 （[#54908] 和 [#54957]，都已解决），但这两个问题都与有效性不变量有关，而不是栈借用。除了这些例外，栈借用模型通过了整个测试套件。

[#54908]: https://github.com/rust-lang/rust/issues/54908
[#54957]: https://github.com/rust-lang/rust/issues/54957

在早期版本中有更多的失败测试（如第 4 节所述），但是最终的模型接受 Miri 的测试套件所涵盖的所有代码。

此外，我编写了一系列编译失败的测试，以确保模型捕捉到关键属性的各种违反行为。

我不得不对标准库进行的最有趣的 [更改](https://github.com/rust-lang/rust/pull/56161) 是 `NonNull::From`。该函数原本是通过 `&T` 将 `&mut T` 转换为
`*const T`。这意味着最终的裸指针是从共享引用创建的，因此不能用于可变。

这篇文章的一个早期版本描述了一个允许这种行为的模型，但我认为我们实际上应该尝试排除它：毕竟，在 Rust 中，“不通过共享引用（来自共享引用的指针）进行可变”是一条古老的规则。

总体而言，我对此相当满意！我预计会有更多的麻烦，比如会遇到标准库做一些常见的或很难宣布为非法的奇怪事情，而我的模型无法合理地允许这些事情。

我认为测试套件的通过表明该模型可能非常适合 Rust。

然而，Miri 的测试套件很小，我只有一个大脑来想出反例！事实上，我真的很担心，因为不到两周的时间，我就想出了一个 `demo_refcell` ，所以我可能还错过了什么？

这是你的用武之地。请测试一下这个模型！想出一些你认为应该有用的有趣的东西（我想尤其在有趣的 `transmute`  方面，如果你愿意尝试的话，可以通过 unions
或裸指针使用类型双关），或者你可能有一些包含一些不安全代码和测试套件的箱子，来使用 Miri 运行（你有一个测试套件，对吗？）。

尝试该模型的最简单方法是 [playground](https://play.rust-lang.org/)：输入代码，选择右上角的“Tools-Miri”，你将看到它的结果。

对于太长的代码，你必须在自己的电脑上安装 Miri。Miri 依赖 nightly rustc，必须定期更新才能继续工作，因此它不太适合 crates.io。

Miri 的 [README](https://github.com/solson/miri/#running-miri) 提供了安装说明。我们仍在努力让安装 Miri 变得更容易。

如果你有什么问题，请告诉我。你可以报告问题、评论这篇文章、或者在聊天中找到我（最近，我偏爱 Zulip，那里有一个 [不安全的代码指南流](ucg)）。

安装了 Miri 之后，你就可以对一个二进制项目使用 `cargo miri` 来运行。应该完全支持依赖关系，所以你可以使用任何你喜欢的库。

然而，由于 Miri 不支持某些操作，你可能会遇到某些问题。在这种情况下，请搜索问题 [issue]，如果是新的问题，请报告。我们不能支持所有事情，但我们也许能为你的情况做些什么。

[ucg]: https://rust-lang.zulipchat.com/#narrow/stream/136281-wg-unsafe-code-guidelines
[issue]: https://github.com/solson/miri/issues

也支持 `cargo miri test`，你可以在库中使用它！

如你所见，每个人都有足够的工作要做。别害羞！我的实习只剩下两周了，之后我将不得不大幅减少 Rust 活动，以利于完成我的博士学位。

不过，我不会完全消失，别担心 —— 如果你想帮助你完成上面的任何任务，我仍然可以指导你。:)

感谢 @nikomatsakis 对这篇帖子草稿的反馈，感谢 @shepmaster 让 Miri 可以在 playground 上使用，感谢 @oli-obk 以无与伦比的速度审查了我的所有PR。

如果你想帮助或报告你的实验结果，或者有任何问题或建议，请加入论坛的 [讨论](https://internals.rust-lang.org/t/stacked-borrows-implemented/8847)。

## 更新日志

**2018-11-21**：解引用指针现在始终保留标签，但强制转换为裸指针会将标签重置为 `Shr(None)`。`Box` 被视为可变引用。

**2018-12-22**：创建共享引用不会将 `Shr` 项推送到栈（除非有 `UnsafeCell` ）。此外，创建裸指针是一种特殊的重新标签。
