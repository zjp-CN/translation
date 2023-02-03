---

## title: "Stacked Borrows Implemented"
categories: internship rust
forum: https://internals.rust-lang.org/t/stacked-borrows-implemented/8847

## 标题：《堆积贷款实施》类别：实习生锈论坛：https://internals.rust-lang.org/t/stacked-borrows-implemented/8847

Three months ago, I proposed Stacked Borrows as a model for defining what kinds
of aliasing are allowed in Rust, and the idea of a \[validity invariant\]({%
post_url 2018-08-22-two-kinds-of-invariants %}) that has to be maintained by all
code at all times.  Since then I have been busy implementing both of these, and
developed Stacked Borrows further in doing so.  This post describes the latest
version of Stacked Borrows, and reports my findings from the implementation
phase: What worked, what did not, and what remains to be done.  There will also
be an opportunity for you to help the effort!

三个月前，我提出了一种定义Rust中允许哪些类型的别名的模型，并提出了[有效性不变量]({%post_url 2018-08-22-Two-Kind-Of-Instants%})的概念，它必须由所有代码始终维护。从那时起，我一直忙于实现这两个功能，并在此过程中进一步开发了堆叠借款。这篇文章描述了堆叠借阅的最新版本，并报告了我在实施阶段的发现：什么管用，什么不管用，还有什么需要做。也会有机会为您助力的！

This post is a self-contained introduction to Stacked Borrows.  Other than
historical curiosity and some comparison with my earlier work on
\[Types as Contracts\]({% post_url 2017-07-17-types-as-contracts %}) there is no
reason to read the \[original post\]({% post_url 2018-08-07-stacked-borrows %}) at
this point.

这篇文章是对堆叠借阅的一个完整的介绍。除了历史上的好奇心和我早先在[Types as Contracts]({%post_url 2017-07-17-Types-as-Contracts%})上的一些比较之外，没有理由在这一点上阅读[原始帖子]({%post_url 2018-08-07-STACKED-BORROWS%})。

<!-- MORE -->

What Stacked Borrows does is that it defines a semantics for Rust programs such
that some things about references always hold true for every valid execution
(meaning executions where no \[undefined behavior\]({% post_url
2017-07-14-undefined-behavior %}) occurred): `&mut` references are unique (we
can rely on no accesses by other functions happening to the memory they point
to), and `&` references are immutable (we can rely on no writes happening to the
memory they point to, unless there is an `UnsafeCell`).  Usually we have the
borrow checker guarding us against such nefarious violations of reference type
guarantees, but alas, when we are writing unsafe code, the borrow checker cannot
help us.  We have to define a set of rules that makes sense even for unsafe
code.

堆叠借用所做的是，它定义了Rust程序的语义，使得引用的某些内容对于每个有效的执行总是成立的(意味着没有出现[未定义行为]({%post_url 2017-07-14-unfined-behavior%})的执行)：`&mu`引用是唯一的(我们可以相信其他函数不会访问它们所指向的内存)，而`&`引用是不可变的(我们可以相信不会对它们所指向的内存进行写入，除非有`UnSafeCell`)。通常，我们有借用检查器来保护我们，防止这种对引用类型保证的恶意违反，但唉，当我们编写不安全的代码时，借用检查器无法帮助我们。我们必须定义一组即使对于不安全的代码也有意义的规则。

I will explain these rules again in this post.  The explanation is not going to
be the same as \[last time\]({% post_url 2018-08-07-stacked-borrows %}), not only
because it changed a bit, but also because I think I understand the model better
myself now so I can do a better job explaining it.

我将在这篇文章中再次解释这些规则。解释不会和[上次]({%post_url 2018-08-07-STACKED-LOWROWS%})相同，这不仅是因为它有一点变化，还因为我认为我现在更好地理解了这个模型，所以我可以更好地解释它。

Ready?  Let's get started.  I hope you brought some time, because this is a
rather lengthy post.  If you are not interested in a detailed description of
Stacked Borrows, you can skip most of this post and go right to [section 4](#4-differences-to-the-original-proposal).  If
you only want to know how to help, jump to [section 6](#6-evaluation-and-how-you-can-help).

准备好的?我们开始吧。我希望你带来了一些时间，因为这是一个相当长的帖子。如果你对堆叠借阅的详细描述不感兴趣，你可以跳过这篇文章的大部分内容，直接转到第4节。如果你只想知道如何帮助，请跳到第6节。

## 1 Enforcing Uniqueness

## 1强制执行唯一性

Let us first ignore the part about `&` references being immutable and focus on
uniqueness of mutable references.  Namely, we want to define our model in a way
that calling the following function will trigger undefined behavior:

让我们首先忽略关于`&`引用是不可变的部分，而关注可变引用的唯一性。也就是说，我们希望以这样一种方式定义我们的模型：调用以下函数将触发未定义的行为：

{% highlight rust %}
fn demo0() {
let x = &mut 1u8;
let y = &mut \*x;
\*y = 5;
// Write through a pointer aliasing `y`
\*x = 3;
// Use `y` again, asserting it is still exclusive
let \_val = \*y;
}
{% endhighlight %}

{%Highlight Rust%}fn demo0(){let x=&mut 1u8；let y=&mut*x；*y=5；//通过别名为`y`*x=3的指针写入；//再次使用`y`，断言它仍然是独占的let_val=*y；}{%endHighth%}

We want this function to be disallowed because between two uses of `y`, there is
a use of another pointer for the same location, violating the fact that `y`
should be unique.

我们希望这个函数不被允许，因为在两次使用`y`之间，同一位置使用了另一个指针，这违反了`y`应该是唯一的。

Notice that this function does not compile, the borrow checker won't allow it.
That's great!  It is undefined behavior, after all.  But the entire point of
this exercise is to explain *why* we have undefined behavior here *without*
referring to the borrow checker, because we want to have rules that also work
for unsafe code.  In fact, you could say that retroactively, these rules explain
why the borrow checker works the way it does:  We can pretend that the model came
first, and the borrow checker is merely doing compile-time checks to make sure
we follow the rules of the model.

请注意，此函数不能编译，借用检查器不允许它。太好了!毕竟，这是一种未定义的行为。但本练习的整个要点是解释为什么我们在这里有未定义的行为，而不是引用借用检查器，因为我们希望有也适用于不安全代码的规则。事实上，您可以说，这些规则可以追溯地解释借用检查器为什么会这样工作：我们可以假装模型先出现，而借用检查器只是执行编译时检查，以确保我们遵循模型的规则。

To be able to do this, we have to pretend our machine has two things which real
CPUs do not have.  This is an example of adding "shadow state" or "instrumented
state" to the "virtual machine" that we \[use to specify Rust\]({% post_url
2017-06-06-MIR-semantics %}).  This is not an uncommon approach, often times
source languages make distinctions that do not appear in the actual hardware.  A
related example is
[valgrind's memcheck](http://valgrind.org/docs/manual/mc-manual.html) which
keeps track of which memory is initialized to be able to detect memory errors:
During a normal execution, uninitialized memory looks just like all other
memory, but to figure out whether the program is violating C's memory rules, we
have to keep track of some extra state.

要做到这一点，我们必须假装我们的机器有两个真正的CPU没有的东西。这是一个向我们[用于指定Rust]({%post_url 2017-06-06-MIR-Semantics%})的“虚拟机”添加“影子状态”或“检测状态”的示例。这并不是一种罕见的方法，源语言经常会做出实际硬件中不会出现的区别。一个相关的例子是valgrind的Memcheck，它跟踪哪个内存被初始化以能够检测内存错误：在正常执行期间，未初始化的内存看起来就像所有其他内存一样，但要确定程序是否违反了C的内存规则，我们必须跟踪一些额外的状态。

For stacked borrows, the extra state looks as follows:

对于堆叠借用，额外状态如下所示：

1. For every pointer, we keep track of an extra "tag" that records when and how
   this pointer was created.
1. For every location in memory, we keep track of a stack of "items", indicating
   which tag a pointer must have to be allowed to access this location.

These exist separately, i.e., when a pointer is stored in memory, then we both
have a tag stored as part of this pointer value (remember,
\[bytes are more than `u8`\]({% post_url 2018-07-24-pointers-and-bytes %})), and
every byte occupied by the pointer has a stack regulating access to this
location.  Also these two do not interact, i.e., when loading a pointer from
memory, we just load the tag that was stored as part of this pointer.  The stack
of a location, and the tag of a pointer stored at some location, do not have any
effect on each other.

对于每个指针，我们跟踪记录该指针何时以及如何创建的额外的“标记”。对于存储器中的每个位置，我们跟踪“项”的堆栈，指示必须允许指针访问该位置的标记。这些项单独存在，即，当指针存储在存储器中时，我们都有一个标记存储为该指针值的一部分(记住，[字节数大于`u8`]({%post_url 2018-07-24-Points-and-byte%}))，指针占用的每个字节都有一个堆栈来管理对该位置的访问。这两者也不交互，也就是说，当从内存加载指针时，我们只加载作为该指针的一部分存储的标记。位置的堆栈和存储在某个位置的指针的标记彼此之间没有任何影响。

In our example, there are two pointers (`x` and `y`) and one location of
interest (the one both of these pointers point to, initialized with `1u8`).
When we initially create `x`, it gets tagged `Uniq(0)` to indicate that it is a
unique reference, and the location's stack has `Uniq(0)` at its top to indicate
that this is the latest reference allowed to access said location.  When we
create `y`, it gets a new tag, `Uniq(1)`, so that we can distinguish it from
`x`.  We also push `Uniq(1)` onto the stack, indicating not only that `Uniq(1)`
is the latest reference allow to access, but also that it is "derived from"
`Uniq(0)`: The tags higher up in the stack are descendants of the ones further
down.

在我们的示例中，有两个指针(`x`和`y`)和一个感兴趣的位置(这两个指针都指向的位置，初始化为`1u8`)。当我们最初创建`x`时，它被标记为`uniq(0)`，表示它是唯一的引用，位置的堆栈的顶部有`uniq(0)`，表示这是允许访问该位置的最新引用。当我们创建`y`时，它会有一个新的标签`uniq(1)`，这样我们就可以将它与`x`区分开来。我们还将`uniq(1)`推送到堆栈上，不仅表明`uniq(1)`是允许访问的最新引用，而且还表明它是从`uniq(0)`派生的：堆栈中更高的标签是更低的标签的后代。

So after both references are created, we have: `x` tagged `Uniq(0)`, `y` tagged
`Uniq(1)`, and the stack contains `[Uniq(0), Uniq(1)]`. (Top of the stack is on
the right.)

所以在创建两个引用后，我们有：`x`标记为`uniq(0)`，`y`标记为`uniq(1)`，堆栈包含`[uniq(0)，uniq(1)]`。(堆栈的顶部在右侧。)

When we use `y` to access the location, we make sure its tag is at the top of
the stack: check, no problem here.  When we use `x`, we do the same thing: Since
it is not at the top yet, we pop the stack until it is, which is easy.  Now the
stack is just `[Uniq(0)]`.  Now we use `y` again and... blast!  Its tag is not
on the stack.  We have undefined behavior.

当我们使用`y‘访问位置时，我们确保它的标签在堆栈的顶部：检查，这里没有问题。当我们使用`x`时，我们做同样的事情：因为它还不在顶部，我们弹出堆栈，直到它在顶部，这很容易。现在堆栈只是`[uniq(0)]`。现在我们再用‘y’，然后..。他妈的！其标记不在堆栈上。我们有不确定的行为。

In case you got lost, here is the source code with comments indicating the tags
and the stack of the one location that interests us:

如果您迷路了，以下是源代码，带有注释，指示我们感兴趣的一个位置的标签和堆栈：

{% highlight rust %}
fn demo0() {
let x = &mut 1u8; // tag: `Uniq(0)`
// stack: \[Uniq(0)\]

{%Highlight Rust%}fn demo0(){let x=&mut 1u8；//tag：`uniq(0)`//栈：[uniq(0)]

let y = &mut \*x; // tag: `Uniq(1)`
// stack: \[Uniq(0), Uniq(1)\]

设y=&mut*x；//tag：`uniq(1)`//栈：[uniq(0)，uniq(1)]

// Pop until `Uniq(1)`, the tag of `y`, is on top of the stack:
// Nothing changes.
\*y = 5;
// stack: \[Uniq(0), Uniq(1)\]

//弹出到`y`的标签`uniq(1)`位于堆栈顶部：//没有变化。*y=5；//堆栈：[uniq(0)，uniq(1)]

// Pop until `Uniq(0)`, the tag of `x`, is on top of the stack:
// We pop `Uniq(1)`.
\*x = 3;
// stack: \[Uniq(0)\]

//弹出到`x`的标签`uniq(0)`在堆栈顶部：//我们弹出`uniq(1)`*x=3；//堆栈：[uniq(0)]

// Pop until `Uniq(1)`, the tag of `y`, is on top of the stack:
// That is not possible, hence we have undefined behavior.
let \_val = \*y;
}
{% endhighlight %}

//弹出到`y`的标签`uniq(1)`位于堆栈顶部：//这是不可能的，因此我们有未定义的行为。Let_val=*y；}{%endHighth%}

Well, actually having undefined behavior here is good news, since that's what we
wanted from the start!  And since there is an implementation of the model in
[Miri](https://github.com/solson/miri/), you can try this yourself: The amazing
@shepmaster has integrated Miri into the playground, so you can
[put the example there](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=d15868687f79072688a0d0dd1e053721)
(adjusting it slightly to circumvent the borrow checker), then select "Tools -
Miri" and it will complain (together with a rather unreadable backtrace, we sure
have to improve that one):

实际上，在这里有未定义的行为是个好消息，因为这是我们从一开始就想要的！由于在Miri中实现了该模型，您可以自己尝试一下：令人惊叹的@shepmaster已将Miri集成到操场中，因此您可以将示例放在那里(略微调整它以绕过借用检查器)，然后选择“Tools-Miri”，它会发出警告(连同一个相当难读的回溯，我们肯定要改进那个回溯)：

```
error[E0080]: constant evaluation error: Borrow being dereferenced (Uniq(1037)) does not exist on the stack
 --> src/main.rs:6:14
  |
6 |   let _val = *y;
  |              ^^ Borrow being dereferenced (Uniq(1037)) does not exist on the stack
  |
```

## 2 Enabling Sharing

## 2启用共享

If we just had unique pointers, Rust would be a rather dull language.  Luckily
enough, there are also two ways to have shared access to a location: through
shared references (safely), and through raw pointers (unsafely).  Moreover,
shared references *sometimes* (but not when they point to an `UnsafeCell`)
assert an additional guarantee: Their destination is immutable.

如果我们只有独特的指针，铁锈将是一门相当枯燥的语言。幸运的是，对某个位置的共享访问也有两种方式：通过共享引用(安全)和通过原始指针(不安全)。此外，共享引用有时(但当它们指向`UnSafeCell`时不会)断言一个额外的保证：它们的目的地是不可变的。

For example, we want the following code to be allowed -- not least because this
is actually safe code accepted by the borrow checker, so we better make sure
this is not undefined behavior:

例如，我们希望允许以下代码--尤其是因为这实际上是借入检查器接受的安全代码，所以我们最好确保这不是未定义的行为：

{% highlight rust %}
fn demo1() {
let x = &mut 1u8;
// Create several shared references, and we can also still read from `x`
let y1 = &\*x;
let \_val = \*x;
let y2 = &\*x;
let \_val = \*y1;
let \_val = \*y2;
}
{% endhighlight %}

{%Highlight Rust%}fn demo1(){let x=&mut 1u8；//创建多个共享引用，我们仍然可以从`x`let y1=&*x；let_val=*x；let y2=&*x；let_val=*y1；let_val=*y2；}{%endHighth%%}读取

However, the following code is *not* okay:

但是，以下代码有问题：

{% highlight rust %}
fn demo2() {
let x = &mut 1u8;
let y = &\*x;
// Create raw reference aliasing `y` and write through it
let z = x as \*const u8 as \*mut u8;
unsafe { \*z = 3; }
// Use `y` again, asserting it still points to the same value
let \_val = \*y;
}
{% endhighlight %}

{%Highlight Rust%}fn demo2(){let x=&mut 1u8；let y=&*x；//创建原始引用别名`y`并写过它，让z=x as*const u8 as*mut U8；unSafe{*z=3；}//再次使用`y`，断言它仍然指向相同的值let_val=*y；}{%endHighlight%}

If you
[try this in Miri](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=1bc8c2f432941d02246fea0808e2e4f4),
you will see it complain:

如果你在Miri上尝试这一点，你会看到它抱怨：

```
 --> src/main.rs:6:14
  |
6 |   let _val = *y;
  |              ^^ Location is not frozen long enough
  |
```

How is it doing that, and what is a "frozen" location?

它是如何做到这一点的，什么是“冻结”的地点？

To explain this, we have to extend the "shadow state" of our "virtual machine" a
bit.  First of all, we introduce a new kind of tag that a pointer can carry: A
*shared* tag.  The following Rust type describes the possible tags of a pointer:

要解释这一点，我们必须稍微扩展一下“虚拟机”的“影子状态”。首先，我们介绍了一种指针可以携带的新标签：共享标签。下面的铁锈类型描述了指针的可能标记：

{% highlight rust %}
pub type Timestamp = u64;
pub enum Borrow {
Uniq(Timestamp),
Shr(Option<Timestamp>),
}
{% endhighlight %}

{%Highlight Rust%}pub type Timestamp=U64；pub枚举借入{uniq(时间戳)，Shr(选项)，}{%endHighlight%}

You can think of the timestamp as a unique ID, but as we will see, for shared
references, it is also important to be able to determine which of these IDs was
created first.  The timestamp is optional in the shared tag because that tag is
also used by raw pointers, and for raw pointers, we are often not able to track
when and how they are created (for example, when raw pointers are converted to
integers and back).

您可以将时间戳视为唯一的ID，但正如我们将看到的，对于共享引用，能够确定这些ID中的哪个ID是首先创建的也很重要。时间戳在共享标记中是可选的，因为该标记也由原始指针使用，并且对于原始指针，我们通常无法跟踪它们是何时以及如何创建的(例如，原始指针何时被转换为整数并被转换回来)。

We use a separate type for the items on our stack, because there we do not need
a timestamp for shared pointers:

我们对堆栈上的项使用单独的类型，因为在那里我们不需要共享指针的时间戳：

{% highlight rust %}
pub enum BorStackItem {
Uniq(Timestamp),
Shr,
}
{% endhighlight %}

{%Highlight Rust%}pub enum BorStackItem{uniq(时间戳)，Shr，}{%endHighth%}

And finally, a "borrow stack" consists of a stack of `BorStackItem`, together
with an indication of whether the stack (and the location it governs) is
currently *frozen*, meaning it may only be read, not written:

最后，“借入堆栈”由一个`BorStackItem`堆栈组成，并指示该堆栈(及其管理的位置)当前是否处于冻结状态，这意味着它只能读而不能写：

{% highlight rust %}
pub struct Stack {
borrows: Vec<BorStackItem>, // used as a stack; never empty
frozen_since: Option<Timestamp>, // virtual frozen "item" on top of the stack
}
{% endhighlight %}

{%Highlight Rust%}pub struct Stack{借入：vec，//用作堆栈；从不为空冻结_自：选项，//堆栈顶部的虚拟冻结“项目”}{%endHighth%}

### 2.1 Executing the Examples

### 2.1执行示例

Let us now look at what happens when we execute our two example programs.  To
this end, I will embed comments in the source code.  There is only one location
of interest here, so whenever I talk about a "stack", I am referring to the
stack of that location.

现在让我们来看看当我们执行两个示例程序时会发生什么。为此，我将在源代码中嵌入注释。这里只有一个感兴趣的位置，所以每当我谈到“栈”时，我指的是该位置的栈。

{% highlight rust %}
fn demo1() {
let x = &mut 1u8; // tag: `Uniq(0)`
// stack: \[Uniq(0)\]; not frozen

{%Highlight Rust%}fn demo1(){let x=&mut 1u8；//tag：`uniq(0)`//栈：[uniq(0)]；未冻结

let y1 = &\*x; // tag: `Shr(Some(1))`
// stack: \[Uniq(0), Shr\]; frozen since 1

设y1=&*x；//tag：`Shr(Some(1))`//堆栈：[uniq(0)，Shr]；从1开始冻结

// Access through `x`.  We first check whether its tag `Uniq(0)` is in the
// stack (it is).  Next, we make sure that either our item *or* `Shr` is on
// top *or* the location is frozen.  The latter is the case, so we go on.
let \_val = \*x;
// stack: \[Uniq(0), Shr\]; frozen since 1

//通过`x`访问。我们首先检查它的标签`uniq(0)`是否在//堆栈中(它在)。接下来，我们确保我们的项目或`Shr`在//顶部，或者位置被冻结。后者就是这种情况，所以我们继续。Let_val=*x；//堆栈：[uniq(0)，Shr]；从1开始冻结

// This is not an access, but we still dereference `x`, so we do the same
// actions as on a read.  Just like in the previous line, nothing happens.
let y2 = &\*x; // tag: `Shr(Some(2))`
// stack: \[Uniq(0), Shr\]; frozen since 1

//这不是访问，但我们仍然取消了对`x`的引用，所以我们的操作与Read相同//。就像上一行一样，什么都不会发生。设y2=&*x；//tag：`Shr(Some(2))`//堆栈：[uniq(0)，Shr]；从1开始冻结

// Access through `y1`.  Since the shared tag has a timestamp (1) and the type
// (`u8`) does not allow interior mutability (no `UnsafeCell`), we check that
// the location is frozen since (at least) that timestamp.  It is.
let \_val = \*y1;
// stack: \[Uniq(0), Shr\]; frozen since 1

//通过`y1`访问。因为共享标签有时间戳(1)，并且类型//(`u8`)不允许内部可变(没有`UnSafeCell`)，所以我们检查//位置从该时间戳起(至少)是冻结的。它是。Let_val=*y1；//堆栈：[uniq(0)，Shr]；从1开始冻结

// Same as with `y2`: The location is frozen at least since 2 (actually, it
// is frozen since 1), so we are good.
let \_val = \*y2;
// stack: \[Uniq(0), Shr\]; frozen since 1
}
{% endhighlight %}

//和`y2`一样：位置至少从2开始冻结(实际上是从1开始冻结)，所以我们很好。Let_val=*y2；//堆栈：[uniq(0)，Shr]；从1开始冻结}{%endHighth%}

This example demonstrates a few new aspects.  First of all, there are actually
two operations that perform tag-related checks in this model (so far):
Dereferencing a pointer (whenever you have a `*`, also implicitly), and actual
memory accesses.  Operations like `&*x` are an example of operations that
dereference a pointer without accessing memory.  Secondly, *reading* through a
mutable reference is actually okay *even when that reference is not exclusive*.
It is only *writing* through a mutable reference that "re-asserts" its
exclusivity.  I will come back to these points later, but let us first go
through another example.

这个例子展示了一些新的方面。首先，在这个模型中，实际上有两个操作执行与标记相关的检查(到目前为止)：取消引用指针(无论何时有`*`，也是隐式的)和实际的内存访问。像`&*x`这样的操作是在不访问内存的情况下取消引用指针的操作的示例。其次，读取可变引用实际上是可以的，即使该引用不是独占的。它只是通过一个可变的引用来“重新断言”它的排他性。我稍后会再谈这些问题，但让我们先来看看另一个例子。

{% highlight rust %}
fn demo2() {
let x = &mut 1u8; // tag: `Uniq(0)`
// stack: \[Uniq(0)\]; not frozen

{%Highlight Rust%}fn demo2(){let x=&mut 1u8；//tag：`uniq(0)`//栈：[uniq(0)]；未冻结

let y = &\*x; // tag: `Shr(Some(1))`
// stack: \[Uniq(0), Shr\]; frozen since 1

设y=&*x；//tag：`Shr(Some(1))`//堆栈：[uniq(0)，Shr]；从1开始冻结

// The `x` here really is a `&*x`, but we have already seen above what
// happens: `Uniq(0)` must be in the stack, but we leave it unchanged.
let z = x as \*const u8 as \*mut u8; // tag erased: `Shr(None)`
// stack: \[Uniq(0), Shr\]; frozen since 1

//这里的`x`实际上是一个`&*x`，但是我们已经看到了上面//发生的事情：`uniq(0)`必须在堆栈中，但我们保持它不变。设z=x as*const U8 as*mut U8；//标签擦除：`Shr(None)`//堆栈：[Uniq(0)，Shr]；从1开始冻结

// A write access through a raw pointer: Unfreeze the location and make sure
// that `Shr` is at the top of the stack.
unsafe { \*z = 3; }
// stack: \[Uniq(0), Shr\]; not frozen

//通过原始指针进行写访问：解冻位置，//确保`Shr`位于堆栈顶部。不安全的{*z=3；}//堆栈：[uniq(0)，shr]；未冻结

// Access through `y`.  There is a timestamp in the `Shr` tag, and the type
// `u8` does not allow interior mutability, but the location is not frozen.
// This is undefined behavior.
let \_val = \*y;
}
{% endhighlight %}

//通过`y`访问。`Shr`标签中有时间戳，类型//`u8`不允许内部可变，但位置不会冻结。//这是未定义的行为。Let_val=*y；}{%endHighth%}

### 2.2 Dereferencing a Pointer

### 2.2取消引用指针

As we have seen, we consider the tag of a pointer already when dereferencing it,
before any memory access happens.  The operation on a dereference never mutates
the stack, but it performs some basic checks that might declare the program UB.
The reason for this is twofold: First of all, I think we should require some
basic validity for pointers that are dereferenced even when they do not access
memory. Secondly, there is the practical concern for the implementation in Miri:
When we dereference a pointer, we are guaranteed to have type information
available (crucial for things that depend on the presence of an `UnsafeCell`),
whereas having type information on every memory access would be quite hard to
achieve in Miri.

正如我们已经看到的，在任何内存访问发生之前，我们在取消对指针的引用时已经考虑了它的标记。取消引用的操作永远不会改变堆栈，但它会执行一些基本检查，这些检查可能会声明程序UB。这样做的原因有两个：首先，我认为我们应该为即使在不访问内存的情况下解除引用的指针要求一些基本的有效性。其次，在MIRI中实现有一个实际问题：当我们取消引用一个指针时，我们保证有可用的类型信息(对于依赖于“不安全单元”的存在至关重要)，而拥有关于每次内存访问的类型信息在MIRI中是很难实现的。

Notice that on a dereference, we have *both* a tag at the pointer *and* the type
of a pointer, and the two might not agree, which we do not always want to rule
out (after a `transmute`, we might have raw or shared pointers with a unique
tag, for example).

注意，在取消引用时，我们在指针和指针的类型上都有一个标记，这两者可能不一致，我们并不总是想要排除这一点(例如，在“变换”之后，我们可能具有具有唯一标记的原始或共享指针)。

The following checks are done on every pointer dereference, for every location
covered by the pointer (`size_of_val` tells us how many bytes the pointer
covers):

对于指针覆盖的每个位置，在每个指针取消引用时都会执行以下检查(`size_of_val`告诉我们指针覆盖的字节数)：

1. If this is a raw pointer, do nothing.  Raw accesses are checked as little as possible.
1. If this is a unique reference and the tag is `Shr(Some(_))`, that's an error.
1. If the tag is `Uniq`, make sure there is a matching `Uniq` item with the same
   ID on the stack.
1. If the tag is `Shr(None)`, make sure that either the location is frozen or
   else there is a `Shr` item on the stack.
1. If the tag is `Shr(Some(t))`, then the check depends on whether the location
   is inside an `UnsafeCell` or not, according to the type of the reference.
   * Locations outside `UnsafeCell` must have `frozen_since` set to `t` or an
     older timestamp.
   * `UnsafeCell` locations must either be frozen or else have a `Shr` item in
     their stack (same check as if the tag had no timestamp).

### 2.3 Accessing Memory

### 如果这是一个原始指针，则什么都不做。尽可能少地检查原始访问。如果这是唯一引用并且标签是`Shr(Some(_))`，则这是一个错误。如果标签是`Uniq`，则确保堆栈上存在具有相同ID的匹配的`Uniq`项。如果标签是`Shr(None)`，则确保位置被冻结或者堆栈上有`Shr`项。如果标签是`Shr(Some(T))`，则检查取决于该位置是否在`UnSafeCell`内，根据引用的类型。`UnSafeCell`之外的位置必须将`Freeze_since`设置为`t`或更早的时间戳。`UnSafeCell`位置必须被冻结或在其堆栈中有一个`Shr`项(与标记没有时间戳的检查相同)。2.3访问内存

On an actual memory access, we know the tag of the pointer that was used to
access (we always use the actual tag and disregard the type of the pointer), and
we know whether we are reading from or writing to the current location.  We
perform the following operations on all locations affected by the access:

在实际的内存访问中，我们知道用于访问的指针的标记(我们总是使用实际的标记，而忽略指针的类型)，并且我们知道我们是从当前位置读取还是向当前位置写入。我们在受访问影响的所有位置上执行以下操作：

1. If the location is frozen and this is a read access, nothing happens (even
   if the tag is `Uniq`).
   
   如果位置被冻结，并且这是读访问，则不会发生任何事情(即使标签是`Uniq`)。

1. Otherwise, if this is a write access, unfreeze the location (set
   `frozen_since` to `None`).  (If this is a read access and we come here, the
   location is already unfrozen.)
   
   否则，如果这是写访问，则解冻位置(将`冻结_自`设置为`无`)。(如果这是读访问权限，而我们来到这里，则该位置已解冻。)

1. Pop the stack until the top item matches the tag of the pointer.
   
   弹出堆栈，直到顶部的项与指针的标记匹配。
   
   * A `Uniq` item matches a `Uniq` tag with the same ID.
   * A `Shr` item matches any `Shr` tag (with or without timestamp).
   * When we are reading, a `Shr` item matches a `Uniq` tag.
   If we pop the entire stack without finding a match, then we have undefined
   behavior.
   
   \`Uniq`项匹配具有相同ID的`Uniq`标签。`Shr`项匹配任何`Shr`标签(有或没有时间戳)。当我们阅读时，`Shr`项匹配`Uniq`标签。如果我们弹出整个堆栈但没有找到匹配，那么我们就有未定义的行为。

To understand these rules better, try going back through the three examples we
have seen so far and applying these rules for dereferencing pointers and
accessing memory to understand how they interact.

为了更好地理解这些规则，请尝试回顾我们到目前为止看到的三个示例，并应用这些规则来取消引用指针和访问内存，以了解它们是如何交互的。

The most subtle point here is that we make a `Uniq` tag match a `Shr` item and
also accept `Uniq` reads on frozen locations.  This is required to make `demo1`
work: Rust permits read accesses through mutable references even when they are
not currently actually unique.  Our model hence has to do the same.

这里最微妙的一点是，我们让`Uniq`标签匹配`Shr`项，并接受冻结位置的`Uniq`读取。这是使`demo1‘工作所必需的：Ruust允许通过可变引用进行读访问，即使它们当前实际上不是唯一的。因此，我们的模型也必须这样做。

## 3 Retagging and Creating Raw Pointers

## 3重新标记和创建原始指针

We have talked quite a bit about what happens when we *use* a pointer.  It is
time we take a close look at *how pointers are created*.  However, before we go
there, I would like us to consider one more example:

我们已经谈了相当多关于使用指针时会发生什么。现在是我们仔细看看指针是如何创建的时候了。然而，在我们谈到这一点之前，我想让我们再考虑一个例子：

{% highlight rust %}
fn demo3(x: &mut u8) -> u8 {
some_function();
\*x
}
{% endhighlight %}

{%Highlight Rust%}FN demo3(x：&mut U8)->U8{Some_Function()；*x}{%endHighlight%}

The question is: Can we move the load of `x` to before the function call?
Remember that the entire point of Stacked Borrows is to enforce a certain
discipline when using references, in particular, to enforce uniqueness of
mutable references.  So we should hope that the answer to that question is "yes"
(and that, in turn, is good because we might use it for optimizations).
Unfortunately, things are not so easy.

问题是：我们可以将`x`的负载移到函数调用之前吗？请记住，堆叠借用的全部目的是在使用引用时强制执行特定规则，特别是强制可变引用的唯一性。因此，我们应该希望这个问题的答案是“是的”(这反过来是好的，因为我们可以将其用于优化)。不幸的是，事情并不是那么容易。

The uniqueness of mutable references entirely rests on the fact that the pointer
has a unique tag: If our tag is at the top of the stack (and the location is not
frozen), then any access with another tag will pop our item from the stack (or
cause undefined behavior).  This is ensured by the memory access checks.  Hence,
if our tag is *still* on the stack after some other accesses happened (and we
know it is still on the stack every time we dereference the pointer, as per the
dereference checks described above), we know that no access through a pointer
with a different tag can have happened.

可变引用的唯一性完全取决于这样一个事实，即指针具有唯一的标记：如果我们的标记位于堆栈的顶部(并且位置没有被冻结)，则对另一个标记的任何访问都将从堆栈中弹出我们的项(或者导致未定义的行为)。这是由存储器访问检查确保的。因此，如果在发生了一些其他访问之后，我们的标记仍然在堆栈上(并且我们知道，每次我们取消引用指针时，它仍然在堆栈上，根据上面描述的取消引用检查)，我们知道不可能发生通过具有不同标记的指针的访问。

### 3.1 Guaranteed Freshness

### 3.1保证新鲜度

However, what if `some_function` has an exact copy of `x`?  We got `x` from our
caller (whom we do not trust), maybe they used that same tag for another
reference (copied it with `transmute_copy` or so) and gave that to
`some_function`?  There is a simple way we can circumvent this concern: Generate
a new tag for `x`.  If *we* generate the tag (and we know generation never emits
the same tag twice, which is easy), we can be sure this tag is not used for any
other reference.  So let us make this explicit by putting a `Retag` instruction
into the code where we generate new tags:

但是，如果`某些_函数`具有与`x`完全相同的副本，该怎么办？我们从调用者(我们不信任的调用者)那里获得了`x`，也许他们将相同的标记用于另一个引用(使用`Transmute_Copy`左右复制它)，并将其提供给`Some_Function`？有一个简单的方法可以绕过这个问题：为`x`生成一个新的标签。如果我们生成标记(我们知道生成不会发出相同的标记两次，这很容易)，我们可以确保此标记不用于任何其他引用。因此，让我们通过将`Retag`指令放入我们生成新标记的代码中来明确这一点：

{% highlight rust %}
fn demo3(x: &mut u8) -> u8 {
Retag(x);
some_function();
\*x
}
{% endhighlight %}

{%Highlight Rust%}FN demo3(x：&mut U8)->U8{retag(X)；Some_Function()；*x}{%endHighlight%}

These `Retag` instructions are inserted by the compiler pretty much any time
references are copied: At the beginning of every function, all inputs of
reference type get retagged.  On every assignment, if the assigned value is of
reference type, it gets retagged.  Moreover, we do this even when the reference
value is inside the field of a `struct` or `enum`, to make sure we really cover
all references.  (This recursive descent is already implemented, but the
implementation has not landed yet.)  Finally, `Box` is treated like a mutable
reference, to encode that it asserts unique access.  However, we do *not*
descend recursively through references: Retagging a `&mut &mut u8` will only
retag the *outer* reference.

几乎在复制引用的任何时候，编译器都会插入这些‘Retager’指令：在每个函数的开头，所有引用类型的输入都会被重新标记。在每次赋值时，如果赋值类型为引用类型，则会重新标记该值。此外，即使当引用值位于`struct`或`枚举`的字段内时，我们也会这样做，以确保我们真的覆盖了所有引用。(这种递归下降已经实现，但实现尚未实现。)最后，`Box`被视为可变引用，以编码它断言唯一访问。然而，我们不会通过引用递归下降：重新标记一个`&mut&mut u8`只会重新标记外部引用。

Retagging is the *only* operation that generates fresh tags.  Taking a reference
simply forwards the tag of the pointer we are basing this reference on.

重新标记是唯一可以生成新标签的操作。引用只是转发我们引用所基于的指针的标记。

Here is our very first example with explicit retagging:

以下是我们的第一个显式重新标记的示例：

{% highlight rust %}
fn demo0() {
let x = &mut 1u8; // nothing interesting happens here
Retag(x); // tag of `x` gets changed to `Uniq(0)`
// stack: \[Uniq(0)\]; not frozen

{%Highlight Rust%}fn demo0(){let x=&mut 1u8；//retag(X)；//`x`的标签变为`uniq(0)`//栈：[uniq(0)]；未冻结

let y = &mut \*x; // nothing interesting happens here
Retag(y); // tag of `y` gets changed to `Uniq(1)`
// stack: \[Uniq(0), Uniq(1)\]; not frozen

让y=&mut*x；//这里没有什么有趣的事情，retag(Y)；//`y`的标签变成了`uniq(1)`//栈：[uniq(0)，uniq(1)]；没有冻结

// Check that `Uniq(1)` is on the stack, then pop to bring it to the top.
\*y = 5;
// stack: \[Uniq(0), Uniq(1)\]; not frozen

//检查`uniq(1)`是否在堆栈上，弹出后放到最上面。*y=5；//堆栈：[uniq(0)，uniq(1)]；未冻结

// Check that `Uniq(0)` is on the stack, then pop to bring it to the top.
\*x = 3;
// stack: \[Uniq(0)\]; not frozen

//检查`uniq(0)`是否在堆栈上，弹出后放到最上面。*x=3；//堆栈：[uniq(0)]；未冻结

// Check that `Uniq(1)` is on the stack -- it is not, hence UB.
let \_val = \*y;
}
{% endhighlight %}

//检查`uniq(1)`是否在堆栈上--它不在堆栈上，因此为ub。Let_val=*y；}{%endHighth%}

For each reference and `Box`, `Retag` does the following (we will slightly
refine these instructions later) on all locations covered by the reference
(again, according to `size_of_val`):

对于每个引用和`Box`，`Retag`对引用覆盖的所有位置执行以下操作(我们稍后将略微细化这些指令)(同样，根据`Size_of_val`)：

1. Compute a fresh tag: `Uniq(_)` for mutable references and `Box`, and
   `Shr(Some(_))` for shared references.
1. Perform the checks that would also happen when we dereference this reference.
1. Perform the actions that would also happen when an actual access happens
   through this reference (for shared references a read access, for mutable
   references a write access).
1. Check if the new tag is `Shr(Some(t))` and the location is inside an `UnsafeCell`.
   * If both conditions apply, freeze the location with timestamp `t`.  If it
     is already frozen, do nothing.
   * Otherwise, push a new item onto the stack: `Shr` if the tag is a `Shr(_)`,
     `Uniq(id)` if the tag is `Uniq(id)`.

One high-level way to think about retagging is that it computes a fresh tag, and
then performs a reborrow of the old reference with the new tag.

计算一个新的标签：`uniq(_)`表示可变引用，`Box`表示共享引用，`Shr(Some(_))`表示共享引用。执行取消引用时也会发生的检查。执行通过该引用进行实际访问时也会发生的操作(对于共享引用是读访问，对于可变引用是写访问)。检查新标记是否为`Shr(Some(T))`以及位置是否在`UnSafeCell`内。如果这两个条件都适用，则使用时间戳`t`冻结位置。如果已经冻结，则不执行任何操作。否则，将一个新项推送到堆栈上：如果标记是`Shr(_)`，则将新项推送到堆栈上；如果标记是`uniq(Id)`，则推送到`uniq(Id)`。考虑重新标记的一个高级方法是计算一个新标记，然后使用新标记重新借用旧引用。

### 3.2 When Pointers Escape

### 3.2指针转义时

Creating a shared reference is not the only way to share a location: We can also
create raw pointers, and if we are careful enough, use them to access a location
from different aliasing pointers.  (Of course, "careful enough" is not very
precise, but the precise answer is the very model I am describing here.)

创建共享引用并不是共享位置的唯一方法：我们还可以创建原始指针，如果足够小心，可以使用它们从不同的别名指针访问位置。(当然，“足够小心”并不是非常准确，但准确的答案就是我在这里描述的模型。)

To account for this, we consider the act of casting to a raw pointer as a
special way of creating a reference
(["creating a raw reference"](https://github.com/rust-lang/rfcs/pull/2582), so
to speak).  As usual for creating a new reference, that operation is followed by
retagging.  This retagging is special though because unlike normal retagging it
acts on raw pointers.  Consider the
[following example](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015&gist=253868e96b7eba85ef28e1eabd557f66):

为了说明这一点，我们将强制转换为原始指针的行为视为创建引用的一种特殊方式(所谓的“创建原始引用”)。像通常创建新引用一样，该操作之后是重新标记。然而，这种重新标记是特殊的，因为与普通的重新标记不同，它作用于原始指针。请考虑以下示例：

{% highlight rust %}
fn demo4() {
let x = &mut 1u8;
Retag(x); // tag of `x` gets changed to `Uniq(0)`
// stack: \[Uniq(0)\]; not frozen

{%Highlight Rust%}fn demo4(){let x=&mut 1u8；retag(X)；//`x`的标签改为`uniq(0)`//栈：[uniq(0)]；未冻结

let y1 = x as \*mut u8;
// Make sure what `x` points to is accessible through raw pointers.
Retag(\[raw\] y1) // tag of `y` gets erased to `Shr(None)`
// stack: \[Uniq(0), Shr\]; not frozen

设y1=x为*mut u8；//确保通过原始指针可以访问`x`指向的内容。Retag([原始]y1)//`y`的标签擦除为`Shr(无)`//堆栈：[uniq(0)，Shr]；未冻结

let y2 = y1;
unsafe {
// All of these first dereference a raw pointer (no checks, tag gets
// ignored) and then perform a read or write access with `Shr(None)` as
// the tag, which is already the top of the stack so nothing changes.
\*y1 = 3;
\*y2 = 5;
\*y2 = \*y1;
}

设y2=y1；unSafe{//所有这些都首先取消引用原始指针(没有检查，//忽略标记)，然后使用`Shr(None)`作为//标记执行读或写访问，该标记已经是堆栈的顶部，因此什么都不会改变。*y1=3；*y2=5；*y2=*y1；}

// Writing to `x` again pops `Shr` off the stack, as per the rules for
// write accesses.
\*x = 7;
// stack: \[Uniq(0)\]; not frozen

//写入`x`再次弹出`Shr`，按照//写访问规则*x=7；//堆栈：[uniq(0)]；未冻结

// Any further access through the raw pointers is undefined behavior, even
// reads: The write to `x` re-asserted that `x` is the unique reference for
// this memory.
let \_val = unsafe { \*y1 };
}
{% endhighlight %}

//任何通过原始指针的进一步访问都是未定义的行为，甚至//读取：对`x`的写入重新断言`x`是//该内存的唯一引用。Let_val=不安全的{*y1}；}{%endHighth%}

`Retag([raw])` acts almost like a normal retag, except that it does not ignore
raw pointers and instead tags them `Shr(None)`, and pushes `Shr` to the stack.

\`Retag([raw])`的作用与普通的retag几乎相同，不同之处在于它不会忽略原始指针，而是将它们标记为`Shr(None)`，并将`Shr`推入堆栈。

This way, even if the program casts the pointer to an integer and back (where we
cannot always keep track of the tag, so it might get reset to `Shr(None)`),
there is a matching `Shr` on the stack, making sure the raw pointer can actually
be used.  One way to think about this is to consider the reference to "escape"
when it is cast to a raw pointer, which is reflected by the `Shr` item on the
stack.

这样，即使程序将指针强制转换为一个整数(我们不能总是跟踪标记，因此它可能会被重置为`Shr(None)`)，堆栈上也会有一个匹配的‘Shr`，以确保原始指针可以实际使用。考虑这一点的一种方法是考虑在将其强制转换为原始指针时对“ESCRY”的引用，堆栈上的`Shr`项反映了这一点。

Knowing about how `Retag` interacts with raw pointers, you can now go back to
`demo2` and should be able to fully explain why the stack changes the way it
does in that example.

了解了`Retag`如何与原始指针交互后，您现在可以返回到`demo2`，并且应该能够完整地解释为什么堆栈改变了该示例中的方式。

### 3.3 The Case of the Aliasing References

### 3.3别名引用的情况

Everything I described so far was pretty much in working condition as of about a
week ago.  However, there was one thorny problem that I only discovered fairly
late, and as usual it is best demonstrated by an example -- entirely in safe
code:

到目前为止，我所描述的一切都在大约一周前的工作状态。然而，有一个棘手的问题是我很晚才发现的，像往常一样，它最好通过一个示例来演示--完全在安全代码中：

{% highlight rust %}
fn demo_refcell() {
let rc: &mut RefCell<u8> = &mut RefCell::new(23u8);
Retag(rc); // tag gets changed to `Uniq(0)`
// We will consider the stack of the location where `23` is stored; the
// `RefCell` bookkeeping counters are not of interest.
// stack: \[Uniq(0)\]

{%Highlight Rust%}fn demo_refcell(){let rc：&mut RefCell=&mut RefCell：：new(23u8)；retag(Rc)；//tag改为`Uniq(0)`//我们会考虑`23`存放位置的堆栈；//`RefCell`记账计数器不感兴趣。//堆栈：[uniq(0)]

// Taking a shared reference shares the location but does not freeze, due
// to the `UnsafeCell`.
let rc_shr: &RefCell<u8> = &\*rc;
Retag(rc_shr); // tag gets changed to `Shr(Some(1))`
// stack: \[Uniq(0), Shr\]; not frozen

//获取共享引用会共享位置，但不会冻结，//原因是`UnSafeCell`让rc_shr：&RefCell=&*rc；retag(Rc_Shr)；//标签变为`Shr(Some(1))`//堆栈：[uniq(0)，Shr]；未冻结

// Lots of stuff happens here but it does not matter for this example.
let mut bmut: RefMut<u8> = rc_shr.borrow_mut();

//这里发生了很多事情，但对于这个例子来说并不重要。让mut bmut：RefMut=rc_sh.borrow_mut()；

// Obtain a mutable reference into the `RefCell`.
let mut_ref: &mut u8 = &mut \*bmut;
Retag(mut_ref); // tag gets changed to `Uniq(2)`
// stack: \[Uniq(0), Shr, Uniq(2)\]; not frozen

//获取对`RefCell`的可变引用。让mut_ref：&mut U8=&mut*bmut；retag(Mut_Ref)；//标签变为`uniq(2)`//堆栈：[uniq(0)，shr，uniq(2)]；不冻结

// And at the same time, a fresh shared reference to its outside!
// This counts as a read access through `rc`, so we have to pop until
// at least a `Shr` is at the top of the stack.
let shr_ref: &RefCell<u8> = &\*rc; // tag gets changed to `Shr(Some(3))`
Retag(shr_ref);
// stack: \[Uniq(0), Shr\]; not frozen

//同时，对其外部的新的共享引用！//这被视为通过`rc`的读访问，所以我们必须弹出，直到//至少有一个`Shr`位于堆栈的顶部。让shr_ref：&RefCell=&*rc；//标签改为`Shr(Some(3))`retag(Shr_Ref)；//堆栈：[uniq(0)，Shr]；未冻结

// Now using `mut_ref` is UB because its tag is no longer on the stack.  But
// that is bad, because it is usable in safe code.
\*mut_ref += 19;
}
{% endhighlight %}

//现在使用`mut_ref`是UB，因为它的标签不在堆栈上。但//这很糟糕，因为它在安全代码中是可用的。*mut_ref+=19；}{%endHighlight%}

Notice how `mut_ref` and `shr_ref` alias!  And yet, creating a shared reference
to the memory already covered by our unique `mut_ref` must not invalidate
`mut_ref`.  If we follow the instructions above, when we retag `shr_ref` after
it got created, we have no choice but pop the item matching `mut_ref` off the
stack.  Ouch.

注意`mut_ref`和`shr_ref`的别名！然而，创建对我们唯一的`mut_ref`已经覆盖的内存的共享引用不能使`mut_ref`无效。如果我们按照上面的说明操作，当我们在创建`shr_ref`后重新标记它时，我们别无选择，只能将匹配`mut_ref`的项从堆栈中弹出。唉哟。

This made me realize that creating a shared reference has to be very weak inside
`UnsafeCell`.  In fact, it is entirely equivalent to `Retag([raw])`: We just
have to make sure some kind of shared access is possible, but we have to accept
that there might be active mutable references assuming exclusive access to the
same locations.  That on its own is not enough, though.

这让我意识到，在`UnSafeCell`中创建共享引用肯定非常弱。事实上，它完全等同于`Retag([RAW])`：我们只需要确保某种类型的共享访问是可能的，但我们必须接受可能存在假定独占访问相同位置的活动可变引用。不过，单靠这一点是不够的。

I also added a new check to the retagging procedure: Before taking any action
(i.e., before step 3, which could pop items off the stack), we check if the
reborrow is redundant: If the new reference we want to create is already
dereferencable (because its item is already on the stack and, if applicable, the
stack is already frozen), *and* if the item that justifies this is moreover
"derived from" the item that corresponds to the old reference, then we just do
nothing.  Here, "derived from" means "further up the stack".  Basically, the
reborrow has already happened and the new reference is ready for use; *and*
because of that "derived from" check, we know that using the new reference will
*not* pop the item corresponding to the old reference off the stack.  In that
case, we avoid popping anything, to keep other references valid.

我还在重新标记过程中添加了一个新的检查：在采取任何操作之前(即，在步骤3之前，可能会将项从堆栈中弹出)，我们检查重新借用是否是冗余的：如果我们想要创建的新引用已经是可取消引用的(因为它的项已经在堆栈上，并且如果适用，堆栈已经冻结)，并且如果证明这一点的项是而且是从对应于旧引用的项“派生”的，那么我们就不做任何事情。在这里，“派生自”的意思是“在堆栈中更靠上”。基本上，重新借用已经发生，新引用已经准备好可以使用；由于“派生自”检查，我们知道使用新引用不会将对应于旧引用的项从堆栈中弹出。在这种情况下，我们避免弹出任何内容，以保持其他引用有效。

It may seem like this rule can never apply, because how can our fresh tag match
something that's already on the stack?  This is indeed impossible for `Uniq`
tags, but for `Shr` tags, matching is more liberal.  For example, this rule
applies in our example above when we create `shr_ref` from `mut_ref`.  We do not
require freezing (because there is an `UnsafeCell`), there is already a `Shr` on
the stack (so the new reference is dereferencable) and the item matching the old
reference (`Uniq(0)`) is below that `Shr` (so after using the new reference, the
old one remains dereferencable).  Hence we do nothing, keeping the `Uniq(2)` on
the stack, such that the access through `mut_ref` at the end remains valid.

这条规则似乎永远不适用，因为我们的新标记如何与堆栈中已有的标记匹配？这对于`Uniq`标签来说确实是不可能的，但是对于`Shr`标签来说，匹配更自由。例如，在上面的示例中，当我们从`mut_ref`创建`shr_ref`时，此规则适用。我们不需要冻结(因为有一个`UnSafeCell`)，堆栈上已经有一个`Shr`(所以新的引用是可取消引用的)，并且匹配旧引用的项(`uniq(0)`)在那个`Shr`的下面(所以在使用新引用之后，旧的引用仍然是可取消引用的)。因此，我们什么都不做，将`uniq(2)`保留在堆栈上，以便通过末尾的`mut_ref`进行的访问保持有效。

This may sound like a weird rule, and it is.  I would surely not have thought of
this if `RefCell` would not force our hands here.  However, as we shall see in
[section 5](#5-key-properties), it also does not to break any of the important properties of the
model (mutable references being unique and shared references being immutable
except for `UnsafeCell`).  Moreover, when pushing an item to the stack (at the
end of the retag action), we can now be sure that the stack is not yet frozen:
if it were frozen, the reborrow would be redundant.

这可能听起来像是一个奇怪的规则，事实的确如此。如果‘RefCell’不强迫我们在这里，我肯定不会想到这一点。然而，正如我们将在第5节中看到的，它也不会破坏模型的任何重要属性(可变引用是唯一的，共享引用是不变的，但`UnSafeCell`除外)。此外，在将项目推送到堆栈时(在重新标记操作结束时)，我们现在可以确保堆栈尚未冻结：如果它被冻结，重新借用将是多余的。

With this extension, the instructions for `Retag` now look as follows (again
executed on all locations covered by the reference, according to `size_of_val`):

有了这个扩展，`Retag`的指令现在如下所示(根据`Size_of_val`，在引用覆盖的所有位置上再次执行)：

1. Compute a fresh tag: `Uniq(_)` for mutable references, `Box`, `Shr(Some(_))`
   for shared references, and `Shr(None)` for raw pointers.
1. Perform the checks that would also happen when we dereference this reference.
   Remember the position of the item matching the tag in the stack.
1. Redundancy check: If the new tag passes the checks performed on a
   dereference, and if the item that makes this check succeed is *above* the one
   we remembered in step 2 (where the "frozen" state is considered above every
   item in the stack), then stop.  We are done for this location.
1. Perform the actions that would also happen when an actual access happens
   through this reference (for shared references a read access, for mutable
   references a write access).<br>
   Now the location cannot be frozen any more: If the fresh tag is `Uniq`, we
   just unfroze; if the fresh tag is `Shr` and the location was already frozen,
   then the redundancy check (step 3) would have kicked in.
1. Check if the new tag is `Shr(Some(t))` and the location is inside an `UnsafeCell`.
   * If both conditions apply, freeze the location with timestamp `t`.  If it
     is already frozen, do nothing.
   * Otherwise, push a new item onto the stack: `Shr` if the tag is a `Shr(_)`,
     `Uniq(id)` if the tag is `Uniq(id)`.

The one thing I find slightly unsatisfying about the redundancy check is that it
seems to overlap a bit with the rule that on a *read* access, a `Shr` item
matches a `Uniq` tag.  Both of these together enable the read-only use of
mutable references that have already been shared; I would prefer to have a
single condition enabling that instead of two working together.  Still, overall
I think this is a pleasingly clean model; certainly much cleaner than what I
proposed last year and at the same time much more compatible with existing code.

计算一个新的标签：`uniq(_)`用于可变引用，`Box`，`Shr(Some(_))`用于共享引用，`Shr(None)`用于原始指针。执行当我们取消引用该引用时也会发生的检查。记住与堆栈中的标记匹配的项的位置。冗余检查：如果新标记通过了对取消引用执行的检查，并且如果使此检查成功的项高于我们在步骤2中记住的项(在步骤2中，“冻结”状态被认为高于堆栈中的每个项)，则停止。此位置的操作已完成。执行通过此引用进行实际访问时也会发生的操作(对于共享引用为读访问，对于可变引用为写访问)。现在不能再冻结位置：如果新标签是`Uniq`，我们就解冻；如果新标签是`Shr`，并且位置已经被冻结，那么冗余校验(步骤3)就开始了。检查新标签是不是`Shr(Some(T))`，位置是否在`UnSafeCell`内。如果这两个条件都满足，那么冻结带有时间戳`t`的位置。如果已经冻结，则不执行任何操作。否则，将新项推送到堆栈上：如果标记是`Shr(_)`，则推送新项；如果标记是`uniq(Id)`，则推送`uniq(Id)`。我发现冗余检查有点不满意的一点是，它似乎与在读取访问时，`Shr`项与`uniq`标记匹配的规则有些重叠。这两者一起支持以只读方式使用已经共享的可变引用；我更希望有一个单独的条件来实现这一点，而不是两个一起工作。尽管如此，总的来说，我认为这是一个令人愉快的干净模型；当然比我去年提出的要干净得多，同时也更兼容现有代码。

## 4 Differences to the Original Proposal

## 4与原提案不同之处

The key differences to the original proposal is that the check performed on a
dereference, and the check performed on an access, are not the same check.  This
means there are more "moving parts" in the model, but it also means we do not
need a weird special exception (about reads from frozen locations) for `demo1`
any more like the original proposal did.  The main reason for this change,
however, is that on an access, we just do not know if we are inside an
`UnsafeCell` or not, so we cannot do all the checks we would like to do.
Accordingly, I also rearranged terminology a bit.  There is no longer one
"reactivation" action, instead there is a "deref" check and an "access" action,
as described above in sections [2.2](#22-dereferencing-a-pointer) and [2.3](#23-accessing-memory).

与原始建议的关键区别在于，对取消引用执行的检查和对访问执行的检查不是相同的检查。这意味着模型中有更多的“移动部件”，但这也意味着我们不再需要像最初的提议那样为`demo1‘提供一个奇怪的特殊例外(关于从冻结位置读取数据)。然而，这一变化的主要原因是，在访问时，我们只是不知道我们是否在‘不安全单元’中，所以我们不能执行我们想要执行的所有检查。因此，我也稍微重新整理了一下术语。不再有一个“重新激活”操作，而是有一个“deref”检查和“访问”操作，如上文第2.2和2.3节所述。

Beyond that, I made the behavior of shared references and raw pointers more
uniform.  This helped to fix test failures around `iter_mut` on slices, which
first creates a raw reference and then a shared reference: In the original
model, creating the shared reference invalidates previously created raw
pointers.  As a result of the more uniform treatment, this no longer happens.
(Coincidentally, I did not make this change with the intention of fixing
`iter_mut`.  I did this change because I wanted to reduce the number of case
distinctions in the model.  Then I realized the relevant test suddenly passed
even with the full model enabled, investigated what happened, and realized I
accidentally had had a great idea. :D )

除此之外，我还使共享引用和原始指针的行为更加统一。这有助于修复切片上关于`iter_mu`的测试失败，它首先创建一个原始引用，然后创建一个共享引用：在原始模型中，创建共享引用会使先前创建的原始指针失效。由于更统一的待遇，这种情况不再发生。(巧合的是，我进行此更改并不是为了修复`iter_mu`。我进行此更改是因为我想减少模型中区分大小写的数量。然后我意识到，即使在启用了完整模型的情况下，相关测试也突然通过了，调查了发生了什么，并意识到我意外地有了一个很棒的想法。：D)

The tag is now "typed" (`Uniq` vs `Shr`) to be able to support `transmute`
between references and shared pointers.  Such `transmute` were an open question
in the original model and some people raised concerns about it in the ensuing
discussion.  I invite all of you to come up with strange things you think you
should be able to `transmute` and throw them at Miri so that we can see if your
use-cases are covered. :)

该标签现在是“类型化的”(`Uniq`vs`Shr`)，以便能够支持引用和共享指针之间的`Transmute`。这种“变形”在最初的模型中是一个悬而未决的问题，在随后的讨论中，一些人提出了对它的担忧。我邀请你们所有人想出一些奇怪的东西，你们认为你们应该能够‘变形’，并把它们扔给Miri，这样我们就可以看看你们的用例是否被覆盖了。：)

The redundancy check during retagging can be seen as refining a similar check
that the original model did whenever a new reference was created (where we
wouldn't change the state if the new borrow is already active).

重新标记期间的冗余检查可以被视为改进了原始模型在创建新引用时所做的类似检查(如果新借用已经处于活动状态，则不会更改状态)。

Finally, the notion of "function barriers" from the original Stacked Borrows has
not been implemented yet.  This is the next item on my todo list.

最后，最初的堆叠借款的“功能障碍”的概念还没有实现。这是我待办事项清单上的下一项。

## 5 Key Properties

## 5个关键属性

Let us look at the two key properties that I set out as design goals, and see
how the model guarantees that they hold true in all valid (UB-free) executions.

让我们来看看我作为设计目标列出的两个关键属性，并看看模型如何保证它们在所有有效(无UB)的执行中都是正确的。

### 5.1 Mutable References are Unique

### 5.1可变引用是唯一的

The property I would like to establish here is that: After creating (retagging,
really) a `&mut`, if we then run some unknown code *that does not get passed the
reference*, and then we use the reference again (reading or writing), we can be
sure that this unknown code did not access the memory behind our mutable
reference at all (or we have UB).  For example:

我想在这里建立的属性是：在创建(实际上是重新标记)`&mu`之后，如果我们运行一些未传递引用的未知代码，然后再次使用该引用(读取或写入)，我们可以确定该未知代码根本没有访问我们可变引用背后的内存(或者我们有UB)。例如：

{% highlight rust %}
fn demo_mut_unique(our: &mut i32) -> i32 {
Retag(our); // So we can be sure the tag is unique

{%Highlight Rust%}FN DEMO_MUT_UNIQUE(OUR：&MUT I32)->I32{retag(Our)；//这样我们就可以确保标签是唯一的

\*our = 5;

\*Our=5；

unknown_code();

未知代码()；

// We know this will return 5, and moreover if `unknown_code` does not panic
// we know we could do the write after calling `unknown_code` (because it
// cannot even read from `our`).
\*our
}
{% endhighlight %}

//我们知道这会返回5，而且如果`UNKNOWN_CODE`没有死机//我们知道可以在调用`UNKNOWN_CODE`之后进行写入(因为它//甚至不能从`our`中读取)。*我们的}{%endHighth%%}

The proof sketch goes as follows: After retagging the reference, we know it is
at the top of the stack and the location is not frozen.  (The "redundant
reborrow" rule does not apply because a fresh `Uniq` tag can never be
redundant.)  For any access performed by the unknown code, we know that access
cannot use the tag of our reference because the tags are unique and not
forgeable.  Hence if the unknown code accesses our locations, that would pop our
tag from the stack.  When we use our reference again, we know it is on the
stack, and hence has not been popped off.  Thus there cannot have been an access
from the unknown code.

验证草图如下：重新标记引用后，我们知道它位于堆栈的顶部，并且位置没有冻结。(冗余再借用规则不适用，因为新的`Uniq`标签永远不能是多余的。)对于未知代码执行的任何访问，我们知道Access不能使用我们引用的标记，因为这些标记是唯一的且不可伪造。因此，如果未知代码访问我们的位置，将从堆栈中弹出我们的标记。当我们再次使用我们的引用时，我们知道它在堆栈上，因此没有被弹出。因此，不可能有来自未知代码的访问。

Actually this theorem applies *any time* we have a reference whose tag we can be
sure has not been leaked to anyone else, and which points to locations which
have this tag at the top of the (unfrozen) stack.  This is not just the case
immediately after retagging.  We know our reference is at the top of the stack
after writing to it, so in the following example we know that `unknown_code_2`
cannot access `our`:

实际上，这个定理适用于任何时候，只要我们有一个引用，我们可以确保它的标记没有泄露给其他任何人，并且它指向将这个标记放在(未冻结的)堆栈顶部的位置。这不仅仅是重新标记后立即出现的情况。我们知道我们的引用在写入后位于堆栈的顶部，所以在下面的示例中，我们知道`UNKNOWN_CODE_2`无法访问`our`：

{% highlight rust %}
fn demo_mut_advanced_unique(our: &mut u8) -> u8 {
Retag(our); // So we can be sure the tag is unique

{%Highlight Rust%}FN DEMO_MUT_ADVANCED_UNIQUE(OUR：&MUT U8)->U8{retag(Our)；//这样我们就可以确保标签是唯一的

unknown_code_1(&\*our);

UNKNOWN_CODE_1(&*our)；

// This "re-asserts" uniqueness of the reference: After writing, we know
// our tag is at the top of the stack.
\*our = 5;

//这“重新断言”了引用的唯一性：在编写之后，我们知道//我们的标记位于堆栈的顶部。*Our=5；

unknown_code_2();

未知代码2()；

// We know this will return 5
\*our
}
{% endhighlight %}

//我们知道这将返回5*Our}{%endHighth%}

### 5.2 Shared References (without `UnsafeCell)` are Immutable

### 5.2共享引用(没有`UnSafeCell)`是不可变的

The key property of shared references is that: After creating (retagging,
really) a shared reference, if we then run some unknown code (it can even have
our reference if it wants), and then we use the reference again, we know that
the value pointed to by the reference has not been changed.  For example:

共享引用的关键属性是：在创建(实际上是重新标记)共享引用之后，如果我们运行一些未知代码(如果需要，它甚至可以拥有我们的引用)，然后我们再次使用该引用，我们知道该引用指向的值没有更改。例如：

{% highlight rust %}
fn demo_shr_frozen(our: &u8) -> u8 {
Retag(our); // So we can be sure the tag actually carries a timestamp

{%Highlight Rust%}fn demo_shr_冻结(our：&u8)->u8{retag(Our)；//这样我们就可以确定标签确实带有时间戳

// See what's in there.
let val = \*our;

//看看里面有什么。让Val=*我们的；

unknown_code(our);

未知代码(OUR)；

// We know this will return `val`
\*our
}
{% endhighlight %}

//我们知道这将返回`val`*our}{%endHighlight%}

The proof sketch goes as follows: After retagging the reference, we know the
location is frozen (this is the case even if the "redundant reborrow" rule
applies).  If the unknown code does any write, we know this will unfreeze the
location.  The location might get re-frozen, but only at the then-current
timestamp.  When we do our read after coming back from the unknown code, this
checks that the location is frozen *at least* since the timestamp given in its
tag, so if the location is unfrozen or got re-frozen by the unknown code, the
check would fail.  Thus the unknown code cannot have written to the location.

证明草图如下：在重新标记引用之后，我们知道该位置是冻结的(即使“冗余再借用”规则适用，情况也是如此)。如果未知代码执行任何写入操作，我们知道这将解冻该位置。该位置可能会重新冻结，但仅限于当时的时间戳。当我们从未知代码返回后进行读取时，这将检查位置是否至少自其标记中给出的时间戳以来被冻结，因此如果位置被未知代码解冻或重新冻结，则检查将失败。因此，未知代码不可能已写入该位置。

One interesting observation here for both of these proofs is that all we rely on
when the unknown code is executed are the actions performed on every memory
access.  The additional checks that happen when a pointer is dereferenced only
matter in *our* code, not in the foreign code.  Hence we have no problem
reasoning about the case where we call some code via FFI that is written in a
language without a notion of "dereferencing", all we care about is the actual
memory accesses performed by that foreign code.  This also indicates that we
could see the checks on pointer dereference as another "shadow state operation"
next to `Retag`, and then these two operations plus the actions on memory
accesses are all that there is to Stacked Borrows.  This is difficult to
implement in Miri because dereferences can happen any time a path is evaluated,
but it is nevertheless interesting and might be useful in a "lower-level MIR"
that does not permit dereferences in paths.

这里对这两个证明的一个有趣的观察是，当执行未知代码时，我们所依赖的是在每次内存访问时执行的操作。当指针被取消引用时发生的额外检查只在我们的代码中起作用，而不是在外来代码中。因此，如果我们通过FFI调用某些代码，而这些代码是用一种没有“取消引用”概念的语言编写的，那么我们在推理时就没有问题了，我们所关心的只是该外来代码执行的实际内存访问。这也表明，我们可以将对指针取消引用的检查视为`Retag`旁边的另一个“影子状态操作”，然后这两个操作加上对内存访问的操作就是堆叠借入的全部内容。这在MIRI中很难实现，因为在对路径求值的任何时候都可能发生取消引用，但它仍然很有趣，并且在不允许在路径中取消引用的“低级MIR”中可能很有用。

## 6 Evaluation, and How You Can Help

## 6评估，以及您可以如何提供帮助

I have implemented both the validity invariant and the model as described above
in Miri. This [uncovered](https://github.com/rust-lang/rust/issues/54908) two
[issues](https://github.com/rust-lang/rust/issues/54957) in the standard
library, but both were related to validity invariants, not Stacked Borrows.
With these exceptions, the model passes the entire test suite.  There were some
more test failures in earlier versions (as mentioned in [section 4](#4-differences-to-the-original-proposal)), but the
final model accepts all the code covered by Miri's test suite.  (If you look
close enough, you can see that three libstd methods are currently whitelisted
and what they do is not checked.  However, even before I ran into these cases,
[efforts](https://github.com/rust-lang/rust/pull/54668) were already
[underway](https://github.com/rust-lang/rfcs/pull/2582) that would fix all of
them, so I am not concerned about them.)  Moreover I wrote a bunch of
compile-fail tests to make sure the model catches various violations of the key
properties it should ensure.

我已经实现了有效性不变量和上面在MIRI中描述的模型。这揭示了标准库中的两个问题，但这两个问题都与有效性不变量有关，而不是堆叠借阅。除了这些例外，模型通过了整个测试套件。在早期版本中有更多的测试失败(如第4节所述)，但是最终的模型接受Miri的测试套件所涵盖的所有代码。(如果仔细观察，您可以看到三个libstd方法当前被列入白名单，并且它们所做的工作没有被选中。然而，即使在我遇到这些情况之前，解决所有这些问题的努力已经在进行中，所以我并不担心它们。)此外，我编写了一系列编译失败测试，以确保模型捕捉到它应该确保的关键属性的各种违规。

The most interesting change I had to make to libstd is
[in `NonNull::from`](https://github.com/rust-lang/rust/pull/56161).  That
function turned a `&mut T` into a `*const T` going through a `&T`.  This means
that the final raw pointer was created from a shared reference, and hence must
not be used for mutation.  An earlier version of this post described a model
that would permit such behavior, but I think we should actually at least
experiment with ruling it out: "no mutation through (pointers derived from)
shared references" is an old rule in Rust, after all.

我不得不对libstd进行的最有趣的更改是在`NonNull：：From`中。该函数通过`&T`将`&mut T`转换为`*const T`。这意味着最终的原始指针是从共享引用创建的，因此不能用于突变。这篇文章的一个早期版本描述了一个允许这种行为的模型，但我认为我们实际上应该尝试排除它：毕竟，在Rust中，没有通过共享引用(来自共享引用的指针)进行突变是一条古老的规则。

Overall, I am quite happy with this!  I was expecting much more trouble, expecting to run
into cases where libstd does strange things that are common or otherwise hard to
declare illegal and that my model could not reasonably allow.  I see the test
suite passing as an indication that this model may be well-suited for Rust.

总体而言，我对此相当满意！我预计会有更多的麻烦，预计会遇到libstd做一些常见的或很难宣布为非法的奇怪事情的情况，而我的模型无法合理地允许这些事情。我认为测试套件的通过表明该模型可能非常适合Rust。

However, Miri's test suite is tiny, and I have but one brain to come up with
counterexamples!  In fact I am quite a bit worried because I literally came up
with `demo_refcell` less than two weeks ago, so what else might I have missed?
This where you come in.  Please test this model!  Come up with something funny
you think should work (I am thinking about funny `transmute` in particular,
using type punning through unions or raw pointers if you prefer that), or maybe
you have some crate that has some unsafe code and a test suite (you do have a
test suite, right?) that might run under Miri.

然而，Miri的测试套件很小，我只有一个大脑来想出反例！事实上，我真的很担心，因为不到两周前，我就想出了一个‘demo_refcell’，所以我可能还错过了什么？这是你的用武之地。请测试一下这个型号！想出一些你认为应该有用的有趣的东西(我特别在想有趣的‘变形’，如果你喜欢的话，可以通过联合或原始指针使用类型双关)，或者你可能有一些包含一些不安全代码和测试套件的箱子(你有一个测试套件，对吗？)这可能会在米里的领导下运行。

The easiest way to try the model is the
[playground](https://play.rust-lang.org/): Type the code, select "Tools - Miri",
and you'll see what it does.

尝试该模型的最简单方法是操场：输入代码，选择“Tools-Miri”，您将看到它的功能。

For things that are too long for the playground, you have to install Miri on
your own computer.  Miri depends on rustc nightly and has to be updated
regularly to keep working, so it is not well-suited for crates.io.  Instead,
installation instructions for Miri are provided
[in the README](https://github.com/solson/miri/#running-miri).  We are still
working on making installing Miri easier.  Please let me know if you are having
trouble with anything.  You can report issues, comment on this post or find me
in chat (as of recently, I am partial to Zulip where we have an
[unsafe code guidelines stream](https://rust-lang.zulipchat.com/#narrow/stream/136281-wg-unsafe-code-guidelines)).

对于操场上太长的东西，你必须在自己的电脑上安装Miri。Miri每晚都依赖rustc，必须定期更新才能继续工作，因此它不太适合crates.io。相反，自述文件中提供了MIRI的安装说明。我们仍在努力让安装Miri变得更容易。如果你有什么问题，请告诉我。你可以报告问题，评论这篇文章，或者在聊天中找到我(最近，我偏爱Zulip，那里有一个不安全的代码指南流)。

With Miri installed, you can `cargo miri` a project with a binary to run it in
Miri.  Dependencies should be fully supported, so you can use any crate you
like.  It is not unlikely, however, that you will run into issues because Miri
does not support some operation.  In that case please search the
[issue tracker](https://github.com/solson/miri/issues) and report the issue if
it is new.  We cannot support everything, but we might be able to do something
for your case.

安装了Miri之后，你就可以用一个二进制文件来加载一个在Miri中运行的项目。应该完全支持依赖关系，所以您可以使用任何您喜欢的板条箱。然而，由于MIRI不支持某些操作，您不太可能遇到问题。在这种情况下，请搜索问题跟踪器并报告问题，如果是新的问题。我们不能支持所有事情，但我们也许能为您的情况做些什么。

Unfortunately, `cargo miri test` is currently broken; if you want to help with
that [here are some details](https://github.com/solson/miri/issues/479).
Moreover, wouldn't it be nice if we could
[run the entire libcore, liballoc and libstd test suite in miri](https://github.com/rust-lang/rust/issues/54914)?
There are tons of interesting cases of Rust's core data structures being
exercise there, and the comparatively tiny Miri test suite has already helped to
find two soundness bugs, so there are probably more.  Once `cargo miri test`
works again, it would be great to find a way to run it on the standard library
test suites, and set up something so that this happens automatically on a
regular basis (so that we notice regressions).
**Update:** `cargo miri test` has been fixed in the mean time, so you can use it on your libraries now! **/Update**

不幸的是，‘Cargo Miri测试’目前出现故障；如果你想帮助解决这个问题，这里有一些详细信息。此外，如果我们可以在MILI中运行整个libcore、liballoc和libstd测试套件，那不是很好吗？那里有大量有趣的Rust核心数据结构测试案例，相对较小的MIRI测试套件已经帮助找到了两个可靠的错误，所以可能还有更多错误。一旦‘Cargo Miri test’再次工作，最好能找到一种方法在标准库测试套件上运行它，并设置一些东西，以便定期自动执行(这样我们就可以注意到回归)。更新：`Cargo Miri Test`已经修复，所以你现在可以在你的库上使用它了！/UPDATE

As you can see, there is more than enough work for everyone.  Don't be shy!  I
have a mere two weeks left on this internship, after which I will have to
significantly reduce my Rust activities in favor of finishing my PhD.  I won't
disappear entirely though, don't worry -- I will still be able to mentor you if
you want to help with any of the above tasks. :)

如你所见，每个人都有足够的工作要做。别害羞！我的实习只剩下两周了，之后我将不得不大幅减少生锈活动，以利于完成我的博士学位。不过，我不会完全消失，别担心--如果你想帮助你完成上面的任何任务，我仍然可以指导你。：)

Thanks to @nikomatsakis for feedback on a draft of this post, to @shepmaster for
making Miri available on the playground, and to @oli-obk for reviewing all my
PRs at unparalleled speed. \<3

感谢@nikomatsakis对这篇帖子草稿的反馈，感谢@shepmaster让Miri可以在操场上使用，感谢@oli-obk以无与伦比的速度审查了我的所有PR。<3

If you want to
help or report results of your experiments, if you have any questions or
comments, please join the
[discussion in the forums](https://internals.rust-lang.org/t/stacked-borrows-implemented/8847).

如果你想帮助或报告你的实验结果，如果你有任何问题或建议，请加入论坛的讨论。

## Changelog

## 更改日志

**2018-11-21:** Dereferencing a pointer now always preserves the tag, but
casting to a raw pointer resets the tag to `Shr(None)`.  `Box` is treated like a
mutable reference.

2018-11-21：取消引用指针现在始终保留标记，但强制转换为原始指针会将标记重置为`Shr(None)`。`Box`被视为可变引用。

**2018-12-22:** Creating a shared reference does not push a `Shr` item to the
stack (unless there is an `UnsafeCell`).  Moreover, creating a raw pointer is a
special kind of retagging.

2018-12-22：创建共享引用不会将`Shr`项推送到堆栈(除非有`UnSafeCell`)。此外，创建原始指针是一种特殊的重新标记。