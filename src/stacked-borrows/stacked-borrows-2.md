# 【版本 2】栈借用

> 原文： [Stacked Borrows 2](https://www.ralfj.de/blog/2019/04/30/stacked-borrows-2.html)
> | 作者：Ralf Jung | 2019 年 04 月 30 日 | [IRLO]

[IRLO]: https://internals.rust-lang.org/t/stacked-borrows-2/9951

最近，我 [大幅更新](https://github.com/rust-lang/miri/pull/695) 了 Stack Borrows，以修复上一版本中未涉及的处理共享引用方面的一些问题。

在这篇文章中，我将描述新版本是什么样子，以及它与 [Stacked Borrow 1](./stacked-borrows-implementation.md) 有何不同。

我假定你对以前的版本有所了解，所以我不会从头开始解释所有内容。

## 回顾

在我们开始之前，让我对到目前为止已经写过的关于栈借用的文章做一个简单的概述。我没有事先计划好这件事，所以事情比我想的要糟糕得多。

* [Stacked Borrows: An Aliasing Model For Rust](./stacked-borrows.md)
  是本系列的第一篇文章，描述了我在开始实现它们之前对栈借用的初步想法。它给出的一些背景可能很有趣，但在很大程度上被下一篇文章所取代。
* [Stacked Borrows Implemented](./stacked-borrows-implementation.md) 描述了栈借用的第 1 
  个版本，以及我实施它的经验。它是独立的；我对我在最初帖子中的一些解释不满意，所以我决定再试一次。如果你还在了解栈借用，这是一篇值得一读的文章。
* [Barriers and Two-phase Borrows in Stacked Borrows](./stacked-borrows-barriers.md) 描述了我是如何扩展栈借用版本
  1、部分支持两阶段借用，并解释了“屏障”的概念。除了特别提到屏障和两阶段借款的部分，你不一定要读过这篇文章才能理解栈借用版本 2。

> 译者注：由于这篇文章开始出现了版本的概念，为了减少不必要的名词翻译，我使用 SB 来表示栈借用 (stacked borrows)，使用
> SB1 和 SB2 表示它的第一个和第二个版本。

## 问题

我想用 SB2 解决的问题是，SB1 只对共享引用执行了很少的跟踪。我的想法是，如果位置无论如何都是只读的，那么授予任何人读访问权限都不会有什么坏处。

然而，正如 [@arielby 所指出的](https://github.com/rust-lang/unsafe-code-guidelines/issues/87)，
在函数接收到可变引用（它应该没有别名），然后从这个可变引用创建共享引用的情况下，这会导致丧失潜在的优化：

```rust
fn main() {
    let x = &mut 0u32;
    let p = x as *mut u32;
    foo(x, p);
}

fn foo(a: &mut u32, y: *mut u32) -> u32 {
    *a = 1;
    let _b = &*a; // This freezes `*a`. Frozen locations can be read by any raw pointer.
    let _val = unsafe { *y }; // Hence, this legal in Stacked Borrows.
    *a = 2; // But we might want to drop the earlier `*a = 1` because it gets overwritten!
    _val
}
```

SB1 允许任何原始指针读取被冻结的位置。基本上，一旦一个位置被冻结，我们就不会跟踪哪个指针被允许读取；此时，我们允许所有指针读取。

然而，这意味着上面的 `&*a` （重新借用可变引用作为共享引用）有一个意外的副作用，即允许原始指针读取 `*a`！

这违反了 `a` 是唯一指针的想法：我们想删除第一次写入 (`*a = 1`)，因为它稍后会被覆盖，同时没有通过 `a` 进行读取。问题是，通过 `y`
进行别名读取，因此删除看似多余的写入会将 `foo` 的返回值从 `1` 改为 `0`。因此，像我们在这里使用指向同一事物的 `a` 和 `y` 来调用
`foo` 肯定是未定义的行为，然而，SB1 认为上面的例子是合法的。

## 解决方案

为了解决这个问题，我不得不用其他东西取代“冻结”的机制。请记住，在 SB1 中，当创建共享引用时，我们存储了它发生的时间戳，
并在内存中记录了该引用所指向的位置从现在起一直处于冻结状态。每当使用共享引用时，我们都会检查该位置是否至少自创建引用以来被冻结。
这与可变引用不同，在可变引用中，借用栈中需要存在该引用唯一确切的 ID。

为了排除类似上述示例的情况，我们不冻结位置来允许此后创建的所有共享引用访问该位置，而是精确跟踪哪种共享引用允许访问。

每个位置的借用栈现在由以下项组成，这些项将特定的权限授予被特定标签的标识的指针：

```rust
pub struct Item {
    /// The pointers the permission is granted to.
    tag: Tag,
    /// The permission this item grants.
    perm: Permission,
}

pub enum Permission {
    /// Grants unique mutable access.
    Unique,
    /// Grants shared mutable access.
    SharedReadWrite,
    /// Greants shared read-only access.
    SharedReadOnly,
}
```

不再有单独的“冻结”状态；并且现在由借用栈本身处理将共享引用保持为只读状态的情况：

```rust,ignore
pub struct Stack {
    borrows: Vec<Item>
}
```

标签也比以前更简单：不再对可变引用和共享引用单独标签。

```rust,ignore
pub type PtrId = u64;
pub enum Tag {
    Tagged(PtrId),
    Untagged,
}
```

就像以前一样，每次重新标签时都会选择新的 `Tag`；特别是每当使用 `&mut <expr>` 和 `& <expr>` 创建引用时，以及当引用被强制转换为裸指针时。

除此之外（比如指针运算），传播标签只用来跟踪该指针来自哪里。然而，与以前不同的是，可变引用和共享引用之间的唯一区别是与该标签相关联的权限。

### 内存访问

但在我们开始创建新引用之前，让我们看看借用栈中的权限如何影响访问内存时所发生的事情。

假设某个标签为 `tag` 的指针用于从内存读取或向内存写入。对于每个受到影响的位置，会经历两个步骤：首先尝试找到授予权限的项
(granting item) ，然后删除不兼容的项 (incompatible item)。

#### 找到授予权限的项

为了找到授予权限的项，我们从上到下遍历借用栈，并搜索具有相同标签的项。如果它是写访问，则继续查找，直到找到正确标签的项，其权限为
`SharedReadWrite` 或 `Unique` —— 我们认为具有 `SharedReadOnly` 权限的项不授予写访问权限。这个项授予标签为 `tag`
的指针执行给定访问的权限，因此被叫做“授予权限的项”（在 SB1 中，相同的概念被称为“匹配项” (matching 
item)）。一旦找到授予权限的项，我们就会记住它在栈中的位置；这对第二步很重要。如果找不到授予权限的项，则访问会导致未定义的行为。

例如，如果栈是

```rust,ignored
[ Item { tag: Tagged(0), perm: Unique },
  Item { tag: Tagged(1), perm: SharedReadOnly} ]
```

那么索引 1 处的项授予标签为 `Tagged(1)` 的读访问；索引 0 处的项授予标签为 `Tagged(0)` 的读/写访问，但是任何项都不授予标签为 `Tagged(1)`
的写访问（因此不允许这种情况发生）。为了使内容更简短、更易于阅读，在下面的内容中，我将使用一个简明的语法来写下项，因此上面的栈将如下所示：

```rust,ignored
[ (0: Unique), (1: SharedReadOnly) ]
```

考虑第二个示例，如果栈是

```rust,ignored
[ (0: Unique), (Untagged: SharedReadWrite), (Untagged: SharedReadOnly) ]
```

则索引 2 处的项授予 `Untagged` 指针的读访问权限，而仅在索引 1 处授予 `Untagged` 指针写访问权限。

#### 删除不兼容的项

在第二步中，我们遍历栈中授予权限项之上的所有项，并查看它们是否与此访问兼容。

这实现了在最初的 SB1 中就已经存在想法：使用一个指针不允许将来使用另一个指针。例如，如果当前栈是

```rust,ignored
[ (0: Unique), (1: Unique) ]
```

那么使用 `Tagged(0)` 指针进行任何类型的访问都应该从栈中删除 `1: Unique` 项。

这与 SB1 中的部分内容一致，在该部分中，项从栈中弹出，直到授予权限的项位于顶部。也就是说，授予
`Unique` 权限与栈上一级项的 `Unique` 权限不兼容，因此必须移除后者。

但是，在新模型中，我们并不总是删除授予权限的项之上的所有项：

```rust,ignored
[ (0: Unique), (1: SharedReadOnly), (2: SharedReadOnly) ]
```

这次，有两个指针进行读取 `Tagged(1)` 和 `Tagged(2)`。使用它们中的任何一个都不应该对另一个产生影响，而且它们的项在栈上的特定顺序也不会产生任何影响。

换句话说，授予 `SharedReadOnly` 权限与其他 `SharedReadOnly` 权限兼容，因此当使用 `SharedReadOnly` 权限授予访问权限时，会保持其上方的其他 `SharedReadOnly` 权限。

有时，兼容性取决于执行的访问类型：在上面的示例中，如果我们使用 `Tagged(0)` 指针写入，则应该删除 `SharedReadOnly`
标签，因为写入唯一指针应该确保该指针位于栈的顶部。换句话说，**在写入时，`Unique` 权限与所有内容都不兼容**。

但是，我们也可以只使用 `Tagded(0)` 指针来读取！这是为了允许 [这样的][ex1] 安全代码从具有别名的可变和共享引用中交错读取。

[ex1]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=d9aa504b1be72a0f55fb5a744cba69ff

要实现这一点， **在读取时，`Unique` 权限只与其他 `Unique` 权限不兼容**。

下表全面记录了哪些权限在读或写访问上分别与其他权限兼容：

| **所兼容的权限**→ <br> ↓**授予权限的项 (granting item)** | `Unique` | `SharedReadWrite` | `SharedReadOnly` |
|:---------------------------------------------------------|----------|-------------------|------------------|
| `Unique`                                                 | 否       | 只读              | 只读             |
| `SharedReadWrite`                                        | 否       | 是                | 只读             |
| `SharedReadOnly`                                         | 否       | 否                | 只读             |

因此，如果授予权限的项具有 `SharedReadWrite` 权限，则栈中位于其上方的所有 `Unique` 都将被移除，而且在写入时，其上方的所有
`SharedReadOnly` 都将被移除。这确保了那些 `SharedReadOnly` 权限授予读取访问权限的指针不能再被使用：那些指针假定位置是只读的，所以我们必须确保它们在写入之后变得无效。

在这一点上，“栈”这个名称有点用词不当，因为我们在删除项时不遵循栈规则：仅仅因为某些项由于不兼容而被删除，并不意味着它上面的所有项也将被删除。

我将在 [后面][strict] 讨论为什么严格的栈规则不起作用。但我觉得在这一点上重命名整个东西会更令人困惑，而且元素在“栈”中的顺序仍然很重要，所以我保留了这个名称。

[strict]: #why-not-a-strict-stack-discipline

在 SB2 中，当指针被解引用时，不会发生更多操作：标签只与实际访问有关。这种简化是可能的，因为访问规则现在比以前更复杂，因此不需要对解引用操作进行额外的检查。

（具体地说，以前的解引用检查依赖于对正确处理内部可变性，需要知道访问发生在哪种 Rust 类型上；但现在所有这些信息都使用
`SharedReadWrite` / `SharedReadOnly` 标签进行编码。这不再依赖类型，还修复了一个关于 `UnsafeCell` 的 [烦人问题](https://github.com/rust-lang/miri/issues/615)。)

### 添加权限时，重新标签

到目前为止，我们所看到的栈中的权限框架具有（不）兼容性这个概念，从而可以表达一些想法，如：

* “当使用任何一个 `Unique` 指针的 父指针时，`Unique` 指针将无效。”  
  这是通过在上面表格的整个第一列中设置“否”来实现的：每次访问都移除授予权限的项之上的所有 `Unique` 权限。
* “当位置被写入时，`SharedReadOnly` 指针将无效。”  
  这是通过在表的整个最后一列中“只读”并且不授予具有 `SharedReadOnly` 权限的写访问来实现的。然而，除此之外，我们还需要确保在栈中，`Unique`
  或 `SharedReadWrite` 永远不会位于 `SharedReadOnly` 之上！在像 `[(0：SharedReadOnly)，(1：SharedReadWrite)]`
  这样的无意义栈中，我们可以使用 `Tagged(1)` 指针来编写代码，而不会使 `Tagged(0)` 指针无效。

现在的问题是，如何使用这种“语言”？我们必须定义将哪些项和权限添加到借用栈。就像以前一样，这种情况发生在重新标签期间。

在一般性地讨论重新标签之前，先看看给我们启发的示例，看看在 SB2 中重新标签是如何工作的，以及如何定义此程序会导致 UB：

```rust,ignored
fn main() {
    let x = &mut 0u32;
    Retag(x); // Tagged(0)
    // Stack: [ (0: Unique) ]

    let p = x as *mut u32;
    Retag([raw] p); // Untagged
    // Stack: [ (0: Unique), (Untagged: SharedReadWrite) ]

    foo(x, p);
}

fn foo(a: &mut u32, y: *mut u32) -> u32 {

    Retag(a); // Tagged(1)
    // Stack: [ (0: Unique), (1: Unique) ]

    *a = 1;

    let _b = &*a;
    Retag(_b); // Tagged(2)
    // Stack: [ (0: Unique), (1: Unique), (2: SharedReadOnly) ]

    // UB: y is Untagged, and there is no granting item in the stack!
    let _val = unsafe { *y };

    *a = 2;

    _val
}
```

Initially, `x` with tag `Tagged(0)` is the only reference, and the stack says that this is the only pointer with any kind of permission.
Next, we cast `x` to a raw pointer.
The raw retagging of `p` turns `p` into an `Untagged` pointer, and adds a new item granting `Untagged` pointers `SharedReadWrite` permission.
(Really, in the MIR it will say `&mut *x as *mut u32`, so there will be an additional `Unique` permission for the temporary mutable reference, but that makes no difference and I hope [we will change that eventually](https://github.com/rust-lang/rfcs/pull/2582).)

}{%endHighth%}最初，标签为`Tagge(0)`的`x`是唯一的引用，栈表示这是唯一有任何权限的指针。接下来，我们将`x‘转换为原始指针。对`p`的原始重新标签将`p`转换为一个`Untagged`指针，并添加一个新的项，授予`Untagged`指针`SharedReadWrite`权限。(实际上，在MIR中，它将`&mut*x表示为*mut u32`，因此临时可变引用将有一个额外的`Unique‘权限，但这没有什么不同，我希望我们最终会改变这一点。)

Then `foo` gets called, which starts with the usual retagging of all reference arguments.
`a` is originally `Tagged(0)`, and retagging a mutable reference acts like an access (just like in Stacked Borrows 1), so the first thing that happens is that all items above the granting item that are incompatible with a write access for a `Unique` permission get removed from the stack.
In our case, this means that the `Untagged: SharedReadWrite` gets removed.
Then, `a` gets the new tag `Tagged(1)` and `1: Unique` gets pushed on top of the stack.
Nothing interesting happens when writing to `a`.
When `_b` gets created, it gets assigned the new tag `Tagged(2)`, and a new item `2: SharedReadOnly` is pushed to the stack.
As we can see, shared and mutable references no longer differ in the tag they carry; the only difference is what kind of permission they get granted.
(I think this is a nice improvement from Stacked Borrows 1, where shared references had a different kind of tag. In particular, transmutes between mutable references and shared pointers no longer need any kind of special treatment.)
Finally, we come to the interesting point: the program reads from `y`.
That pointer is `Untagged`, and there is no item granting any access to such pointers, and thus the access is UB!
This is in contrast to Stacked Borrows 1, where instead of `2: SharedReadOnly` we set a special "frozen" marker on this location that would, as a side-effect, also grant untagged pointers like `y` read-only access.

然后调用`foo`，这以对所有引用参数进行通常的重新标签开始。`a`最初是`Tagded(0)`，重新标签可变引用的作用类似于访问(就像栈借阅1中一样)，因此发生的第一件事是，授予项上与`Unique`权限的写访问不兼容的所有项都从栈中移除。在我们的例子中，这意味着`Untag：SharedReadWrite`被移除。然后，`a`得到新的标签`Tagded(1)`，并将`1：Unique`推送到栈的顶部。在给`a‘写信时，没有什么有趣的事情发生。当创建`_b`时，会为其分配新的标签`Tagded(2)`，并将新条目`2：SharedReadOnly`推送到栈。正如我们所看到的，共享引用和可变引用在它们所携带的标签上不再是不同的；唯一的区别是它们被授予了什么样的权限。(我认为这是对栈借用1的一个很好的改进，在栈借用1中，共享引用有不同类型的标签。特别是，可变引用和共享指针之间的转换不再需要任何类型的特殊处理。)最后，我们来到了有趣的一点：程序从‘y’开始读取。该指针是`Untagged`，并且没有授予对此类指针的任何访问权限的项，因此访问是UB！这与栈借用1不同，在栈借用1中，我们在该位置设置了一个特殊的“冻结”标签，作为副作用，它还会授予像‘y’这样的未标签指针只读访问权限。

In general, during retagging, we start with some original tag used by our "parent" pointer, and we have a fresh tag `N` that will be used for the new pointer.
We have to decide which permission to grant so this tag, and where in the "stack" to insert it.
(We will not always insert new items at the top---again, as we will see following a strict stack discipline does not work.)
All of this happens for every location that the reference covers (as determined by its type).

一般来说，在重新标签期间，我们从“父”指针使用的一些原始标签开始，我们有一个新的标签`N‘，它将用于新的指针。我们必须决定授予哪个权限，以使这个标签，以及在“栈”中插入它的位置。(我们不会总是在顶部插入新项-同样，我们将看到遵循严格的栈规则是不起作用的。)所有这些都发生在引用覆盖的每个位置(由其类型确定)。

**Retagging a mutable reference.**
The simplest case of retagging is handling mutable references:
just like in Stacked Borrows 1, this starts by performing the actions of a write access with the parent tag.
`Unique` permissions are incompatible with anything on a write, so all items above the granting item get removed.
Then we add `N: Unique` on top of the stack to grant the new tag unique access to this location.

重新标签可变引用。重新标签的最简单情况是处理可变引用：就像在栈借用1中一样，这是从使用父标签执行写访问操作开始的。`Unique`权限与写入上的任何内容都不兼容，因此授予项以上的所有项都将被删除。然后，我们在栈顶部添加`N：Unique`，以授予新标签对该位置的唯一访问权限。

This encodes the idea that mutable references must be used in a well-nested way---if you just consider mutable references, Stacked Borrows 2 still follows a stack discipline.

这体现了可变引用必须以良好的嵌套方式使用的思想-如果只考虑可变引用，栈借阅2仍然遵循栈规则。

**Retagging a shared reference.**
When retagging a shared reference, we have to be mindful of `UnsafeCell`.

重新标签共享引用。在重新标签共享引用时，我们必须注意`UnSafeCell`。

Outside of `UnsafeCell`, we start by performing the actions of a read access with the parent tag.
Then we push `N: SharedReadOnly` on top of the borrow stack.
This way, we grant the new pointer read-only access but make sure that it gets invalidated on the next write through any aliasing pointer (because all write-granting items are below us, and thus we test for compatibility when they get used).
There might be items left between the granting item and the one we just added, but that's okay: if any of them gets used for reading, that will be compatible with the `SharedReadOnly` according to our table above; and if any of them gets used for writing then it is important that our `SharedReadOnly` gets removed.

在`UnSafeCell`之外，我们首先使用父标签执行读访问操作。然后我们在借用栈的顶部按下`N：SharedReadOnly`。这样，我们授予新指针只读访问权限，但确保它在通过任何别名指针的下一次写入时无效(因为所有写授予项都在我们下面，因此我们在使用它们时测试兼容性)。在授予项和我们刚刚添加的项之间可能会留下一些项，但没关系：如果它们中的任何一个被用于读取，根据我们的上表，它将与`SharedReadOnly`兼容；如果它们中的任何一个被用于写入，那么重要的是我们的`SharedReadOnly`被移除。

When we are inside an `UnsafeCell`, we will *not* perform the actions of a memory access!
Interior mutability allows all sorts of crazy aliasing, and in particular, one can call a function with signature
{% highlight rust %}
fn aliasing(refcell: &RefCell<i32>, inner: &mut i32)
{% endhighlight %}
such that `inner` points *into* `refcell`, i.e., the two pointers overlap!
Retagging `refcell` must not remove the `Unique` item associated with `inner`.

当我们在`UnSafeCell`中时，我们不会执行内存访问的操作！内部可变性允许各种疯狂的别名，尤其是可以调用签名为{%Highlight Rust%}fn别名的函数(refcell：&RefCell，Internal：&mut I32){%endHighlight%}，使得`inner`指向`refcell‘，即两个指针重叠！重新标签`refcell`不得删除与`inner`关联的`Unique`项。

So, when retagging inside an `UnsafeCell`, we *find* the write-granting item for the parent's tag, and then we add `N: SharedReadWrite` just above it.
This grants write access, as is clearly needed for interior mutability.
We cannot add the new item at the top of the stack because that would add it *on top of* the item granting `inner`.
Any write to `inner` would remove our item from the stack!
Instead, we make our item sit just on top of its parent, reflecting the way one pointer got derived from the other.

因此，当在`UnSafeCell`内部重新标签时，我们会找到父标签的写授予项，然后在它的正上方添加`N：SharedReadWrite`。这授予了写访问权限，这显然是内部易变性所必需的。我们不能将新项添加到栈顶部，因为这会将其添加到授予`inner`的项的顶部。对`inner`的任何写入都会从栈中删除我们的项！相反，我们使我们的项正好位于其父对象的顶部，反映了一个指针从另一个指针派生的方式。

**Retagging a raw pointer.**
Retagging for raw pointers happens only immediately after a reference-to-raw-pointer cast.
Unlike in Stacked Borrows 1, retagging depends on whether this is a `*const T` or `*mut T` pointer---this is a departure from the principle that these two types are basically the same (except for variance).
I have recently learned that the borrow checker actually handles `*mut` and `*const*` casts differently; also see [this long comment](https://github.com/rust-lang/rust/issues/56604#issuecomment-477954315).
Given that the idea of Stacked Borrows is to start with what the borrow checker does and extrapolate to a dynamic model that also encompasses raw pointers, I felt that it makes sense for now to mirror this behavior in Stacked Borrows.
This is certainly not a final decision though, and I feel we should eventually have a discussion whether we should make the borrow checker and Stacked Borrows both treat `*const T` and `*mut T` the same.

重新标签原始指针。原始指针的重新标签仅在对原始指针的引用强制转换之后立即发生。与SB1不同，重新标签取决于这是`*const T‘还是`*mut T’指针-这背离了这两种类型基本相同的原则(除了差异)。我最近了解到，借用检查器实际上处理`*mu`和`*const*`强制转换的方式是不同的；另请参阅这条长评论。考虑到SB的想法是从借用检查器所做的事情开始，并外推到也包含原始指针的动态模型，我觉得现在在SB中反映这种行为是有意义的。不过，这肯定不是最终的决定，我觉得我们最终应该讨论一下，是否应该让借阅检查器和栈借用都将`*const T‘和`*mut T’视为相同。

When casting to a `*mut T`, we basically behave like in the above case for inside an `UnsafeCell` behind a shared reference: we find the write-granting item for our parent's tag, and we add `Untagged: SharedReadWrite` just on top of it.
The way compatibility is defined for `SharedReadWrite`, there can be many such items next to each other on the stack, and using any one of them will not affect the others.
However, writing with a `Unique` permission further up the stack *will* invalidate all of them, reflecting the idea that when writing to a mutable reference, all raw pointers previously created from this reference become invalid.

当强制转换为`*mut T`时，我们的行为基本上类似于上面在共享引用后面的`UnSafeCell`中的情况：我们找到父代标签的写授予项，并在其上方添加`Untag：SharedReadWrite`。按照`SharedReadWrite`定义兼容性的方式，栈上可以有许多这样的项彼此相邻，使用其中任何一项都不会影响其他项。然而，在栈的上一层使用`Unique‘权限进行写入将使它们全部无效，这反映了在写入可变引用时，以前从该引用创建的所有原始指针都将变为无效。

When casting to a `*const T`, we behave just like retagging for a shared reference `&T` (including being mindful of `UnsafeCell`).
There isn't really anything that a `*const T` can do that a shared reference cannot (both in terms of aliasing and mutation), so modeling them the same way makes sense.

当强制转换为`*const T`时，我们的行为就像是为共享引用`&T`重新标签(包括注意`UnSafeCell`)。实际上，`*const T‘并不能做共享引用所不能做的任何事情(就别名和突变而言)，所以以相同的方式对它们建模是有意义的。

### Two-phase borrows

### 两期借贷

Stacked Borrows 1 \[had some support for two-phase borrows\]({% post_url 2018-12-26-stacked-borrows-barriers %}), but some advanced forms of two-phase borrows that used to be allowed by the borrow checker [could not be handled](https://github.com/rust-lang/rust/issues/56254).
With the additional flexibility of Stacked Borrows 2 and its departure from a strict stack discipline, it is possible to accept at least the known examples of this pattern that were previously rejected:
{% highlight rust %}
fn two_phase_overlapping1() {
let mut x = vec![\];
let p = &x;
x.push(p.len());
}
{% endhighlight %}
Previously, when the implicit reborrow of `x` in `x.push` got executed, that would remove the item for `p` from the stack.
This happens as part of the implicit write access that occurs when a mutable reference gets retagged.
With Stacked Borrows 2, we can do something else:
when retagging a two-phase mutable borrow, we do *not* perform the actions of a write access.
Instead, we just find the write-granting item for our parent's tag, and then add `N: Unique` just above it.
This has the consequence that on the first write access to this new pointer, everything on top of it will be removed, but until then the existing item for `p` remains on the stack and can be used just as before.

堆叠借用1[对两阶段借用有一些支持]({%post_url 2018-12-26-STACKED-LOBLES-BALENTS%})，但借用检查器过去允许的一些高级形式的两阶段借用无法处理。利用SB2的额外灵活性及其与严格的栈规则的背离，可以至少接受先前被拒绝的这种模式的已知示例：{%Highlight Rust%}fn Two_Phase_Overlapping1(){let mut x=vec！[]；let p=&x；x.ush(p.len())；}{%endHighth%%}以前，当执行对`x.ush`中的`x`的隐式重新借用时，这将从栈中移除用于`p`的项。这是在重新标签可变引用时发生的隐式写访问的一部分。使用SB2，我们可以做其他事情：当重新标签两阶段可变借用时，我们不执行写访问的操作。相反，我们只需找到父母标签的写授予项，然后在其上方添加`N：Unique‘。这样做的结果是，在第一次对这个新指针进行写访问时，它上面的所有东西都将被移除，但在此之前，`p‘的现有项仍保留在栈中，可以像以前一样使用。

Just like with Stacked Borrows 1, we then proceed by doing a *shared* reborrow of the parent's tag from `N`.
This way, the parent pointer can be used in the ways a shared reference can (including writes, if there is interior mutability) without invalidating `N`.
This is somewhat strange because we then have "parent tag - new tag - parent tag" on the stack in that order, so we no longer properly reflect the way pointers got derived from each other.
More analysis will be needed to figure out the consequences of this.

就像栈借用1一样，我们接着从`N‘共享重新借用父代的标签。这样，父指针可以以共享引用的方式使用(包括写入，如果有内部可变的话)，而不会使`N‘无效。这有点奇怪，因为我们随后在栈上按该顺序拥有“父标签-新标签-父标签”，因此我们不再正确地反映指针从彼此派生的方式。需要更多的分析来弄清楚这一点的后果。

We should also try to fully understand the effect this "weak" form of a mutable reborrow (without a virtual write access) has on the optimizations that can be performed.
Most of the cases of Stacked Borrows violations I found in the standard library are caused by the fact that even just *creating* a mutable reference asserts that it is unique and invalidates aliasing pointers, so if we could weaken that we would remove one of the most common causes of UB caused by Stacked Borrows.
On the other hand, this means that the compiler will have a harder time reordering uses of mutable references with function calls, because there are fewer cases where a mutable reference is actually assumed to be unique.

我们还应该尝试充分理解这种“弱”形式的可变重新借用(没有虚拟写访问)对可以执行的优化的影响。我在标准库中发现的大多数SB违规的情况都是由这样一个事实引起的，即即使只是创建一个可变引用也会断言它是唯一的，并使别名指针无效，所以如果我们可以削弱这一点，我们就可以删除由堆叠借用引起的UB的最常见原因之一。另一方面，这意味着编译器将更难对使用函数调用的可变引用进行重新排序，因为实际上假定可变引用是唯一的情况较少。

### Barriers are dead, long live protectors

### 屏障是死亡的，万岁的保护者

With the departure from a strict stack discipline, I also had to re-think the concept of \[barriers\]({% post_url 2018-12-26-stacked-borrows-barriers %}).
The name was anyway terrible, so I replaced barriers by *protectors*: an `Item` actually consists not only of a `Tag` and a `Permission`, but also optionally of a "protector" represented as a `CallId` referencing some function call (i.e., some stack frame).
As long as that function call is running, the item with the protector may not be removed from the stack (or else we have UB).
This has pretty much the same effects as barriers did in Stacked Borrows 1.

由于背离了严格的栈规则，我也不得不重新思考[Bills]({%post_url 2018-12-26-STACKED-LOWROWS-BALENTS%})的概念。不管怎么说，这个名字很糟糕，所以我用保护器替换了屏障：实际上，`Item`不仅包含一个`Tag`和一个`Permission`，还可以包含一个表示为引用某个函数调用(即某个栈框架)的`CallId`的“保护器”。只要该函数调用仍在运行，带有保护器的项就不能从栈中移除(否则我们就会有UB)。这与SB1中的屏障几乎具有相同的效果。

<a name="why-not-a-strict-stack-discipline"></a>

## 为什么不制定严格的栈规则呢？

The biggest surprise for me in designing Stacked Borrows 2 was that I was not able to enforce a strict stack discipline any more.
For retagging, that is only partially surprising; really in Stacked Borrows 1 we already added barriers below the "frozen" marker sitting next to a stack, and the `aliasing` example with the `RefCell` mentioned above only worked due to a hack that relied on reusing existing items in the middle of the stack instead of pushing a new one.
However, I had initially hoped that the rules for memory accesses would be slightly different:
my plan was that after finding the granting item, we would seek upwards through the stack and find the first incompatible item, and then remove everything starting there.
That would be a stack discipline; we would basically pop items until all items above the granting item are compatible.

在设计栈借用2时，对我来说最大的惊喜是我不能再执行严格的堆叠规则。对于重新标签，这只是一部分令人惊讶的事情；实际上，在Stack Borrow 1中，我们已经在栈旁边的“冻结”标签下添加了屏障，上面提到的`RefCell`的“aliasing`”示例之所以有效，是因为它的黑客攻击依赖于重用栈中间的现有项，而不是推送新的项。然而，我最初希望内存访问的规则会略有不同：我的计划是在找到授予项后，向上查找栈，找到第一个不兼容的项，然后删除从那里开始的所有项。这将是一个栈规则；我们基本上会弹出项，直到授予项上的所有项都兼容。

Unfortunately, that would reject many reasonable programs, such as:
{% highlight rust %}
fn test(x: &mut \[i32\]) -> i32 { unsafe {
let raw = x.as_mut_ptr(); // implicitly: as_mut_ptr(&mut \*x)
let \_val = x\[1\];
return \*raw;
} }
{% endhighlight %}
The issue with this example is that when calling `as_mut_ptr`, we reborrow `x`.
So after the first line the stack would look something like (using `x` as notation for `x`'s tag)

不幸的是，这将拒绝许多合理的程序，例如：{%Highlight Rust%}FN test(x：&mut[I32])->I32{unSafe{let raw=x.as_mut_ptr()；//隐式：as_mut_ptr(&mut*x)let_val=x[1]；Return*raw；}}{%endHighlight%}这个示例的问题是，当调用`as_mut_ptr`时，我们重新借用了`x`。因此，在第一行之后，栈将类似于(使用`x`作为`x`的标签的符号)

```
[ ..., (x: Unique), (_: Unique), (Untagged: SharedReadWrite) ]
```

In other words, there would be some `Unique` item *between* the items for `x` and `raw`.
When reading from `x` in the second line, we determine that this `Unique` tag is not compatible.
This is important; we have to ensure that any access from a parent pointer invalidates `Unique` pointers---that's what makes them unique!
However, if we follow a stack discipline, that means we have to pop the `Untagged: SharedReadWrite` *and* the `_: Unique` off the stack, making the third line UB because the raw pointer got invalidated.

换句话说，在`x`和`raw`的项之间会有一些`Unique`项。在读取第二行的`x`时，我们确定这个`Unique`标签不兼容。这一点很重要；我们必须确保来自父指针的任何访问都会使`Unique‘指针无效-这就是它们的独特之处！然而，如果我们遵循栈规则，这意味着我们必须将`Untag：SharedReadWrite`和`_：Unique`从栈中弹出，使第三行UB，因为原始指针无效。

`test` seems like a perfectly reasonable function, and in fact this pattern is used in the standard library.
So in order to allow such code, accesses in Stacked Borrows 2 do *not* follow a stack discipline:
we remove all incompatible items above the granting item and ignore any interleaved compatible items.
As a consequence, line 2 in the above program removes `_: Unique` but keeps `Untagged: SharedReadWrite`, so line 3 is okay.

\`test‘看起来是一个非常合理的函数，实际上标准库中使用了这个模式。因此，为了允许这样的代码，栈借用2中的访问不遵循栈规则：我们删除授予项上方的所有不兼容项，并忽略任何交错的兼容项。因此，上面程序的第2行删除了`_：Unique`，但保留了`Untag：SharedReadWrite`，所以第3行没有问题。

However, this means we also accept the following, even if we eventually do fully precise tracking of raw pointers:
{% highlight rust %}
fn main() { unsafe {
let x = &mut 0;
let raw1 = x as \*mut \_;
// Stack: \[ (x: Unique), (raw1: SharedReadWrite) \]

然而，这意味着我们也接受以下内容，即使我们最终完全精确跟踪原始指针：{%Highlight Rust%}fn main(){unSafe{let x=&mut 0；let raw1=x as*mut_；//Stack：[(X：Unique)，(raw1：SharedReadWite)]

let tmp = &mut \*raw1;
let raw2 = tmp as \*mut \_;
// Stack: \[ (x: Unique), (raw1: SharedReadWrite), (tmp: Unique), (raw2: SharedReadWrite) \]

让tmp=&mut*raw1；让raw2=tmp as*mut_；//栈：[(X：唯一)，(raw1：共享读写)，(tMP：唯一)，(raw2：共享读写)]

\*raw1 = 1; // This will invalidate tmp, but not raw2.
// Stack: \[ (x: Unique), (raw1: SharedReadWrite), (raw2: SharedReadWrite) \]

\*raw1=1；//这将使tMP无效，但不会使raw2无效。//Stack：[(X：唯一)，(raw1：共享读写)，(raw2：共享读写)]

let \_val = \*raw2;
} }
{% endhighlight %}
Naively I would assume that if we tell LLVM that `tmp` is a unique pointer, it will conclude that `raw2` cannot alias with `raw1` as that was not derived from `tmp`, and hence LLVM might conclude that `_val` must be 0.
This means the program above would have to have UB.
However, in the current framework I do not see a way to make it UB without also making the reasonable example from the beginning of this section UB.
So maybe we can find a way to express to LLVM precisely what we mean, or maybe we can only exploit some of these properties in home-grown optimizations, or maybe we find a way to have UB in the second example but not in the first.
(Like, maybe we do the stack-like behavior on writes but keep the more lenient current behavior on reads? That seems annoying to implement though. Maybe a stack is just the wrong data structure, and we should use something more tree-like.)

Let_val=*raw2；}}{%endHighth%}我天真地假设，如果我们告诉LLVM`tmp`是唯一指针，它会得出结论，`raw2`不能与`raw1`别名，因为它不是从`tmp`派生的，因此LLVM可能会得出`_val`一定是0的结论。这意味着上面的程序必须有UB。然而，在目前的框架中，我看不到一种方法来使它成为UB，而不是从本节UB的开头也做出合理的例子。因此，也许我们可以找到一种方法，准确地向LLVM表达我们的意思，或者我们只能在自行开发的优化中利用这些属性中的一些，或者我们可以找到一种方法，在第二个示例中使用UB，而不是在第一个示例中。(例如，也许我们在写入上执行类似栈的行为，但在读取上保留更宽松的当前行为？不过，这似乎很难实现。也许栈就是错误的数据结构，我们应该使用更像树的数据结构。)

## Conclusion

## 结论

Stacked Borrows 2 shares with Stacked Borrows 1 just the general structure: pointers are tagged, and there is a per-location stack indicating which tags are allowed to perform which kinds of operations on this location.
In the new version, shared references are tagged the same way as mutable references (just with an ID to distinguish multiple references pointing to the same location); the stack keeps track of which IDs are read-only (shared) pointers and which are unique (mutable) pointers.
The only operations affected by Stacked Borrows 2 are memory accesses and the `Retag` instructions explicitly represented in the MIR; there is no longer any action on a pointer dereference.
This balances the extra complexity of the new access rules (the new implementation is actually a dozen lines shorter than the old, despite long comments).
The "stack" is unfortunately not used in a completely stack-like fashion though.

栈借用2与栈借用1共享的只是一般结构：指针被标签，并且存在每个位置的栈，该栈指示允许哪些标签在该位置上执行哪些类型的操作。在新版本中，共享引用的标签方式与可变引用相同(只是使用ID来区分指向同一位置的多个引用)；栈跟踪哪些ID是只读(共享)指针，哪些是唯一(可变)指针。受堆叠借用2影响的唯一操作是内存访问和在MIR中显式表示的‘Retag`指令；不再有任何关于指针解引用的操作。这平衡了新访问规则的额外复杂性(新的实现实际上比旧的短了十几行，尽管注释很长)。遗憾的是，“栈”并没有以完全类似栈的方式使用。

I leave it as an exercise to the reader to convince yourself that the \[key properties of Stacked Borrows 1\]({% post_url 2018-11-16-stacked-borrows-implementation %}#5-key-properties) still hold, and that uniqueness of mutable references is maintained even if we do a shared reborrow, as long as we keep that shared reference to ourselves.

我把它作为一个练习留给读者来说服自己，[堆叠借用的关键属性1]({%post_url 2018-11-16-STACKED-LOBROWS-Implementation%}#5-Key-Properties)仍然有效，而且即使我们进行共享再借用，可变引用的唯一性也会保持不变，只要我们将共享引用保留给自己。

The new model gives some wiggle-room in both the notion of which permissions are compatible with others and where exactly which permissions are added in the "stack" in a reborrow.
We could use that wiggle-room to experiment with a bunch of closely related models to see how they affect which code gets accepted and analyze their impact on optimizations.

新模型在哪些权限与其他权限兼容的概念以及在重新借用时将哪些权限添加到“栈”中的确切位置上都有一定的回旋余地。我们可以利用这种回旋余地来试验一系列密切相关的模型，以了解它们如何影响哪些代码被接受，并分析它们对优化的影响。

I am quite excited by this new model!
It puts us into a good position to do more precise tracking for raw pointers, similar to what already happens for shared references (and something I was worried about in the original model).
That will be needed for compatibility with LLVM.
However, [there are still some known issues](https://github.com/rust-lang/unsafe-code-guidelines/issues?q=is%3Aissue+is%3Aopen+label%3Atopic-stacked-borrows), and also the fact that we do not actually use the "stack" as a stack indicates that maybe we should use a different structure there (potentially something more like a tree).
This is definitely not the final word, but I think it is a step in the right direction, and I am curious to see how it works out.
As usual, if you have any questions or comments, please join the [discussion in the forums](https://internals.rust-lang.org/t/stacked-borrows-2/9951)!

我对这个新型号很感兴趣！它使我们能够对原始指针进行更精确的跟踪，类似于对共享引用已经发生的事情(这也是我在原始模型中担心的事情)。这将是与LLVM兼容所必需的。然而，仍然有一些已知的问题，而且我们实际上并没有使用“栈”作为栈的事实表明，也许我们应该在那里使用不同的结构(可能是更像树的结构)。这肯定不是最终结论，但我认为这是朝着正确方向迈出的一步，我很好奇它会如何发挥作用。像往常一样，如果你有任何问题或建议，请加入论坛的讨论！
