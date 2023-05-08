# 栈借用中的屏障与两阶段借用

> 原文：[Barriers and Two-phase Borrows in Stacked Borrows](https://www.ralfj.de/blog/2018/12/26/stacked-borrows-barriers.html)
> | 作者：Ralf Jung | 2018 年 12 月 26 日 | [IRLO]

[IRLO]: https://internals.rust-lang.org/t/barriers-and-two-phase-borrows-in-stacked-borrows/9100

我在 Mozilla 的实习（“研究助理”）几周前结束了，这篇文章是我对 Miri 和 Stack Borrow 所做的最新调整的报告。

当然，这两个项目都没有“完成”。然而，两者都相当成熟，所以我觉得某种结案报告是有意义的。

此外，如果我想完成我的博士学位，我将不得不认真减少我研究 Rust 的时间 —— 所以至少从我的角度来说，事情从现在开始会进展得更慢。

特别是，现在只需一条命令即可安装 Miri 并在其中运行测试套件！如果你对剩下的内容不感兴趣，可以一直向下翻。

## 调整栈借用

我对栈借用做了一些小调整，这主要是为了让事情总体上更一致。所有这些都已经整合到我的
[上一篇关于栈借用阅的帖子](./stacked-borrows-implementation.md) 中。 

唯一值得注意的功能变化是创建共享引用：以前，这会在冻结位置之前将一个 `Shr` 项推送到栈。然而，这将允许如下例所示的可变性：

```rust,ignore
fn demo_shared_mutation(ptr: &mut u8) {
    let shared = &*ptr; // pushes a `Shr` item
    let raw = shared as *const u8 as *mut u8;
    unsafe { *raw = 5; } // allowed: unfreezing, then matching the `Shr` item
}
```

由于从一开始 Rust 采用“不通过来自共享引用的指针进行可变”这条规则，所以我更改了创建共享引用，不再无条件地推送 `Shr` 项。

**现在，只有当指针指向的对象因 `UnsafeCell` 而可变时，才允许发生这种情况。因此，上述代码现在是被禁止的。**

## 屏障

除了这些小的调整之外，我还实现了在 [原提案](./stacked-borrows.md) 中已经描述的屏障 (barriers)。屏障的目的是处理如下代码：

```rust,ignore
fn aliasing(ptr: &mut u8, raw: *mut u8) -> u8 {
    let val = *ptr;
    unsafe { *raw = 4; }

    val // we would like to move the `ptr` load down here
}

fn demo_barrier() -> u8 {
    let x = &mut 8u8;
    let raw = x as *mut u8;
    aliasing(unsafe { &mut *raw }, raw)
}
```

这里的 `aliasing` 获取两个别名指针作为参数。这意味着写入 `raw` 会更改 `ptr` 中的值，因此所需的优化将更改 `demo_barrier` 的行为（目前它返回 8，将
`ptr` 的职责下移后，它将返回 4）。无屏障情况下的栈借用将允许此代码，因为在 `raw` 之后不会再次使用 `ptr`。

为了禁止此代码并启用所需的优化，我们引入了一种可以推送到每个位置栈的新项：指向特定函数调用的 **函数屏障** （function barrier)。

当函数运行时，屏障物可能不会弹出。因此，借用栈上的项类型现在如下所示：

```rust,ignore
pub enum BorStackItem {
    Uniq(Timestamp),
    Shr,
    FnBarrier(CallId),
}
```

在对函数进行初步的重新标记时，这些屏障会被推入（请记住，我们在每个函数的开头为所有参数插入了 `Retag` 语句）。

我们只对引用这样做（不对 `Box` 这样做），而且不会将屏障推送到含有共享的 `UnsafeCell` 的位置。

具体地说，在重新标记的第 4 步和第 5 步中间（在执行访问操作之后和推送新项之前），我们推送一个指向当前函数调用的
`FnBarrier`。此外，当使用屏障重新标记时，不执行通常的额外的检查 —— 即使不需要重新借用，可能也需要推送屏障。

当指针被解引用时，屏障不起作用：在栈中查找项只是跳过屏障。

然而，当执行内存访问并从栈中弹出项时，会在遇到活动的屏障（即仍在进行的函数调用的屏障）时停止。如果发生这种情况，就会有 UB。

总而言之，这些规则使上面的示例代码无效，从而实现了我们想要的优化：在 `*raw = 4` 中进行内存访问时，会从栈上弹出，直到发现 `Shr` 
项。这将遇到 `aliasing` 对 `val` 进行初步重新标记时添加的屏障，从而触发 UB。

一个屏障带来的有趣的（而且是好的）后果是它们让以下代码 UB：

```rust,ignore
fn aliasing(ptr1: &mut u8, ptr2: &mut u8) {}


fn demo_barrier() -> u8 {
    let x = &mut 8u8;
    let raw = x as *mut u8;
    aliasing(unsafe { &mut *raw }, x)
}
```

没有屏障的话，这段代码没有问题，因为 `aliasing` 实际上并没有以冲突的方式使用这两个别名指针。

有屏障的话，初步重新标记 `ptr2` 会出错，因为它试图弹出刚刚被推送到 `ptr1` 的屏障。这很好，因为我们实际上不想允许将两个别名指针传递给 `aliasing`。

屏障还有一个影响：当内存被释放时，它的栈中不能留下任何活动的屏障。目前屏障确保传递给函数的引用在该函数调用期间不会失效；
这个额外的检查确保它也不会被释放。这一点很重要，因为我们通过 `dereferencable` 属性告诉 LLVM，在函数调用期间，引用是可解引用的。
为了证明这一属性，我们必须证明，程序确实不能释放这一内存。

### 直觉和设计抉择

屏障的一个高层次直觉是，它们编码一个规则：传递给函数的引用比函数调用的时间存活地更长。当函数仍在执行时，与该引用匹配的项永远不会被弹出：
如果它被弹出，就会碰到屏障并触发 UB。这也是为什么我们没有 `Box` 的屏障：它们与函数调用没有任何这样的关系。

另一个例外是，对于共享的 `UnsafeCell` 则没有屏障，这一点更为微妙。

在我的 [上一篇文章](./stacked-borrows-implementation.md) 中，有一个例子，一个共享和一个可变的引用在函数内具有别名（共享引用是
`&RefCell`，而可变引用指向它）。那个示例的关键是重新标记期间的额外的检查，其中共享引用不会被重新借用（否则这将使可变引用无效）。
如果我们想要为该共享引用添加屏障的话，则不能将其推入栈的顶部，而应该将其添加到与该引用匹配的 `Shr` 项的中间位置。但目前，我尽量避免将项添加到栈的中间。

共享的 `UnsafeCell` 和屏障的另一个问题在 [issue#55005](https://github.com/rust-lang/rust/issues/55005) 中总结了：`Arc`
中，实际上可能遇到如下情况，由共享引用（但在 `UnsafeCell` 内）引用的内存被释放，而（将该共享引用作为参数的）函数仍在运行。这违反了
LLVM 对 `dereferencable` 指针的前提，也违反了具有活动屏障的内存不能被释放的规则。我们将不得不声明
`Arc` 的当前实现不正确，或者更改我们用于 LLVM 的属性。模型目前对此的立场是，`Arc` 是正确的，并允许释放共享的 `UnsafeCell` 背后的数据，而所产生的属性错误。

## 两阶段借用

在罗马的 RustFest 聚会期间，有人提醒我，Stack Borrow 目前不支持两阶段借用[^two] ([two-phase borrows])。这意味着以下代码（2018 版次中的安全代码）将被 Miri 拒绝：

[two-phase borrows]: http://smallcultfollowing.com/babysteps/blog/2017/03/01/nested-method-calls-via-two-phase-borrowing/

[^two]: 译者注：关于“两阶段借用”见官方最新说明 [rustc-dev-guide](https://rustc-dev-guide.rust-lang.org/borrow_check/two_phase_borrows.html)。

```rust,ignore
fn two_phase() {
    let mut v = vec![];
    v.push(v.len());
}
```

为了解决这个问题，我只需要做一个很小的改变。重新标记 (retag)  现在为两阶段借用提供了一种特殊模式（在上面的例子中，当重新标记包含第一个参数
`push` 的临时对象时将使用该模式），在这种模式中，当完成所有常规步骤之后，我们在新创建的可变借用基础上创建的共享的重新借用，并具有所有常规效果（冻结或将
`Shr` 推送到栈上。由于通过可变引用读取不会解冻或从栈中弹出 `Shr` 的规则，这意味着原来的 `v`
可用于读取，直到第一次写入新创建的（两阶段）借用时为止。写入操作将解冻并将 `Shr` 从栈中弹出，不允许通过其他引用进行任何进一步读取。这相当于两阶段借款的“激活” (activation)。

不幸的是，这个模型不足以处理 rustc 当前接受的所有代码。有关这一问题的进一步讨论，请参阅这 [issue#56254](https://github.com/rust-lang/rust/issues/56254)。

## 栈借用规范

在这里描述的有着各种变化，在我之前的博客文章中只对栈借用做了独立的描述，但与实际的实现具有明显的差异。

我会尝试找到一个好的时机，对目前在 Miri 中实现的任何版本的栈借用进行“实施方面的描述”，这样就不仅仅只有代码看了。如果我找到了那种描述，我会让你知道的！[^desc]

[^desc]: 译者注：见 Ralf 的 stacked borrows [论文](https://www.ralfj.de/blog/2019/11/18/stacked-borrows-paper.html)。

## 使 Miri 更易于使用

最后，我决定花一些时间使 Miri 更易于安装和使用，并修复运行测试套件的问题。现在你要做的就是

```shell
rustup +nightly component add miri
```

然后转到你的项目目录，运行 `cargo +nightly miri test`（你可能需要先 `cargo clean` 以确保所有依赖项都以正确的方式为 Miri 构建）。

在第一次启动时，Miri 将（在你同意的情况下）准备一个适合在解释器中执行 Rust 程序的 libstd，然后它将运行你的测试套件。
