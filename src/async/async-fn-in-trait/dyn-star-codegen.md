# `dyn*` 如何生成代码

原文：[A Look at dyn* Code Generation](https://blog.theincredibleholk.org/blog/2022/12/12/dyn-star-codegen/) | by Eric Holk | 2022 年 12 月 12 日

正如我以前写过的文章 [AFIT 如何在 rustc 中工作][AFIT-rustc] 所言，异步 Rust 的一个重要目标是在任何地方都支持异步函数，包括在 trait 对象 (`dyn Trait`) 中。

为此，我们增加了一个新的实验类型 `dyn*`，它将使我们更灵活地支持异步方法的动态分发。

现在我们已经为 nightly Rust 提供了 `dyn*` 的实验支持，所以我们可以开始尝试，并利用我们的经验来指导将来的发展。

[AFIT-rustc]: ./how-async-functions-in-traits-could-work-in-rustc.md

我们想要确保的一件事是，使用 `dyn*` 的成本不会比 `dyn` 已导致的成本更高。

理想情况下，我们可以让 `dyn* Trait` 生成的代码与 `dyn Trait` 生成的代码基本相同。

考虑到这一点，在这篇文章中，我想看看 Rust 目前生成的一些代码。

从剖析 `dyn Trait` 开始，然后我们将看到 `dyn* Trait` 让情况如何变化。

# `dyn Trait`

让我们从 Rust 如何表示 trait 对象开始。

Rust 中的 trait 对象（又名 `dyn Trait`）有点奇怪。[^dyn-trait] 它们是未知大小类型 (unsized type) 的一个例子，这意味着在实践中，你通常不会直接与它们打交道。

[^dyn-trait]: `dyn*` 的第二个动机是让 `dyn` 比现在更好。我们有机会消除一些奇怪的东西，让 trait 对象的工作方式更像 Rust 中的其他类型。

例如，我可能希望编写如下内容：

```rust,ignore
fn debug_print(x: dyn Debug) {
    println!("{x:?}")
} 
```

但是，如果这样做，编译器会给我们一个~令人困惑~非常有用的错误消息[^dyn-error]：

[^dyn-error]: 这条信息给我留下了非常深刻的印象。我希望能这样说：“看看这个错误消息是多么难以理解，这是因为 trait 对象令人困惑”，但我本应该知道得更清楚。

```rust,ignore
error[E0277]: the size for values of type `(dyn Debug + 'static)` cannot be known at compilation time
 --> src/lib.rs:3:16
  |
3 | fn debug_print(x: dyn Debug) {
  |                ^ doesn't have a size known at compile-time
  |
  = help: the trait `Sized` is not implemented for `(dyn Debug + 'static)`
help: function arguments must have a statically known size, borrowed types always have a known size
  |
3 | fn debug_print(x: &dyn Debug) {
  |                   + 
```

编译器告诉我们可以做些什么来修复这个问题，所以试一试。

```rust
#use std::fmt::Debug;
#use std::mem::size_of_val;
fn debug_print(x: &dyn Debug) {
    println!("{x:?}");
    println!("{}", size_of_val(&x));
} 
#debug_print(&123u8);
```

这段代码可以很好地编译和运行。

编译器告诉我们的是，借用的类型总是有一个已知的大小，那么它们有多大呢？

`&` 是一个指针，在 64 位机器上，指针是 64 位或者说 8 字节。

但是如果我打印 `x` 的大小，我看到的是 16 个字节。到底怎么回事？

Rust 中的指针并不总是 8 个字节。有时它们会更大，这样你就可以向它们附加额外的数据。

尤其是指向 unsize 数据的指针，比如 `str`、`[T]`，以及我们现在关注的 `dyn Trait`。

这三种情况下，我们会看到 `&str`、`&[T]` 和 `&dyn Trait` 的大小是两个字 (words)或 16 个字节。

有时称这些宽指针 (wide pointer) 或胖指针 (fat pointer)，因为它们的大小是常规指针的两倍。

但是这个多余的字是用来做什么的呢？对于 `&str` 和 `&[T]`，第一个字指向实际数据（字符串中的字符或 `T`
的数组），而第二个字存储长度，编译器将其用于动态边界检查。

不过，trait 对象略有不同。它们仍然在第一个字中存储指向底层数据的指针，但第二个字中存储指向 vtable 的指针。 vtable 看起来有点像这样：

```rust,ignore
struct DebugVtable {
    size: usize,
    alignment: usize,
    drop: fn(*mut ()),
    fmt: fn(*const (), &mut Formatter<'_>),
} 
```

vtable 始终至少有三个部分：
* 底层值 (underlying value) 的大小[^fn:underlying]
* 底层值的对齐方式
* 指向 drop 函数的指针

[^fn:underlying]: 让我有点惊讶的是，即使你不知道具体类型，`std::mem::size_of_val` 也可以读取该字段，并告诉你 `&dyn Trait` 对象引用的底层数据的大小。请见这个 
[示例][underlying]。我认为 `std::mem::align_of_val` 的工作原理与此类似。

[underlying]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=752ac7373ad448429bed6f75fa676cb5

然后，vtable 将拥有 trait 中每个方法的附加条目。因为我们的示例一直在查看`Debug` trait ，所以我们只有一个额外的`fmt`方法条目。

编译器使用此 vtable 调用背后类型上的方法，而不必知道有关它的任何其他细节。

## 虚拟方法调用

到目前为止，在我们的示例中，一直在使用 `Debug` trait 。这在某些方面很方便，因为它在 `std` 中随时可用，被广泛实现，也被广泛使用。

它主要用于 `format!("{x:?}")` 这样的格式字符串的上下文中，这将对 `Debug` 方法的实际调用隐藏在宏生成的代码之后。

因此，为了展示调用虚拟方法，让我们创建一个新的 trait 作为示例。

定义一个 `Counter` trait，它有两个方法：一个用于读取计数器，另一个用于将值添加到计数器。

这些加在一起应该足以探索 trait 对象发生的许多事情。

```rust,ignore
trait Counter {
    fn get_value(&self) -> usize;
    fn increment_by(&mut self, amount: usize);
} 
```

我们可以将指向 `dyn Counter` 对象的指针视为元组或结构体，如下所示：

```rust,ignore
struct DynCounter {
    this: *mut (),
    vtable: *const CounterVtable,
}

struct CounterVtable {
    size: usize,
    alignment: usize,
    drop: fn(*mut ()),
    get_value: fn(*const ()) -> usize,
    increment_by: fn(*mut (), usize),
} 
```

我附上了 vtable 的定义。

现在，来看一些使用计数器的代码：

```rust,ignore
fn use_counter(counter: &mut dyn Counter) -> usize {
    counter.increment_by(42);
    counter.get_value()
} 
```

Rust 不知道 `Counter` 的背后类型是什么，因此不能生成对该方法的直接调用。相反，它使用 vtable 来确定它应该使用哪种方法。

要了解 Rust 如何在 `use_counter` 中用 vtable，让我们从使用通用函数调用语法 (universal function call syntax, UFCS) 重写调用开始。

```rust,ignore
fn use_counter(counter: &mut dyn Counter) -> usize {
    <dyn Counter as Counter>::increment_by(counter, 42);
    <dyn Counter as Counter>::get_value(counter)
} 
```

将 self 参数写成显式，让 vtable 的转换更像是查找和替换。基本上，只要看到 `Counter`，就把它替换成 `counter.this`，而看到
`<dyn Count as Counter>::` 就把它替换成 `counter.vtable`。

```rust,ignore
fn use_counter(counter: DynCounter) -> usize {
    counter.vtable.increment_by(counter.this, 42);
    counter.vtable.get_value(counter.this)
} 
```

为了清楚起见，我省略了一些使其合法化的强制转换和显式解引用[^deref]。下面是无省略的部分：

[^deref]: Rust 还在调用这些方法时使用了 `Deref` 和 `DerefMut` trait，我在本文中完全省略了这些 trait。

```rust,ignore
fn use_counter(counter: DynCounter) -> usize {
    unsafe {
        ((*counter.vtable).increment_by)(counter.this, 42);
        ((*counter.vtable).get_value)(counter.this)
    }
} 
```

查看这段代码，我们可以猜测需要多少内存引用，但最精确的方法是查看生成的汇编代码。

幸运的是，`use_counter` 的原始版本和我们的“手动编译”版本都可以编译成相同的汇编代码，如下所示。

```asm
1  push	r14
2  push	rbx
3  push	rax
4  mov	r14, rsi
5  mov	rbx, rdi
6  mov	esi, 42
7  call	qword ptr [r14 + 32]
8  mov	rdi, rbx
9  mov	rax, r14
10 add	rsp, 8
11 pop	rbx
12 pop	r14
13 jmp	qword ptr [rax + 24]
```

有必要花一点时间讨论一下生成的汇编。就我们的目的而言，有两条指令被视为间接指令。

第 7 行的 `call` 指令和第 13 行 `jmp` 指令，都是通过 vtable 对方法的间接调用。这里包括了 `jmp` 是因为 LLVM 能够应用 [尾部调用优化][tail_call]。

[tail_call]: https://en.wikipedia.org/wiki/Tail_call

我本来希望看到另外一对加载指令读取 `DynCounter` 值的 `this` 和 `vable` 字段（或者在 `&mut dyn Counter` 的情况下也是等价的字段），但是看起来
Rust 选择通过一对寄存器来传递它们，其中 `rsi` 持有 vtable 指针，而 `rdi` 持有 this 指针。

从本质上讲，其余的指令是移动数据，以确保我们遵守调用约定，将数据放在正确的位置作为参数传递，等等。

回到 `dyn*`，理想情况下，我们将能生成与此基本相同的汇编。

## 所有权和 drop

再来看看另一种情况：

```rust,ignore
fn use_counter(mut counter: Box<dyn Counter>) -> usize {
    counter.increment_by(42);
    counter.get_value()
} 
```

与原来的 `&mut dyn Counter` 相比，这个版本有一个重要的区别。

`Box` 是一个被拥有的类型 (owned type)，这意味着当它超出范围时，我们有责任销毁存储在 Box 中的任何值，然后释放 Box。让我们看看这在汇编语言中是什么样子。

```assembly
1    push	r15
2    push	r14
3    push	rbx
4    sub	rsp, 16
5    mov	rbx, rsi
6    mov	r14, rdi
7    mov	qword ptr [rsp], rdi
8    mov	qword ptr [rsp + 8], rsi
9    mov	esi, 42
10   call	qword ptr [rbx + 32]
11   mov	rdi, r14
12   call	qword ptr [rbx + 24]
13   mov	r15, rax
14   mov	rdi, r14
15   call	qword ptr [rbx]
16   mov	rsi, qword ptr [rbx + 8]
17   test	rsi, rsi
18   je	.LBB4_5
19   mov	rdx, qword ptr [rbx + 16]
20   mov	rdi, r14
21   call	qword ptr [rip + __rust_dealloc@GOTPCREL]
22  
23 .LBB4_5:
24   mov	rax, r15
25   add	rsp, 16
26   pop	rbx
27   pop	r14
28   pop	r15
29   ret
```

有些部分应该看起来很眼熟。第 10 行调用 `[ rbx + 32]`，对应于调用 `increment_by`，第 12 行调用 `[rbx + 24]`，对应调用 `get_value`。

但这一次调用 `get_value` 不是尾部调用；因为后面还有更多工作要做。

下一个有趣的部分是第 15 行，调用了 `[rbx]`。这看起来像是对 vtable 中 slot 0 中的值的调用，根据 Rustc 的
[vtable 布局代码][vtable-layout]，这是对该类型的 `drop_in_place` 的调用。

接下来，在第 16 行和第 17 行中，我们将类型的大小从 vtable 中加载出来，如果它不是零，则释放分配。

这是必要的，因为 `Box<dyn Counter>` 可能包装一个零大小的类型 (ZST)，但我们不允许有一个零大小的分配。

最后，我们将 `get_value` 的返回值移回到之前保存它的返回值寄存器 `rax` 中，并返回给调用者。

这也将与 `dyn*` 相关，因为 `dyn*` 也将是一个被拥有的类型。

[vtable-layout]: https://github.com/rust-lang/rust/blob/ff38c3528a6f6c94d45d803b7954a03997f3f4a9/compiler/rustc_middle/src/ty/vtable.rs#L45

### 那第 7 行和第 8 行呢？

你可能已经注意到，第 7 行和第 8 行似乎有些突兀。我们在栈上移动一些寄存器，然后再也不碰它们。

这个问题的答案与这篇文章有些离题，所以你可以跳到下一节，但如果你很好奇，你可以在这下面找到答案。

首先，上面的片段不是完整的反汇编代码。以下是完整的代码：

```assembly
1    push	r15
2    push	r14
3    push	rbx
4    sub	rsp, 16
5    mov	rbx, rsi
6    mov	r14, rdi
7    mov	qword ptr [rsp], rdi
8    mov	qword ptr [rsp + 8], rsi
9    mov	esi, 42
10   call	qword ptr [rbx + 32]
11   mov	rdi, r14
12   call	qword ptr [rbx + 24]
13   mov	r15, rax
14   mov	rdi, r14
15   call	qword ptr [rbx]
16   mov	rsi, qword ptr [rbx + 8]
17   test	rsi, rsi
18   je	.LBB4_5
19   mov	rdx, qword ptr [rbx + 16]
20   mov	rdi, r14
21   call	qword ptr [rip + __rust_dealloc@GOTPCREL]
22 
23 .LBB4_5:
24   mov	rax, r15
25   add	rsp, 16
26   pop	rbx
27   pop	r14
28   pop	r15
29   ret
30   
31   mov	r15, rax
32   mov	rsi, qword ptr [rbx + 8]
33   mov	rdx, qword ptr [rbx + 16]
34   mov	rdi, r14
35   call	alloc::alloc::box_free
36   jmp	.LBB4_7
37   
38   mov	r15, rax
39   mov	rdi, rsp
40   call	core::ptr::drop_in_place<alloc::boxed::Box<dyn playground::Counter>>
41 
42 .LBB4_7:
43   mov	rdi, r15
44   call	_Unwind_Resume@PLT
45   ud2
46   call	qword ptr [rip + core::panicking::panic_no_unwind@GOTPCREL]
47   ud2
```

这里的额外代码与 unwinding 有关。实际的调用边缘没有出现，但发生的是 Rust 编译器生成了一些额外的数据，它们实质上是“如果这个调用
panic 了，请返回到这里来运行清理代码。”

这种情况是通过 LLVM 的 [`invoke`] 指令发生的，它接受一个 `to` 标签（它是正常的返回路径）和一个 `unwind` 标签（当调用抛出异常，在 Rust
术语中是 panic，并且我们需要 unwind 时使用）。

[`invoke`]: https://llvm.org/docs/LangRef.html#invoke-instruction

在我看来，它像是创建了两个不同的着陆点 (landing pads)，一个从第 31 行开始，另一个从第 38 行开始。

我猜这些使用这些函数取决于抛出异常的调用。

例如，第 31 行似乎只调用 `box_frre` 来释放 Box，而没有对 Box 中的值调用 `drop`。

而第 38 行调用 `drop_in_place` 来丢弃 Box 中的内容并释放 Box 本身的分配。

请注意，这些步骤与第 14-21 行中的步骤相同，不同之处在于 LLVM 内联了对 `drop_in_place` 的调用。

LLVM 似乎对内联着陆点更加保守，这似乎是合理的，因为着陆点通常很少执行，所以节省代码大小比尽可能快地 unwinding 更重要。

无论如何，就是在第二个着陆点，我们看到第 7 和 8 行使用了存储。

在第 39 行，我们将栈指针移到 `rdi`，这是用于函数的第一个参数的寄存器。如果查看
[`drop_in_place`] 的签名，会发现它需要对正在 drop 的值的 `*mut`。

最初，`Box<dyn Counter>` 是在寄存器配对 `rdi:rsi` 中传递的，但寄存器没有内存地址。因此，为了能够在 unwinding
过程中 drop，我们需要将 Box 复制到栈上，以便可以将其用作其内存地址。

[`drop_in_place`]: https://doc.rust-lang.org/std/ptr/fn.drop_in_place.html

# 使用 `dyn*` 调用方法

因为现在 nightly Rust 中有 `dyn*` 可用，所以让我们比较一下它生成的程序集。

以下是使用 `dyn*` 的示例：

```rust,ignore
fn use_counter(mut counter: dyn* Counter) -> usize {
    counter.increment_by(42);
    counter.get_value()
} 
```

这将编译成以下汇编代码：

```assembly
1   push rbx
2   sub rsp, 16
3   mov rax, rsi
4   mov qword ptr [rsp], rdi
5   mov qword ptr [rsp + 8], rsi
6   mov rdi, rsp
7   mov esi, 42
8   call qword ptr [rax + 32]
9   mov rax, qword ptr [rsp + 8]
10  mov rdi, rsp
11  call qword ptr [rax + 24]
12  mov rbx, rax
13  mov rax, qword ptr [rsp + 8]
14  mov rdi, rsp
15  call qword ptr [rax]
16  mov rax, rbx
17  add rsp, 16
18  pop rbx
19  ret
```

这其中的大部分应该看起来很熟悉。我们调用了一些东西 + 32 和一些东西 +24，它们对应于前面示例中的两个方法调用。

此外，在最后调用了 `[rax]`，它对应于对析构函数的调用，就像我们在 `Box` 示例中看到的那样。

但这里有趣的部分似乎是第 4-6 行。那似乎正在将存储在 `rdi` 和 `rsi`
中的输入参数复制到栈上，然后当利用 vtable 进行调用时，将该指针作为 `self` 参数传递。但是为什么呢？

为了更清楚一些，让我们用 UFCS （即完全限定语法） 重写这个示例：

```rust,ignore
fn use_counter(mut counter: dyn* Counter) -> usize {
    <dyn* Counter as Counter>::increment_by(&mut counter, 42);
    <dyn* Counter as Counter>::get_value(&counter)
} 
```

请注意，之前我们将 `counter` 作为 `&mut dyn Counter`，但在函数调用中，我们使用了 `<dyn Counter as Counter>`，而没有 `&mut` 部分。

`counter` 作为 `dyn* Counter` 类型传入，并使用 `<dyn* Counter as Counter>` 来查找该方法。

这两个类型这次完全匹配，但这意味着编译器必须插入自动借用，因为 `dyn*Counter` 不是引用类型，而方法通过引用获取 `self`。

这就是为什么我们必须在参数位置给 `increment_by` 添加 `&mut`、给 `counter` 添加 `&`，而上次我们转换为 UFCC 时不必添加。

编译器基本上会在方法调用的任何时候插入这些自动借用。在许多情况下，由于内联、调用约定、数据布局等原因，它们最终是完全无成本的。

但现在这种情况下，它们不是无成本的，我们最终将 `dyn*` 复制到栈上，这样才可以获取它的地址。

有几个原因，但最主要的是编译器没有关于底层类型的信息，因此它不能内联方法调用。

# 结论：我们能让这一切变得更好吗？

我将通过提出这样一个问题来结束这篇文章：“我们能把这件事做得更好吗？”

我现在不回答这个问题，希望以后发的帖子能回答它。简而言之，我们不知道，但有一些想法可能会有所帮助。

一个挑战是，目前 `dyn*` 支持小值的内联存储。换句话说，如果你有一个实现 trait 的值，并且是指针大小的，比如
`usize`，那么可以直接将对象的值存储在 `dyn*` 的数据字段中，而不是存储指向它的指针。

这意味着，与 `&dyn Trait` 及类似的东西不同，我们不知道 `dyn*` 是否总是包含指针。有时是，有时不是，但这两个情况需要不同的处理方式。

目前，我们将所有内容都视为内联值，因此我们支持值的情况，但代价是当 `dyn*` 中的对象已经是指针时，引入不必要的间接性。

我们可以通过要求 `dyn*` 始终包装一个实际的指针而不只是一个指针大小的值来绕过这个问题。

这可能会使实现更加复杂，但或许是值得的。这使得小值用例的成本更高，但考虑到 `dyn*` 的主要用例是在 trait
对象中使用异步函数，并且我怀疑拥有指针大小的 Future 的情况相对较少，这可能是正确的权衡。

也就是说，通过在转换 trait 中增加一些额外的技巧，我们或许能够两全其美。

转换 trait 确实需要单独的博客文章，但如果你迫不及待想知道的话，我们确实有一些关于这个问题的早期会议的粗略 [设计笔记][design]。

[design]: https://hackmd.io/tNiv30enTJ6lGwnFx_lE6w

对于这两个想法是否会奏效，以及如果它们确实奏效，它们是否是正确的权衡，仍然存在一些分歧。

在以后的帖子中，我希望通过更深入地解决这些问题来帮助回答这些问题。

# 致谢

感谢 [Wesley Wiser](https://twitter.com/wesleywiser) 帮助我理解了这篇文章中的一些汇编指令。
