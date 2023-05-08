# 栈借用：Rust 的别名模型

> 原文：[Stacked Borrows: An Aliasing Model For Rust](https://www.ralfj.de/blog/2018/08/07/stacked-borrows.html)
> | 作者：Ralf Jung | 2018 年 8 月 17 日 | [IRLO]

[IRLO]: https://internals.rust-lang.org/t/stacked-borrows-an-aliasing-model-for-rust/8153

在这篇文章中，我提出了“栈借用”：一组在 Rust 中允许哪种别名的规则。

这是为了回答何时可以使用哪种指针来执行哪种内存访问的问题。

这是一个长期未解决的问题，许多不安全代码作者以及想要添加更多优化的编译器作者都会遇到。

我并不是第一次尝试给目前这个模型下定义：该模型在很大程度上基于 [@arielb1] 和 [@ubsan]
的想法，当然还考虑到了我 [去年学到的] 经验教训，当时我第一次尝试定义一个叫 [Types as Contracts] 的模型。

[@arielb1]: https://github.com/nikomatsakis/rust-memory-model/issues/26
[@ubsan]: https://github.com/nikomatsakis/rust-memory-model/issues/28
[去年学到的]: https://www.ralfj.de/blog/2017/08/11/types-as-contracts-evaluation.html
[Types as Contracts]: https://www.ralfj.de/blog/2017/07/17/types-as-contracts.html

但在我深入研究我最新提议之前，我想简要讨论一下以前的模型与这个模型之间的一个关键区别：
* Types as Contracts 是一个完全基于“有效性” (validity) 的模型
* 而 Stacked Borrows 是（在某种程度上）基于“可达性” (access) 的模型。

> 更新：发布这篇文章之后，我又写了另一篇[^another] 博客文章，内容是关于栈借用的一个略微调整的版本（也是第一个实际实现的版本）。
> <u>**那篇文章是独立的，所以如果你只对栈借用的现状感兴趣，我建议你去看那篇。如果你想了解更多的历史背景，请继续阅读。**</u>

[^another]: 译者注：那篇原文链接 [在此][implementation]，翻译 [在此][implementation-zh]。

[implementation]: https://www.ralfj.de/blog/2018/11/16/stacked-borrows-implementation.html
[implementation-zh]: ./stacked-borrows-implementation.md

## 1. 基于有效性与基于可达性

基于“可达性” (access) 的模型是这样一种模型，其中某些性质 (properties) 只有在引用实际用于
**访问** 内存时才被强制实施 —— 在本例中，可变引用是唯一的，并且共享引用指向只读内存。

而基于“有效性” (validity) 的模型要求这些性质始终适用于所有 **可能** 被使用的引用。

在这两种情况下，违反模型要求的性质都是未定义的行为 (undefined behavior) 。

其实类似于 Types as Contracts 的基于有效性的模型，其基本思想是根据给定的类型，其所有数据总是有效的。
这样的模型势必带来限制，因为这相当于急切地检查所有可到达的数据的有效性（例如，当检查程序是否具有未定义的行为时）。

另一方面，基于可达性的模型只要求数据在使用时有效。实施它相当于惰性检查每一次操作的最低限度。

基于有效性的模型有几个优点：
* 急切地检查意味着我们通常可以确定哪些代码实际上负责产生“坏”值
* “所有数据必须始终有效”也比一长串操作及其对数据施加的限制更容易解释

然而，在 Rust 中，我们不能在不讨论生命周期的情况下讨论引用以及在给定类型下引用是否有效。

在 Types as Contracts 的情况下，生命周期的确切终点变得非常重要。这不仅使规范变得复杂和难以理解，而且 Miri
中的实现还必须积极与编译器通常尽早忘记生命周期的意图背道而驰。对于非词法生命周期，生命周期的“结束”甚至不再是那么明确被定义。

## 2. 栈借用

出于这些原因，我的第二个提议是使生命周期，特别是生命周期推断的结果，与程序是否具有未定义行为 (UB) 完全无关。这是设计的核心目标之一。

如果你需要更多关于未定义行为以及它与编译器优化相联系的背景，我建议你先阅读我写的关于这个主题的
[博客文章][UB]。那不是一个很长的帖子，我不会在这里重复所有的事情：)

[UB]: https://www.ralfj.de/blog/2017/07/14/undefined-behavior.html

这个栈借用模型（以及 @arielb1 和 @ubsan 提出的前身）的核心思想是，对于每个位置，我们跟踪允许访问该位置的引用。

（稍后我将讨论如何跟踪这一点；现在，让我们假设它是可以做到的。）

这形成了一个栈：
* 当有一个 `&mut i32` 时，我们可以重新借用 (reborrow) 它来获得新的引用
* 这个新引用现在必须用于该位置的引用，但创建该引用的旧引用不能被忘记
* 在某一时刻，重新借用终止，并且旧引用将再次“激活”
* 在该栈上还会有其他东西，因此将写 `Uniq(x)`，以表示 `x` 是允许访问该位置的唯一引用

来看一个例子：

```rust
fn demo0(x: &mut i32) -> i32 {
  // At the beginning of the function, `x` must be the "active" reference
  // for the 4 locations it points to, meaning `Uniq(x)` is at the top of the stack.
  // (It's 4 locations because `i32` has size 4.)
  let y = &mut *x; // Now `Uniq(y)` is pushed onto the stack, as new active reference.
  // The stack now contains: Uniq(y), Uniq(x), ...
  *y = 5; // Okay because `y` is active.
  *x = 3; // This "reactivates" `x` by popping the stack.
  // The stack now contains: Uniq(x), ...
  *y // This is UB! `Uniq(y)` is not on the stack of borrows, so `y` must not be used.
}
```

当然，此示例不会编译，因为借用检查器会抱怨。然而，在我的解释中，它抱怨的原因是，如果它接受了这个程序，安全代码中就会有 UB！

这里值得琢磨一番：该模型定义了不考虑生命周期的程序语义，因此我们可以运行程序并询问它们是否具有
UB，而无需进行生命周期推断或借用检查（与 Types as Contracts 非常不同）。

因此，一个重要的性质是，如果程序有 UB，并且没有使用任何不安全的代码，那么借用检查器必须检测到这一点。

在某种意义上，我的模型定义了一个动态版本的借用检查器，它可以在不使用生命周期的情况下工作。

事实证明，即使在非词法生命周期内，给定位置的借用结构仍然是嵌套良好的 (well-nested)，这就是为什么我们可以将借用安排到栈里。

### 2.1 裸指针

让我们通过在程序中添加一些不安全的代码来绕过借用检查器：

```rust
fn demo1(x: &mut i32) -> i32 {
  // At the beginning of the function, `x` must be the "active" reference.
  let raw = x as *mut _; // Create raw pointer
  // The stack now contains: Raw, Uniq(x), ...
  let y = unsafe { &mut *raw }; // Now `y` is pushed onto the stack, as new active reference.
  // The stack now contains: Uniq(y), Raw, Uniq(x), ...
  *y = 5; // Okay because `y` is active.
  *x = 3; // This "reactivates" `x` by popping the stack twice.
  *y // This is UB! `Uniq(y)` is not on the stack of borrows, so `y` must not be used.
}
```

这里我们将 `x` 转换为裸指针。

我们无法真正跟踪裸指针是在哪里以及如何创建的 —— 裸指针可以安全地与整数相互转换，数据可以任意流动。

因此，当 `&mut` 被转换为 `*mut` 时，我们改为将 `Raw` 推到栈上，这表明 **任何** 裸指针都可以来访问该位置。

（关于跨分配地址计算的常见限制仍然适用，我在这里只是在谈论借用检查。）

在下一行，我们使用裸指针来创建 `y`。这没有关系，因为 `Raw` 处于活动状态。像往常一样，当创建引用时，我们将其推到栈上。

这使 `y` 变成活动的引用，因此我们可以在下一行中使用它。同样，使用 `x` 将弹出栈，直到 `x` 处于活动状态 ——
在本例中，这将同时删除 `Uniq(y)` 和 `Raw`，使 `y` 不可用，并导致最后一行的 UB。

让我们看另一个涉及裸指针的例子：

```rust,ignore
fn demo2(x: &mut i32) -> i32 {
  // At the beginning of the function, `x` must be the "active" reference.
  let raw = x as *mut _; // Create raw pointer
  // The stack now contains: Raw, Uniq(x), ...
  let y = unsafe { &mut *raw }; // Now `y` is pushed onto the stack, as new active reference.
  // The stack now contains: Uniq(y), Raw, Uniq(x), ...
  *y = 5; // Okay because `y` is active.
  unsafe { *raw = 5 }; // Using a raw pointer, so `Raw` gets reactivated by popping the stack!
  // The stack now contains: Raw, Uniq(x), ...
  *y // This is UB! `Uniq(y)` is not on the stack of borrows, so `y` must not be used.
}
```

因为裸指针是在栈上跟踪的，所以它们必须遵循良好的嵌套结构。

`y` 是从 `raw` 创建的，所以再次使用 `raw` 会使 `y` 无效！

这与第一个例子完全一致，在第一个例子中，`y` 是从 `x` 创建的，所以使用 `x` 再次使 `y` 无效。

### 2.2 共享引用

当然，对于共享引用，有权访问的引用并不是唯一的。

我们必须建模的关键性质是 **共享引用指向不变的内存**（假设不涉及内部可变性）。可以说，内存是冻结的 (frozen)。

为此，我们使用某种“时间戳”来标记共享引用，以表明它在何时创建。

针对每个位置还有一个额外的标志 (flag)，记录自从何时以来该位置被冻结。

如果自创建共享引用以来内存已一直冻结，则可以使用共享引用访问内存。

下面的示例中看到这一点：

```rust,ignore
fn demo3(x: &mut i32) -> i32 {
  // At the beginning of the function, `x` must be the "active" reference.
  let raw = x as *mut _; // Create raw pointer
  // The stack now contains: Raw, Uniq(x), ...
  let y = unsafe { & *raw }; // Now memory gets frozen (recording the timestamp)
  let _val = *y; // Okay because memory was frozen since `y` was created
  *x = 3; // This "reactivates" `x` by unfreezing and popping the stack.
  let z = &*x; // Now memory gets frozen *again*
  *y // This is UB! Memory has been frozen strictly after `y` got created.
}
```

具有内部可变性的共享引用在内存方面实际上没有任何限制，所以我们基本上将它们视为裸指针。

### 2.3 小结

对于内存中的每个位置，我们跟踪一个借用栈 （`Uniq(_)` 或 `Raw`），并可能通过冻结该位置来结束栈。冻结位置不会被写入，也不会推送 `Uniq`。

每当创建可变引用时，该引用“所涉及”的每个位置 —— 也就是使用该引用访问的位置(从它指向的位置开始，然后到
`SIZE_OF_VAL` 个字节）—— 都会有相应的 `Uniq` 被推送到栈上。

每当创建共享引用时，如果没有内部可变性，我们将冻结位置（如果它们尚未冻结）。如果存在内部可变性，我们只需推送一个 `Raw`。

每当从可变引用创建裸指针时，我们都会推送一个 `Raw`。（从共享引用创建裸指针时不会发生任何操作。）

如果一个位置没有被冻结并且 `Uniq(x)` 位于栈的顶部，则可变引用 `x` 对于该位置是“活动的”。

如果这个位置至少自创建以来已被冻结，则不具有内部可变性的共享引用处于活动状态。

如果 `Raw` 位于栈顶部，则具有内部可变性的共享引用处于活动状态。（译者注，如前所述具有内部可变性的共享引用与裸指针基本相同）

每当引用被用来做一些事情（或任何事情）时，我们确保它对于涉及的所有位置都是活动的；这可能包括解除冻结和弹出栈。
如果不能以这种方式重新激活引用[^reactivate]，则有 UB。

[^reactivate]: 我之前说的“激活” (activate) 引用，而不是现在的“重新激活它”
(reactivate)；我决定更改术语，以便更清楚地表明这是再次使用的旧引用，因此它一定已经在栈上（但可能不在顶部）。

## 3. 跟踪借用

到目前为止，我只是假设以某种方式在上面代码中的引用 `x` 和栈上的 `Uniq(x)` 相联系。
我还说过，我们正在跟踪共享引用是何时创建的。要实现这一点，需要以某种方式将信息“标记” (tag) 到引用。

特别要注意，第一个示例中的 `x` 和 `y` 具有相同的地址。如果我们将它们作为裸指针进行比较是相等的。然而，如果我们使用 `x` 或 `y`，就会产生巨大的差异！

如果你读过我上一篇关于 [为什么指针很复杂][pointers-complicated] 的文章，这应该不会太令人惊讶。

[pointers-complicated]: https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html

除了指向内存中的地址之外，指针或引用（我通常互换地使用这两个术语）还有更多的含义。

为了达到此模型的目的，我们假设引用类型的值由两部分组成：内存中的地址和引用创建时间的标记。

“时间”在这里是一个相当抽象的概念，我们其实只需要某种计数器，每次创建新的引用时这种计数器都会出现。

这为每个可变引用提供了唯一的 ID —— 正如我们已经看到的，对于共享引用，实际上利用了 ID
是按递增顺序分发的这一事实（这样就可以测试引用是在位置冻结之前还是之后创建）。

因此，我们实际上可以统一处理可变引用和共享引用，因为两者都只在标记中记录创建它们的时间。

每当我在上文里提到在栈上有 `Uniq(x)` 时，我真正的意思是在栈上有 `Uniq(t_x)`，其中 `t_x`
是某个时钟值 (clock value) ，而 `x` 的“标签”是 `t_x`。为了便于阅读，我将继续使用下面的 `Uniq(x)` 表示法。

由于未跟踪裸指针，因此我们可以在将引用转换成裸指针时擦除标记。这意味着标记不会干扰指针-整数之间的转换，也意味着不必担心一大堆复杂的问题：)

当然，这些标记并不存在于真正的硬件上。但这不是重点。

当指定程序行为时，我们可以使用携带机器上不存在的额外状态的 “[仪表型机器][instrumented-machine]”，
只要只使用该额外状态来定义程序是否 UB：在真实的硬件上，我们可以忽略 UB 的程序（它们可以做任何事情），所以额外的状态并不重要。

[instrumented-machine]: https://www.ralfj.de/blog/2017/06/06/MIR-semantics.html

标记是我在“Types as Contracts”中想要避免的东西 —— 这是我最初给自己设置的设计约束之一，希望避免“复杂指针”带来的麻烦。

然而，我现在得出的结论是，如果标记指针意味着我们可以让生命周期变得无关紧要，那么标记指针是值得付出的代价。

## 4. 重新标记和屏障

我希望你现在对我提出的模型的基本结构有一个清晰的了解：借用栈 (stack of borrows) 、冻结标志 (freeze flag) 和标记创建时间的引用 (tag)。

完整的模型没有那么简单，但也不会复杂太多。我们只需要再添加两个概念：重新标记 (retag) 和屏障 (barriers)。

### 4.1 重新标记

请记住，每次创建可变借用时，都会将当前时钟值作为其标记分配给它。由于标记永远不能更改，这意味着两个不同的变量永远不能有相同的标记
—— 对吗？不幸的是，事情并不是那么简单：例如，使用 [`transmute_copy`] 或 `union`，人们可以以一种 Rust 甚至没有注意到的方式来复制引用。

[`transmute_copy`]: https://doc.rust-lang.org/stable/std/mem/fn.transmute_copy.html

尽管如此，我们还是想对代码做出如下说明：

```rust,ignore
fn demo4(x: &mut i32, y: &mut i32) -> i32 {
  *x = 42;
  *y = 7;
  *x // Will load 42! We can optimize away the load.
}
```

问题是，我们无法阻止函数外部传递具有相同标签的虚假的 `&mut`。

这是否意味着我们在创建带别名的可变引用 UB 方面又回到了起点？

幸运的是，我们没有！有很多机制可供使用，我们只需稍微调整一下即可。

我们要做的是，每次引用进入函数（它可以是一个函数参数，但也可以从内存中加载它或将其作为某个其他函数的返回值得到）时，执行“重新标记”：
**将可变引用的标记更改为当前时钟值，在分配的每个标记之后增加时钟，然后将那些新标记推到借用栈的顶部**。

这样，我们就可以知道 —— 不需要对外来代码做任何假设 —— 所有引用都有不同的 ID。具体地说，对于同一位置，两个不同的引用永远不能同时都是“活动的”。

有了这一额外的步骤，当 `x` 和 `y` 作为别名时（译者注：其后数据都指向同一个值），现在很容易证明上面的 `demo4`
是 UB，无论它们的初始标签是什么：使用 `x` 之后，我们知道它是活动的；接下来，使用并重新激活 `y`，它必须弹出 
`Uniq(x)`，因为它们有不同的标签；最后，`x` 已不在栈中却再次使用 `x`，会触发 UB。

（`Uniq` 只有在创建时才会被推到栈上，所以它在栈里永远不会超过一次。）

### 4.2 屏障

我还想补充一个概念：屏障 (barriers)。即使没有屏障，这个模型也很有意义 ——
但增加屏障这一概念排除了更多我认为不想允许的行为。

还需要解释一点：为什么可以在生成 LLVM IR 时，将 [`noalias`] 参数属性放在函数上。

[noalias]: https://llvm.org/docs/LangRef.html#parameter-attributes

请考虑以下代码：

```rust,ignore
fn demo5(x: &mut i32, y: usize) {
  *x = 42;
  foo(y);
}

fn foo(y: usize) {
  let y = unsafe { &mut *(y as *mut i32) };
  *y = 7;
}
```

问题是：我们可以将 `*x = 42;` 重新排序到 `demo5` 的末尾吗？注意，我们没有再次使用 `x`，所以我们不能假设 `demo5`
末尾的 `x` 是活动的！这是基于可达性模型的常见问题。

然而，可以想象一下，有人可能会把 `x as *mut _ as usize` 作为 `y` 来调用 `demo5`，这意味着重新排序可能会改变程序行为。
要解决这个问题，我们必须确保如果有人以这种方式调用 `demo5`，即使 `x` 没被重复使用，也会有 UB。

为此，我建议通过施加一些额外的限制，将方向盘更多地转向基于有效性的模型。

我们希望确保将整数 `y` 转换为引用不会从栈中弹出 `x` 并继续执行程序（否则我们想要 UB）。如果栈中某个位置包含 
`Raw`，则可能会发生这种情况。请记住，我们不标记裸指针，因此当创建 `x` 涉及到裸指针时，该 `Raw`
项仍将位于栈上，从而允许任何裸指针访问该位置。这有时是至关重要的，但在这种情况下，`demo5`
应该能够防止那些涉及创建 `x` 的旧借用被重新激活。

其想法是在调用 `demo5` 时在所有函数参数的栈中放置一个“屏障”，并把在 `demo5`
返回之前从栈中弹出该屏障变成 UB。这样，栈中更底部的所有借用（`Uniq(x)` 以下）都被临时禁用，并且在
`demo5` 运行时无法重新激活。这意味着即使 `y` 恰好是 `x` 指向的内存地址，将 `y` 转换为引用就是 UB，因为 `Raw` 项不能被重新激活。

另一种思考屏障的方式是：该模型通常忽略生命周期，不知道它们能持续多久。我们所知道的是，当引用被使用时，其生命周期必须还在，所以称这是重新激活借用的时候。

最重要的是，屏障编码了这样一个事实，即当引用作为参数传递给函数时，其生命周期（无论它是什么）将延长到当前函数调用之外。

在我们的示例中，这意味着在运行 `demo5` 时，不能使用堆栈更靠顶部的借用（这些是生命周期更长的借用）。

屏障与重新编号 (renumbering) 相结合的一个很好的副作用是，即使上一小节中的 `demo4` 根本不使用它的参数，用两个别名引用来调用它仍是
UB：当重新编号 `x` 时，我们正在推动一个屏障。重新编号 `y` 会尝试重新激活 `Uniq(y)`，但它只能在屏障后面，因此无法重新激活。

## 5. 代码中的模型

现在所有的东西都在一起了。我将尝试以伪 Rust 代码的形式给出模型的另一种更精确的描述，而不是进行概括。

这基本上是一个代码草案，有望很快出现在 Miri 中，来实际动态跟踪借用栈并执行规则。这也是我开发这样的模型的方式 ——
我发现使用某种形式的伪 Rust 比用单纯的英语更容易精确描述。到目前为止，高级别的描述中已经省略了一些细节，它们应该都在这段代码中。

如果你只对高级别的描述感兴趣，可以跳到最后。剩余的内容更像是一份规范，而不是一篇解释性的博客文章。

好事是，即使有了规范，这篇文章仍然比引入“Types as Contracts”的那篇文章要短：)

### 5.1 对每个地点的操作

假设有一个类型 `MemoryByte` 存储了内存中每个位置的信息，并用于放置借用栈和冻结的信息：

```rust
/// Information about a potentially mutable borrow
enum Mut {
  /// A unique, mutable reference
  Uniq(Timestamp),
  /// Any raw pointer, or a shared borrow with interior mutability
  Raw,
}
/// Information about any kind of borrow
enum Borrow {
  /// A mutable borrow, a raw pointer, or a shared borrow with interior mutability
  Mut(Mut),
  /// A shared borrow without interior mutability
  Frz(Timestamp)
}
/// An item in the borrow stack
enum BorStackItem {
  /// Defines which references are permitted to mutate *if* the location is not frozen
  Mut(Mut),
  /// A barrier, tracking the function it belongs to by its index on the call stack
  FnBarrier(usize)
}

struct MemoryByte {
  borrows: Vec<BorStackItem>, // used as a stack
  frz_since: Option<Timestamp>,
  /* More fields, to store the actual value and what else might be needed */
}
```

接下来，我们定义一些对每个位置的操作，稍后使用这些操作来定义使用引用时怎么做。

下面的 `assert!` 用来表示由于解释器不变量（即如果不变量不能保持不变，Miri 将报错）而应该总是为真的东西，而 `bail!` 用来表示程序有 UB。

```rust,ignore
impl MemoryByte {

  /// Check if the given borrow may be used on this location.
  fn check(&self, bor: Borrow) → bool {
    match bor {
      Frz(acc_t) =>
        // Must be frozen at least as long as the `acc_t` says.
        self.frz_since.map_or(false, |loc_t| loc_t <= acc_t),
      Mut(acc_m) =>
        // Raw pointers are fine with frozen locations. This is important because &Cell is raw!
        if self.frozen_since.is_some() {
          acc_m.is_raw()
        } else {
          self.borrows.last().map_or(false, |loc_itm| loc_itm == Mut(acc_m))
        }
    }
  }

  /// Reactivate the given existing borrow for this location, fail if that is not possible.
  fn reactivate(&mut self, bor: Borrow) {
    // Do NOT change anything if `bor` is already active -- in particular, if
    // it is a `Mut(Raw)` and we are frozen.
    if self.check(bor) { return; }
    let acc_m = match bor {
      Frz(acc_t) => bail!("Location should be frozen but it is not"),
      Mut(acc_m) => acc_m,
    };
    // We definitely have to unfreeze this, even if we use the topmost item.
    self.frozen_since = None;
    // Pop until we see the one we are looking for.
    while let Some(itm) = self.borrows.last() {
      match itm {
        FnBarrier(_) => {
          bail!("Trying to reactivate a borrow that lives behind a barrier");
        }
        Mut(loc_m) => {
          if loc_m == acc_m { return; }
          self.borrows.pop();
        }
      }
    }
    bail!("Borrow-to-reactivate does not exist on the stack");
  }

  /// Initiate the given (new) borrow for the location.
  /// This is "pushing to the stack", except that it also handles initiating a `Frz`.
  fn initiate(&mut self, bor: Borrow) {
    match bor {
      Frz(t) => {
        if self.frozen_since.is_none() {
          self.frozen_since = Some(t);
        }
      }
      Mut(m) => {
        if m.is_uniq() && self.frozen_since.is_some() {
          bail!("Must not initiate Uniq when frozen!");
        }
        self.borrows.push(Mut(m));
      }
    }
  }

  /// Reset the borrow tracking for this location.
  fn reset(&mut self) {
    if self.borrows.iter().any(|itm| if let FnBarrier(_) = item { true } else { false }) {
      assert!("Cannot reset while there are barriers");
    }
    self.frozen_since = None;
    self.borrows.clear();
  }
}
```

### 5.2 MIR 操作

最后，我们按照我上面描述的模型，通过文字记录来增强一些 MIR 操作。这就是代码变得更“伪”和更少 Rust 的地方。

对于每个操作，我们都迭代所有受影响的位置；把循环变量 `loc` 的类型记为 `MemoryByte`。还有一个变量
`tag`，其中包含正在操作的指针的标记（加载、存储或转换为裸指针等）。

此外，还有一个 bool 变量 `in_unsafe_cell`，表示根据指针的类型，当前正在处理的位置是否涉及
[`UnsafeCell`]。（这实现了检查是否具有内部可变性的条件。）例如，在 `&Cell<i32>` 中，所有 4 个位置都在一个
`UnsafeCell` 中。然而，在 `&(i32, Cell<i32>)` 中，只有 8 个位置中的最后 4 个在 `UnsafeCell` 中。

[`UnsafeCell`]: https://doc.rust-lang.org/stable/std/cell/struct.UnsafeCell.html

最后，给定引用类型、标签以及是否在 `UnsafeCell` 中，我们可以计算相应的 `Borrow`：
* 可变引用使用 `Mut(Uniq(tag))`
* `UnsafeCell` 中的共享引用使用 `Mut(Raw)`
* 其他共享引用使用 `Frz(tag)`

用 `bor` 来表示正在处理的指针的 `Borrow`。现在，我们可以看看每个操作发生了什么。

* 直接使用裸指针被脱糖为创建共享引用（在读取时）或可变引用（在写入时），并使用它。然后：
* 任何时候使用（可变或共享）引用来访问内存，以及任何时候传递给“外部世界”的引用
  （如将其传递给函数、存储在内存中、返回给调用者；也在结构体或枚举体之下，但不在 unions 或间接指针之下），我们重新激活
    * `loc.reactive(borrow)`
* 每当创建 **新的** 引用（任何时候执行表达式 `&mut foo` 或 `&foo`）时，我们（重新）借用
    * 增加时钟，并将旧时间记住为 `new_tag`
    * 根据 `new_tag` 和要创建的引用的类型计算 `new_bor`
    * 
      ```rust,ignore
      if loc.check(new_bor) {
        // 新借用已处于活动状态！之所以会发生这种情况，是因为可变引用可以多次共享。
        // 我们不做其他任何事情。特殊例外是，我们不会重新激活 bor，即使它是“已使用”的，因为这将位置解除冻结！
      } else {
        // 我们可能正在创建对局部变量的引用。在这种情况下，使用 `loc.reset()`。否则，`reactive(bor)`。
        initiate(new_bor)
      }
      ```
  * 使用 `new_tag` 作为新引用。
* 任何时候从“外部世界”传递给我们的引用（如作为函数参数、从内存加载或从被调用者返回；并在结构体或枚举体下面，但不在
  unions 或间接指针下面），我们都会重新标记。
    * 增加时钟，并记住旧时间为 `new_tag`
    * 根据 `new_tag` 和正在创建的引用的类型计算 `new_bor`
    * `reactive(bor)`
    * 如果这是传入的函数参数，则 `loc.borrows.push(FnBarrier(stack_height))`

    * `initiate(new_bor)`。注意，如果 `new_bor` 已经处于活动状态，那么这是个空操作。
      尤其是，如果位置被冻结，并且这是具有内部可变性的共享引用，则不会在屏障之上推送任何内容。
      这一点很重要，因为在屏障仍然存在时，任何解除冻结该位置的重新激活都一定导致 UB。
    * 将引用标签更改为 `new_tag`
* 每次从引用创建裸指针时，都可能需要执行原始的重新借用
    * `reactivate(bor)`
    * `initiate(Mut(Raw))`，当这来自于共享引用，那么这是个空操作
*每当函数返回时，我们都必须清除屏障。
    * 迭代所有内存并移除相应的 `FnBarrier`。这就是“栈”变得有点像谎言的地方，因为我们还清除了栈中间的屏障。
      这可以通过添加一个间接操作来优化，所以只需要在某个地方记录这个函数调用已经结束。
* 任何时候释放内存，都被算作修改 (mutation)，所以通常的规则适用于此。在此之后，释放的字节栈不能包含任何屏障。

如果你想测试自己对“栈借用”的理解，我建议你到 [Types as Contracts 的第 2.2 节][TaC-example]，查看那里的三个示例。忽略
`Valiate` 调用，该部分不再相关。这些都是我们希望有效的优化示例，事实上，这三个优化在“栈借用”中仍然有效。你能解释一下为什么会是这样吗？

[TaC-example]: https://www.ralfj.de/blog/2017/07/17/types-as-contracts.html#22-examples

## 总结

我已经描述了另一个 Rust 内存模型，该模型定义了何时可以使用引用来执行哪些内存操作。

此模型的主要设计约束是生命周期不应对程序执行产生影响。令我惊讶的是，考虑到所有因素，这个模型实际上最终相当简单。

我想我已经涵盖了大部分相关功能，不过我将不得不更仔细地研究两阶段借用 (two-phase borrows)，看看它们是否需要对模型进行一些进一步的更改。

当然，现在最大的问题是这个模型是否真的“能工作” ——
它是否允许我们想要允许的所有代码（甚至允许所有安全代码），以及它是否排除了足够的代码，以便我们可以获得有用的优化？

我希望在接下来的几周里通过实现一个动态检查器来进一步探讨这个问题，以在实际代码上测试该模型。

当你不必在每个微小的更改后手动重新评估所有示例时，回答这些问题就更容易了。

但是，我总是可以使用更多的例子，所以如果你认为你发现了一些有趣的或令人惊讶的角落案例，请让我知道！

像往常一样，如果你有任何问题或意见，请随时在 [论坛上提问][IRLO]。
