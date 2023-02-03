---

## title: "Stacked Borrows: An Aliasing Model For Rust"
categories: internship rust
forum: https://internals.rust-lang.org/t/stacked-borrows-an-aliasing-model-for-rust/8153

## 标题：《堆叠的借款：生锈的别名模型》类别：实习生锈论坛：https://internals.rust-lang.org/t/stacked-borrows-an-aliasing-model-for-rust/8153

In this post, I am proposing "Stacked Borrows": A set of rules defining which kinds of aliasing are allowed in Rust.
This is intended to answer the question which pointer may be used when to perform which kinds of memory accesses.

在这篇文章中，我提出了“堆叠借入”：一套规则，定义了在Rust中允许哪些类型的别名。这是为了回答何时可以使用哪个指针来执行哪种类型的存储器访问的问题。

This is a long-standing open question of many unsafe code authors, and also by compiler authors who want to add more optimizations.
The model I am proposing here is by far not the first attempt at giving a definition: The model is heavily based on ideas by [@arielb1](https://github.com/nikomatsakis/rust-memory-model/issues/26) and [@ubsan](https://github.com/nikomatsakis/rust-memory-model/issues/28), and of course taking into account the lessons I \[learned last year\]({% post_url 2017-08-11-types-as-contracts-evaluation %}) when I took my first stab at defining such a model, dubbed \["Types as Contracts"\]({% post_url 2017-07-17-types-as-contracts %}).

这是许多不安全代码作者以及想要添加更多优化的编译器作者长期未解决的问题。到目前为止，我在这里提出的模型并不是第一次尝试给出定义：该模型在很大程度上基于@arielb1和@ubsan的想法，当然还考虑到了我去年[学到]的经验教训({%post_url 2017-08-11-Types-as-Constraints%})，当时我第一次尝试定义这样一个模型，称为[“Types as-Contracts”]({%post_url 2017-07-17-Types-as-Contracts%})。

<!-- MORE -->

But before I delve into my latest proposal, I want to briefly discuss a key difference between my previous model and this one:
"Types as Contracts" was a fully "validity"-based model, while "Stacked Borrows" is (to some extent) "access"-based.

但在我深入研究我的最新提议之前，我想简要讨论一下我以前的模型与这个模型之间的一个关键区别：“类型作为合同”是一个完全基于“有效性”的模型，而“堆叠借款”是(在某种程度上)基于“访问”的模型。

**Update:**
Since publishing this post, I have written \[another\]({% post_url 2018-11-16-stacked-borrows-implementation %}) blog post about a slightly adjusted version of Stacked Borrows (the first version that actually got implemented).
That other post is self-contained, so if you are just interested in the current state of Stacked Borrows, I suggest you go there.
Only go on reading here if you want some additional historic context.
**/Update**

更新：自从发布这篇文章以来，我已经写了[另一个]({%post_url 2018-11-16-STACKED-LORROWS-Implementation%})博客文章，内容是关于Stack Borrow的一个略微调整的版本(第一个实际实现的版本)。另一个帖子是独立的，所以如果你只对堆叠借阅的现状感兴趣，我建议你去那里。如果你想了解更多的历史背景，请继续阅读。/更新

## 1 Validity-based vs. Access-based

## 1基于有效性与基于访问权限

An "access"-based model is one where certain properties -- in this case, mutable references being unique and shared references pointing to read-only memory -- are only enforced when the reference is actually used to *access* memory.
In contrast, a "validity"-based model requires these properties to always hold for all references that *could* be used.
In both cases, violating a property that the model requires to hold is undefined behavior..

基于“访问”的模型是这样一种模型，其中某些属性--在本例中，可变引用是唯一的，并且共享引用指向只读内存--只有在引用实际用于访问内存时才被强制执行。相反，基于“有效性”的模型要求这些属性始终适用于所有可能使用的引用。在这两种情况下，违反模型要求持有的属性都是未定义的行为。

Essentially, with a validity-based model like "Types as Contracts", the basic idea is that all data is always valid according to the type it is given.
Enforcing the restrictions of such a model (e.g., when checking whether a program has undefined behavior) amounts to eagerly checking all reachable data for validity.
An access-based model, on the other hand, only requires data to be valid when used.
Enforcing it amounts to lazily checking the bare minimum at each operation.

从本质上讲，使用类似于“类型作为契约”的基于有效性的模型，其基本思想是所有数据根据其给定的类型总是有效的。强制这样的模型的限制(例如，当检查程序是否具有未定义的行为时)相当于急切地检查所有可到达的数据的有效性。另一方面，基于访问的模型只要求数据在使用时有效。执行它相当于懒惰地检查每一次操作的最低限度。

Validity-based models have several advantages: Eager checking means we can typically identify which code is actually responsible for producing the "bad" value.
"All data must always be valid" is also easier to explain than a long list of operations and the kind of restrictions they place upon the data.

基于有效性的模型有几个优点：急切的检查意味着我们通常可以确定哪些代码实际上负责产生“坏”值。“所有数据必须始终有效”也比一长串操作及其对数据施加的限制更容易解释。

However, in Rust, we cannot talk about references and whether the are valid at their given type without talking about lifetimes.
With "Types as Contracts", the exact place where a lifetime ended turned out to be really important.
Not only did this make the specification complex and hard to understand; the implementation in Miri also had to actively work against the compiler's general intention to forget about lifetimes as early as possible.
With non-lexical lifetimes, the "end" of a lifetime is not even so clearly defined any more.

然而，在Rust中，我们不能在不讨论生存期的情况下讨论引用以及在给定类型下是否有效。在“以类型为合同”的情况下，生命的确切终点变得非常重要。这不仅使规范变得复杂和难以理解，而且Miri中的实现还必须积极地与编译器尽早忘记生存期的一般意图背道而驰。对于非词汇生命周期，生命的“结束”甚至不再是如此明确的定义。

## 2 Stacking Borrows

## 2个堆叠借阅

For these reasons, my second proposal makes lifetimes in general and the result of lifetime inference in particular completely irrelevant for whether a program has undefined behavior (UB).
This is one of the core design goals.

出于这些原因，我的第二个建议使生命周期，特别是生命周期推断的结果，与程序是否具有未定义的行为(UB)完全无关。这是设计的核心目标之一。

If you need some more context on undefined behavior and how it relates to compiler optimizations, I suggest you read \[my blog post on this topic\]({% post_url 2017-07-14-undefined-behavior %}) first.
It's not a long post, and I cannot repeat everything again here. :)

如果你需要更多关于未定义的行为以及它与编译器优化的关系的上下文，我建议你先阅读[我关于这个主题的博客文章]({%post_url 2017-07-14-unfined-behavior%})。这不是一个很长的帖子，我不能在这里重复所有的事情。：)

The central idea of this model (and its precursors by @arielb1 and @ubsan) is that, for every location, we keep track of the references that are allowed to access this location.
(I will discuss later just how we keep track of this; for now, let's just assume it can be done.)
This forms a stack: When we have an `&mut i32`, we can *reborrow* it to obtain a new reference.
That new reference is now the one that must be used for this location, but the old reference it was created from cannot be forgotten: At some point, the reborrow will expire and the old reference will be "active" again.
We will have other items on that stack as well, so we will write `Uniq(x)` to indicate that `x` is the unique reference permitted to access this location.

这个模型(以及它的前身@arielb1和@ubsan)的核心思想是，对于每个位置，我们跟踪允许访问该位置的引用。(稍后我将讨论如何跟踪这一点；现在，让我们假设它是可以做到的。)这形成了一个堆栈：当我们有一个`&mut i32`时，我们可以重新借用它来获得新的引用。该新引用现在是必须用于该位置的引用，但创建该引用的旧引用不能被忘记：在某一时刻，重新借用将到期，并且旧引用将再次“激活”。我们在该堆栈上还会有其他项，因此我们将写入`uniq(X)`，以指示`x`是允许访问该位置的唯一引用。

Let us look at an example:
{% highlight rust %}
fn demo0(x: &mut i32) -> i32 {
// At the beginning of the function, `x` must be the "active" reference
// for the 4 locations it points to, meaning `Uniq(x)` is at the top of the stack.
// (It's 4 locations because `i32` has size 4.)
let y = &mut \*x; // Now `Uniq(y)` is pushed onto the stack, as new active reference.
// The stack now contains: Uniq(y), Uniq(x), ...
\*y = 5; // Okay because `y` is active.
\*x = 3; // This "reactivates" `x` by popping the stack.
// The stack now contains: Uniq(x), ...
\*y // This is UB! `Uniq(y)` is not on the stack of borrows, so `y` must not be used.
}
{% endhighlight %}
Of course, this example would not compile because the borrow checker would complain.
However, in my interpretation, the *reason* it complains is that if it accepted the program, we would have UB in safe code!

让我们来看一个例子：{%Highlight Rust%}FN demo0(x：&mut I32)->I32{//在函数的开头，`x`必须是它所指向的4个位置的“活动”引用//，这意味着`uniq(X)`位于堆栈的顶部。//(因为`i32`的大小是4，所以是4个位置。)让y=&mut*x；//现在将`uniq(Y)`作为新的活动引用推送到堆栈上。//堆栈现在包含：uniq(Y)，uniq(X)，...*y=5；//因为`y`是活动的。*x=3；//通过弹出堆栈重新激活`x`。//堆栈现在包含：uniq(X)，...*y//This is ub！`uniq(Y)`不在借阅堆栈上，所以不能使用`y`。}{%endHighlight%}当然，此示例不会编译，因为借用检查器会抱怨。然而，在我的解释中，它抱怨的原因是，如果它接受了这个程序，我们就会有UB在安全代码中！

This is worth pondering a bit: The model defines program semantics without taking lifetimes into account, so we can run programs and ask whether they have UB without
ever doing lifetime inference or borrow checking (very much unlike "Types as Contracts").
One important property, then, is that *if* the program has UB and does not use any unsafe code, the borrow checker must detect this.
In some sense, my model defines a dynamic version of the borrow checker *that works without lifetimes*.
It turns out that even with non-lexical lifetimes, the borrow structure for a given location is still well-nested, which is why we can arrange borrows in a stack.

这一点值得深思：该模型定义了不考虑生存期的程序语义，因此我们可以运行程序并询问它们是否具有UB，而无需进行生存期推断或借用检查(与“类型作为契约”非常不同)。因此，一个重要的特性是，如果程序有UB，并且没有使用任何不安全的代码，借入检查器必须检测到这一点。在某种意义上，我的模型定义了一个动态版本的借入检查器，它可以在不使用寿命的情况下工作。事实证明，即使在非词法生存期内，给定位置的借用结构仍然是嵌套良好的，这就是为什么我们可以将借用安排在堆栈中。

### 2.1 Raw Pointers

### 2.1原始指针

Let us bypass the borrow checker by adding some unsafe code to our program:
{% highlight rust %}
fn demo1(x: &mut i32) -> i32 {
// At the beginning of the function, `x` must be the "active" reference.
let raw = x as \*mut \_; // Create raw pointer
// The stack now contains: Raw, Uniq(x), ...
let y = unsafe { &mut \*raw }; // Now `y` is pushed onto the stack, as new active reference.
// The stack now contains: Uniq(y), Raw, Uniq(x), ...
\*y = 5; // Okay because `y` is active.
\*x = 3; // This "reactivates" `x` by popping the stack twice.
\*y // This is UB! `Uniq(y)` is not on the stack of borrows, so `y` must not be used.
}
{% endhighlight %}

让我们通过在我们的程序中添加一些不安全的代码来绕过借用检查器：{%Highlight Rust%}FN demo1(x：&mut i32)->I32{//在函数的开头，`x`必须是“活动”引用。让RAW=x as*mut_；//创建RAW指针//堆栈现在包含：RAW，uniq(X)，...设y=unsafe{&mut*raw}；//现在将`y`作为新的活动引用推送到堆栈上。//堆栈现在包含：uniq(Y)，Raw，uniq(X)，...*y=5；//因为`y`是活动的。*x=3；//通过弹出堆栈两次，重新激活`x`。*y//这是UB！`uniq(Y)`不在借阅堆栈上，所以不能使用`y`。}{%结束高亮显示%}

What happens here is that we are casting `x` to a raw pointer.
For raw pointers, we cannot really keep track of where and how they have been created -- raw pointers can be safely cast to and from integers, and data could flow arbitrarily.
So, when a `&mut` is cast to `*mut` like above, we instead push `Raw` onto the stack, indicating that *any* raw pointer may be used to access this location.
(The usual restrictions about address arithmetic across allocations still apply, I am just talking about the borrow checking here.)

这里发生的情况是，我们将`x‘转换为原始指针。对于原始指针，我们无法真正跟踪它们是在哪里以及如何创建的--原始指针可以安全地与整数相互转换，数据可以任意流动。因此，当`&mu`被强制转换为`*mu`时，我们改为将`Raw`推送到堆栈上，这表明任何原始指针都可以用来访问该位置。(关于跨分配的地址计算的常见限制仍然适用，我在这里只是在谈论借用检查。)

In the next line, we use a raw pointer to create `y`.
That is okay because `Raw` is active.
As usual when a reference is created, we push it onto the stack.
This makes `y` the active reference, so we can use it in the next line.
And again, using `x` pops the stack until `x` is active -- in this case, this removes both the `Uniq(y)` and the `Raw`, making `y` unusable and causing UB in the last line.

在下一行中，我们使用原始指针来创建`y‘。这没有关系，因为`Raw`是活动的。像往常一样，当创建引用时，我们将其压入堆栈。这使‘y’成为活动引用，因此我们可以在下一行中使用它。同样，使用`x`弹出堆栈，直到`x`处于活动状态--在本例中，这将同时删除`uniq(Y)`和`raw`，使`y‘不可用，并导致最后一行中的ub。

Let us look at another example involving raw pointers:
{% highlight rust %}
fn demo2(x: &mut i32) -> i32 {
// At the beginning of the function, `x` must be the "active" reference.
let raw = x as \*mut \_; // Create raw pointer
// The stack now contains: Raw, Uniq(x), ...
let y = unsafe { &mut \*raw }; // Now `y` is pushed onto the stack, as new active reference.
// The stack now contains: Uniq(y), Raw, Uniq(x), ...
\*y = 5; // Okay because `y` is active.
unsafe { \*raw = 5 }; // Using a raw pointer, so `Raw` gets reactivated by popping the stack!
// The stack now contains: Raw, Uniq(x), ...
\*y // This is UB! `Uniq(y)` is not on the stack of borrows, so `y` must not be used.
}
{% endhighlight %}
Because raw pointers are tracked on the stack, they have to follow the well-nested structure.
`y` was "created from" `raw`, so using `raw` again invalidates `y`!
This is exactly in symmetry with the first example where `y` was "created from" `x`, so using `x` again invalidated `y`.

让我们看另一个涉及原始指针的例子：{%Highlight Rust%}FN demo2(x：&mut i32)->i32{//在函数的开头，`x`必须是“活动”引用。让RAW=x as*mut_；//创建RAW指针//堆栈现在包含：RAW，uniq(X)，...设y=unsafe{&mut*raw}；//现在将`y`作为新的活动引用推送到堆栈上。//堆栈现在包含：uniq(Y)，Raw，uniq(X)，...*y=5；//因为`y`是活动的。不安全的{*RAW=5}；//使用原始指针，所以`Raw`被弹出堆栈重新激活！//堆栈现在包含：RAW，uniq(X)，...*y//This is ub！`uniq(Y)`不在借阅堆栈上，所以不能使用`y`。}{%endHighth%%}因为原始指针是在堆栈上跟踪的，所以它们必须遵循良好的嵌套结构。`y`是从`raw`创建的，所以再次使用`raw`会使`y`无效！这与第一个例子完全一致，在第一个例子中，`y`是从`x`创建的，所以使用`x`再次使`y`无效。

### 2.2 Shared References

### 2.2共享引用

For shared references, of course, we do not have a single reference which is the only one with permission to access.
The key property we have to model is that shared references point to memory that does not change (assuming no interior mutability is involved).
The memory is, so to speak, *frozen*.

当然，对于共享引用，我们没有唯一有权访问的引用。我们必须建模的关键属性是共享引用指向不变的内存(假设不涉及内部可变性)。可以说，记忆是冻结的。

For this purpose, we tag shared references with some kind of "timestamp" indicating *when* it was created.
We also have an extra flag for each location storing *since when* the location is frozen.
Using a shared reference to access memory is okay if memory has been frozen continuously since the reference was created.

为此，我们使用某种“时间戳”来标记共享引用，以指示它是在何时创建的。我们也有一个额外的标志，每个位置存储以来，该位置被冻结。如果自创建共享引用以来内存已连续冻结，则可以使用共享引用访问内存。

We can see this in action in the following example:
{% highlight rust %}
fn demo3(x: &mut i32) -> i32 {
// At the beginning of the function, `x` must be the "active" reference.
let raw = x as \*mut \_; // Create raw pointer
// The stack now contains: Raw, Uniq(x), ...
let y = unsafe { & \*raw }; // Now memory gets frozen (recording the timestamp)
let \_val = \*y; // Okay because memory was frozen since `y` was created
\*x = 3; // This "reactivates" `x` by unfreezing and popping the stack.
let z = &\*x; // Now memory gets frozen *again*
\*y // This is UB! Memory has been frozen strictly after `y` got created.
}
{% endhighlight %}

我们可以在下面的示例中看到这一点：{%Highlight Rust%}FN demo3(x：&mut I32)->I32{//在函数的开头，`x`必须是“active”引用。让RAW=x as*mut_；//创建RAW指针//堆栈现在包含：RAW，uniq(X)，...设y=unSafe{&*RAW}；//现在内存冻结(记录时间戳)let_val=*y；//好的，因为`y`创建后内存就被冻结了*x=3；//这会通过解冻和弹出堆栈来重新激活`x`。让z=&*x；//现在内存再次冻结*y//这是ub！创建`y‘后，内存被严格冻结。}{%结束高亮显示%}

Shared references with interior mutability do not really have any restrictions in terms of what can happen to memory, so we treat them basically like raw pointers.

具有内部可变性的共享引用在内存方面实际上没有任何限制，所以我们基本上将它们视为原始指针。

### 2.3 Recap

### 2.3小结

For every location in memory, we keep track of a stack of borrows (`Uniq(_)` or `Raw`), and potentially "top off" this stack by freezing the location.
A frozen location is never written to, and no `Uniq` is pushed.

对于内存中的每个位置，我们跟踪一个借用堆栈(`uniq(_)`或`Raw`)，并可能通过冻结该位置来“封顶”该堆栈。冻结位置不会写入，也不会推送`Uniq`。

Whenever a mutable reference is created, a matching `Uniq` is pushed onto the stack for every location "covered by" the reference -- i.e., the locations that would be accessed when the reference is used (starting at where it points to, and going on for `size_of_val` many bytes).
Whenever a shared reference is created, if there is no interior mutability, we freeze the locations if they are not already frozen.
If there is interior mutability, we just push a `Raw`.
Whenever a raw pointer is created from a mutable reference, we push a `Raw`.
(Nothing happens when a raw pointer is created from a shared reference.)

每当创建可变引用时，匹配的`Uniq`就会被推送到堆栈上，用于该引用“覆盖”的每个位置--即使用该引用时将被访问的位置(从它指向的位置开始，然后是`SIZE_OF_VALL‘很多字节)。每当创建共享引用时，如果没有内部可变性，我们将冻结位置(如果它们尚未冻结)。如果存在内部易变性，我们只需推送一个“Raw`”。每当从可变引用创建原始指针时，我们都会推送一个`Raw`。(从共享引用创建原始指针时不会发生任何操作。)

A mutable reference `x` is "active" for a location if that location is not frozen and `Uniq(x)` is on top of the stack.
A shared reference without interior mutability is active if the location is frozen at least since the location was created.
A shared reference with interior mutability is active is `Raw` is on top of the stack.

如果一个位置没有被冻结并且`uniq(X)`位于堆栈的顶部，则可变引用`x`对于该位置是“活动的”。如果位置至少自位置创建以来已冻结，则不具有内部易变性的共享参照处于活动状态。具有内部可变性的共享引用处于活动状态，即`Raw`位于堆栈顶部。

Whenever a reference is used to do something (anything), we make sure that it is active again for all locations that it covers; this can involve unfreezing and popping the stack.
If it is not possible to reactivate the reference this way, we have UB.

每当引用被用来做一些事情(任何事情)时，我们确保它对于它覆盖的所有位置都是活动的；这可能涉及解冻和弹出堆栈。如果不能以这种方式重新激活参考，我们有UB。

**Update:** I previously talked about "activating" the reference instead of "reactivating it"; I decided to change terminology to make it clearer that this is an old reference being used again, so it must be on the stack already (but might not be at the top). **/Update**

更新：我之前谈到了“激活”引用，而不是“重新激活它”；我决定更改术语，以便更清楚地表明这是再次使用的旧引用，因此它一定已经在堆栈上(但可能不在顶部)。/更新

## 3 Tracking Borrows

## 3跟踪借入

So far, I have just been assuming that we can somehow keep a connection between a reference like `x` in the code above, and an item `Uniq(x)` on the stack.
I also said we are keeping track of when a shared reference was created.
To realize this, we need to somehow have information "tagged" to the reference.
In particular, notice that `x` and `y` in the first example have the same address.
If we compared them as raw pointers, they would turn out equal.
And yet, it makes a huge difference if we use `x` or `y`!

到目前为止，我只是假设我们可以以某种方式在上面代码中的`x`引用和堆栈上的项`uniq(X)`之间保持连接。我还说过，我们正在跟踪共享引用是何时创建的。要实现这一点，我们需要以某种方式将信息“标记”到引用。特别要注意，第一个示例中的`x`和`y`具有相同的地址。如果我们将它们作为原始指针进行比较，它们将被证明是平等的。然而，如果我们使用`x‘或`y’，就会产生巨大的差异！

If you read my previous post on \[why pointers are complicated\]({% post_url 2018-07-24-pointers-and-bytes %}), this should not come as too much of a surprise.
There is more to a pointer, or a reference (I am using these terms mostly interchangeably), than the address in memory that it points to.

如果你读过我上一篇关于[为什么指针很复杂]({%post_url 2018-07-24-POINTERS-AND-BYTES%})的文章，这应该不会太令人惊讶。除了指向内存中的地址之外，指针或引用(我主要是互换地使用这些术语)还有更多的含义。

For the purpose of this model, we assume that a value of reference type consists of two parts: An address in memory, and a tag used to store the time when the reference was created.
"Time" here is a rather abstract notion, we really just need some counter that we bump up every time a new reference is created.
This gives us a unique ID for each mutable reference -- and, as we have seen, for shared references we actually exploit the fact that IDs are handed out in increasing order
(so that we can test if a reference was created before or after a location was frozen).
So, we can actually treat mutable and shard references uniformly in that both just record, in their tag, the time at which they were created.

出于此模型的目的，我们假设引用类型的值由两部分组成：内存中的地址和用于存储创建引用的时间的标记。“时间”在这里是一个相当抽象的概念，我们真的只需要一些计数器，我们每次创建新的引用时都会遇到这个计数器。这为每个可变引用提供了唯一的ID--正如我们已经看到的，对于共享引用，我们实际上利用了ID是按递增顺序分发的这一事实(这样我们就可以测试引用是在位置冻结之前还是之后创建的)。因此，我们实际上可以统一处理可变引用和碎片引用，因为两者都只在标记中记录创建它们的时间。

Whenever I said above that we have `Uniq(x)` on the stack, what I really meant is that we have `Uniq(t_x)` on the stack, where `t_x` is some clock value, and that the "tag" of `x` is `t_x`.
For the sake of readability, I will continue to use the `Uniq(x)` notation below.

每当我在上面提到我们在堆栈上有`uniq(X)`时，我真正的意思是我们在堆栈上有`uniq(T_X)`，其中`t_x`是某个时钟值，而`x`的“标签”是`t_x`。为了便于阅读，我将继续使用下面的`uniq(X)‘表示法。

Since raw pointers are not tracked, we can erase the tag when casting a reference to a raw pointer.
This means our tag does not interfere with pointer-integer casts, which means there are a whole bunch of complicated questions we do not have to worry about. :)

由于未跟踪原始指针，因此我们可以在转换对原始指针的引用时擦除标记。这意味着我们的标记不会干扰指针整型转换，这意味着我们不必担心一大堆复杂的问题。：)

Of course, these tags do not exist on real hardware.
But that is besides the point.
When *specifying* program behavior, we can work with an \["instrumented machine"\]({% post_url 2017-06-06-MIR-semantics %}) that has extra state which is not present on the real machine, as long as we only use that extra state to define whether a program is UB or not:
On real hardware, we can ignore programs that are UB (they may just do whatever), so the extra state does not matter.

当然，这些标签并不存在于真正的硬件上。但这不是重点。当指定程序行为时，我们可以使用具有真实机器上不存在的额外状态的[“仪表化机器”]({%post_url 2017-06-06-MIR-Semantics%})，只要我们只使用该额外状态来定义程序是否是UB的：在真实的硬件上，我们可以忽略UB的程序(它们可以做任何事情)，所以额外的状态并不重要。

Tags are something I wanted to avoid in "Types as Contracts" -- that was one of the initial design constraints I had put upon myself, in the hope of avoiding the trouble coming with "complicated pointers".
However, I now came to the conclusion that tagging pointers is a price worth paying if it means we can make lifetimes irrelevant.

标记是我在“类型作为契约”中想要避免的东西--这是我最初给自己设置的设计约束之一，希望避免“复杂指针”带来的麻烦。然而，我现在得出的结论是，如果标记指针意味着我们可以让生命变得无关紧要，那么标记指针是值得付出的代价。

## 4 Retagging and Barriers

## 4重新标记和障碍

I hope you now have a clear idea of the basic structure of the model I am proposing: The stack of borrows, the freeze flag, and references tagged with the time at which they got created.
The full model is not quite as simple, but it is not much more complicated either.
We need to add just two more concepts: Retagging and barriers.

我希望您现在对我提出的模型的基本结构有一个清晰的了解：借入堆栈、冻结标志和标记了它们创建时间的引用。完整的模型没有那么简单，但也不会复杂得多。我们只需要再添加两个概念：重新标记和壁垒。

### 4.1 Retagging

### 4.1重新加注标签

Remember that every time we create a mutable borrow, we assign it the current
clock values as its tag.  Since the tag can never be changed, this means two
different variables can never have the same tag -- right?  Well, unfortunately,
things are not so simple: Using
e.g. [`transmute_copy`](https://doc.rust-lang.org/stable/std/mem/fn.transmute_copy.html)
or a `union`, one can make a copy of a reference in a way that Rust does not
even notice.

请记住，每次创建可变借入时，都会将当前时钟值作为其标记分配给它。由于标记永远不能更改，这意味着两个不同的变量永远不能有相同的标记--对吗？不幸的是，事情并不是那么简单：例如，使用`Transmute_Copy`或`union`，人们可以以一种Rust甚至没有注意到的方式来复制引用。

Still, we would like to make statements about code like this:
{% highlight rust %}
fn demo4(x: &mut i32, y: &mut i32) -> i32 {
\*x = 42;
\*y = 7;
\*x // Will load 42! We can optimize away the load.
}
{% endhighlight %}
The trouble is, we cannot prevent the outside world from passing bogus `&mut` that have the same tag.
Does this mean we are back to square one in terms of making aliased mutable references UB?
Lucky enough, we are not! We have a lot of machinery at our disposal, we just have to tweak it a little.

尽管如此，我们还是想对代码做出如下声明：{%Highlight Rust%}FN demo4(x：&mut I32，y：&mut I32)->I32{*x=42；*y=7；*x//将加载42！我们可以对负载进行优化。}{%endHighth%%}问题是，我们无法阻止外部世界传递具有相同标签的虚假`&mu`。这是否意味着我们在创建带别名的可变引用UB方面又回到了起点？幸运的是，我们没有！我们有很多机器可供使用，我们只需稍微调整一下即可。

What we will do is, every time a reference comes "into" our function (this can be a function argument, but also loading it from memory or getting it as the return value of some other function), we perform "retagging":
We change the tags of the mutable references to the current clock value, bumping up the clock after every tag we assign, and then we push those new tags on top of the borrow stack.
This way, we can know -- without making any assumptions about foreign code -- that all references have distinct IDs.
In particular, two different references can never be both "active" for the same location at the same time.

我们要做的是，每次引用进入我们的函数(这可以是一个函数参数，但也可以从内存中加载它或将其作为某个其他函数的返回值获取)时，我们执行“重新标记”：我们将可变引用的标记更改为当前时钟值，在我们分配的每个标记之后增加时钟，然后我们将那些新标记压入借入堆栈的顶部。这样，我们就可以知道--不需要对外来代码做任何假设--所有引用都有不同的ID。具体地说，对于同一位置，两个不同的引用永远不能同时都是“活动的”。

With this additional step, it is now easy to argue that `demo4` above is UB when `x` and `y` alias, no matter their initial tag:
After using `x`, we know it is active.
Next we use and reactivate `y`, which has to pop `Uniq(x)` as they have distinct tags.
Finally, we use `x` again even though it is no longer in the stack, triggering UB.
(A `Uniq` is only ever pushed when it is created, so it is never in the stack more than once.)

有了这一额外的步骤，当`x`和`y`作为别名时，现在很容易争论上面的`demo4`是UB，无论它们的初始标签是什么：使用`x`之后，我们知道它是活动的。接下来，我们使用并重新激活`y`，它必须弹出`uniq(X)`，因为它们有不同的标签。最后，我们再次使用`x`，即使它已不在堆栈中，也会触发UB。(`Uniq`只有在创建时才会被推送，所以它在堆栈中永远不会超过一次。)

### 4.2 Barriers

### 4.2障碍

There is one more concept I would like to add: Barriers.
The model would make a lot of sense even without barriers -- but adding barriers rules out some more behavior that I do not think we want to allow.
It is also needed to explain why we can put the [`noalias` parameter attribute](https://llvm.org/docs/LangRef.html#parameter-attributes) on our functions when generating LLVM IR.

我还想补充一个概念：障碍。即使没有障碍，这个模型也很有意义--但增加障碍排除了更多我认为我们不想允许的行为。还需要解释为什么我们可以在生成LLVM IR时将`noalias`参数属性放在我们的函数上。

Consider the following code:
{% highlight rust %}
fn demo5(x: &mut i32, y: usize) {
\*x = 42;
foo(y);
}

请考虑以下代码：{%Highlight Rust%}FN demo5(x：&mut I32，y：usize){*x=42；foo(Y)；}

fn foo(y: usize) {
let y = unsafe { &mut \*(y as \*mut i32) };
\*y = 7;
}
{% endhighlight %}
The question is: Can we reorder the `*x = 42;` down to the end of `demo5`?
Notice that we are *not* using `x` again, so we cannot assume that `x` is active at the end of `demo5`!
This is the usual trouble with access-based models.

Fn foo(y：usize){let y=unSafe{&mut*(y as*mut I32)}；*y=7；}{%endHighlight%}问题是：我们可以将`*x=42；`重新排序到`demo5`的末尾吗？注意，我们没有再次使用`x`，所以我们不能假设`demo5`末尾的`x`是活动的！这是基于访问的模型的常见问题。

However, someone might conceivably call `demo5` with `y` being `x as *mut _ as usize`, which means reordering could change program behavior.
To fix this, we have to make sure that if someone actually calls `demo5` this way, we have UB *even though* `x` is not used again.

然而，可以想象，有人可能会调用`demo5`，而`y`是`x as*mut_as usize`，这意味着重新排序可能会改变程序行为。要解决这个问题，我们必须确保如果有人以这种方式调用`demo5`，即使`x`不会被重复使用，我们也会有UB。

To this end, I propose to turn the dial a little more towards a validity-based model by imposing some extra constraints.
We want to ensure that turning the integer `y` into a reference does not pop `x` from the stack and continue executing the program (we want UB instead).
This could happen if the stack contained, somewhere, a `Raw`.
Remember that we do not tag raw pointers, so when a raw pointer was involved in creating `x`, that `Raw` item will still be on the stack, enabling any raw pointer to be used to access this location.
This is sometimes crucial, but in this case, `demo5` should be able to prevent those old historic borrows involved in creating `x` from being reactivated.

为此，我建议通过施加一些额外的限制，将刻度盘更多地转向基于有效性的模型。我们希望确保将整数`y`转换为引用不会从堆栈中弹出`x`并继续执行程序(我们希望使用UB)。如果堆栈中的某个位置包含“Raw`”，则可能会发生这种情况。请记住，我们不标记原始指针，因此当创建`x`时涉及到原始指针时，该`Raw`项仍将位于堆栈上，从而允许任何原始指针用于访问该位置。这有时是至关重要的，但在这种情况下，`demo5`应该能够防止那些涉及创建`x`的旧历史借用被重新激活。

The idea is to put a "barrier" into the stack of all function arguments when `demo5` gets called, and to make it UB to pop that barrier from the stack before `demo5` returns.
This way, all the borrows further down in the stack (below `Uniq(x)`) are temporarily disabled and cannot be reactivated while `demo5` runs.
This means that even if `y` happens to be the memory address `x` points to, it is UB to cast `y` to a reference because the `Raw` item cannot be reactivated.

其想法是在调用`demo5`时在所有函数参数的堆栈中放置一个“屏障”，并使ub在`demo5`返回之前从堆栈中弹出该屏障。这样，堆栈中更低的所有借入(`uniq(X)`以下)都被临时禁用，并且在`demo5`运行时无法重新激活。这意味着即使`y`恰好是`x`指向的内存地址，也是UB将`y`强制转换为引用，因为`Raw`项不能被重新激活。

Another way to think about barriers is as follows:
The model generally ignores lifetimes and does not know how long they last.
All we know is that when a reference is used, its lifetime must be ongoing, so we say that is when we reactivate the borrow.
On top of this, barriers encode the fact that, when a reference is passed as an argument to a function, then its lifetime (whatever it is) extends beyond the current function call.
In our example, this means that no borrow further up the stack (these are the borrows with even longer lifetimes) can be used while `demo5` is running.

另一种思考障碍的方式是：该模型通常忽略生命周期，不知道它们能持续多久。我们所知道的是，当引用被使用时，其生存期必须是持续的，所以我们说这是我们重新激活借入的时候。最重要的是，障碍编码了这样一个事实，即当引用作为参数传递给函数时，其生存期(无论它是什么)将延长到当前函数调用之外。在我们的示例中，这意味着在运行`demo5‘时，不能使用堆栈更靠上的借入(这些是寿命更长的借入)。

A nice side-effect of barriers in combination with renumbering is that even if `demo4` from the previous subsection would not use its arguments at all, it would *still* be UB to call it with two aliasing references:
When renumbering `x`, we are pushing a barrier. Renumbering `y` would attempt to reactivate `Uniq(y)`, but that can only be behind the barrier, so it cannot be reactivated.

障碍与重新编号相结合的一个很好的副作用是，即使上一小节中的`demo4‘根本不使用它的参数，UB仍然会用两个别名引用来调用它：当重新编号`x`时，我们正在推动一个障碍。重新编号`y‘会尝试重新激活`uniq(Y)`，但它只能在屏障后面，因此无法重新激活。

## 5 The Model in Code

## 5代码中的模型

Now we have everything together.
Instead of giving another recap, I will try to give an alternative, more precise description of the model in the form of pseudo Rust code.
This is essentially a draft of the code that will hopefully be in Miri soon, to actually dynamically track the borrow stack and enforce the rules.
This is also how I go about developing such models -- I use some form of pseudo-Rust, which I find it easier to be precise in than pure English.
Some details have been omitted in the high-level description so far, they should all be in this code.

现在我们所有的东西都在一起了。我将尝试以伪铁锈代码的形式给出模型的另一种更精确的描述，而不是给出另一种概括。这基本上是一个代码草案，有望很快出现在Miri中，以实际动态跟踪借入堆栈并执行规则。这也是我开发这样的模型的方式--我使用某种形式的伪铁锈，我发现用它比纯英语更容易精确。到目前为止，高级描述中已经省略了一些细节，它们应该都在这段代码中。

If you are only interested in the high-level picture, feel free to skip to the end.
The rest of this is more like a specification than an explanatory blog post.
The nice thing is that even with the spec, this post is still shorter than the one introducing "Types as Contracts". :)

如果你只对高层次的图片感兴趣，可以跳到最后。其余的内容更像是一份规范，而不是一篇解释性的博客文章。好的是，即使有了规范，这篇文章仍然比引入“类型作为合同”的那篇文章要短。：)

### 5.1 Per-Location Operations

### 5.1按地点运营

Imagine we have a type `MemoryByte` storing the per-location information in memory.
This is where we put the borrow stack and the information about freezing:

假设我们有一个在内存中存储每个位置信息的类型`Memory yByte`。这是我们放置借入堆栈和有关冻结的信息的位置：

{% highlight rust %}
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

{%Highlight Rust%}/有关潜在可变借入枚举Mut的信息{/唯一、可变引用Uniq(时间戳)、/任何原始指针或具有内部可变原始的共享借入、}/有关任何种类借入枚举借入的信息{/可变借入、原始指针或具有内部可变Mut的共享借入(Mut)，/不具有内部可变性的共享借入FRZ(时间戳)}/借入堆栈中的项enum BorStackItem{/定义如果位置不是冻结的Mut(Mut)，允许哪些引用发生变化，/障碍，通过其在调用堆栈上的索引跟踪它所属的函数FnBarrier(USize)}

struct MemoryByte {
borrows: Vec<BorStackItem>, // used as a stack
frz_since: Option<Timestamp>,
/\* More fields, to store the actual value and what else might be needed \*/
}
{% endhighlight %}

结构内存字节{借入：VEC，//用作堆栈frz_Since：选项，/*更多字段，用于存储实际值和其他可能需要的内容*/}{%endHighth%}

Next, we define some per-location operations that we will use later to define what happens when working with references.
Below, `assert!` is used for things that should always be true because of interpreter invariants (i.e., Miri will ICE if they fail to hold), and `bail!` is used to indicate that the program has UB.

接下来，我们定义一些每个位置的操作，稍后我们将使用这些操作来定义在使用引用时发生的情况。下面，`Assert！`用来表示由于解释器不变量(即，如果它们不能保持，Miri将被ICE)而应该总是为真的东西，而`baal！`用来表示程序有UB。

{% highlight rust %}
impl MemoryByte {

{%高亮显示%}实施内存字节{

/// Check if the given borrow may be used on this location.
fn check(&self, bor: Borrow) → bool {
match bor {
Frz(acc_t) =>
// Must be frozen at least as long as the `acc_t` says.
self.frz_since.map_or(false, |loc_t| loc_t \<= acc_t),
Mut(acc_m) =>
// Raw pointers are fine with frozen locations. This is important because &Cell is raw!
if self.frozen_since.is_some() {
acc_m.is_raw()
} else {
self.borrows.last().map_or(false, |loc_itm| loc_itm == Mut(acc_m))
}
}
}

/检查给定的借用是否可以在此位置使用。Fn check(&self，bor：→)bool{Match bor{frz(Acc_T)=>//必须至少冻结`acc_t`表示的时间。Self.frz_since.map_or(FALSE，|loc_t|loc_t<=acc_t)，Mut(Acc_M)=>//原始指针适用于冻结位置。这一点很重要，因为&Cell是原始的！If self.like_since.is_ome(){acc_m.is_raw()}Else{self.borbs.last().map_or(FALSE，|loc_itm|loc_itm==Mut(Acc_M))}}

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
FnBarrier(\_) => {
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

/重新激活此位置的给定现有借入，如果不可能，则失败。FN REACTIVATE(&MUT SELF，BOR：BORROW){//如果`bor`已经处于活动状态，请不要更改任何内容--尤其是如果//它是`Mut(Raw)`，并且我们被冻结。If self.check(Bor){Return；}let acc_m=Match bor{frz(Acc_T)=>baal！(“位置应该冻结但没有冻结”)，Mut(Acc_M)=>acc_m，}；//即使我们使用最上面的项，也一定要解冻。一直弹出，直到看到我们要找的那个人。WILLE Some(ITM)=self.borbs.last(){Match ITM{FnBarrier(_)=>{baal！(“正在尝试重新激活位于屏障后面的借款人”)；}Mut(Loc_M)=>{if loc_m==acc_m{Return；}self.借入s.op()；}}}保释！(“借入以重新激活在堆栈上不存在”)；}

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

/为该位置启动给定(新)借用。/除了它还处理`Frz`的发起之外，这是“推栈”。FN INITIATE(&MUT SELF，BOR：Borrow){Match bor{FRZ(T)=>{if self.From_since.is_None(){self.From_Since=Some(T)；}}Mut(M)=>{if m.is_Uniq()&&self.From_since.is_Some(){baal！(“冻结时不能启动Uniq！”)；}self.iobs.ush(Mut(M))；}}}

/// Reset the borrow tracking for this location.
fn reset(&mut self) {
if self.borrows.iter().any(|itm| if let FnBarrier(\_) = item { true } else { false }) {
assert!("Cannot reset while there are barriers");
}
self.frozen_since = None;
self.borrows.clear();
}

/重置此位置的借用跟踪。FN RESET(&MUT SELF){if self.借入s.iter().any(|ITM|If Let FnBarrier(_)=Item{true}Else{False}){Assert！(“有障碍时无法重置”)；}self.From_Since=None；self.borbs.lear()；}

}
{% endhighlight %}

}{%结束高亮显示%}

### 5.2 MIR operations

### 5.2 MIR操作

Finally, we enhance some MIR operations with bookkeeping, following the model I described above.
This is where the code gets more "pseudo" and less Rust. ;)

最后，我们按照我上面描述的模型，通过簿记来增强一些MIR操作。这就是代码变得更“伪”和更少锈蚀的地方。；)

For each of these operation, we iterate over all affected locations; let us call the loop variable `loc` of type `MemoryByte`.
We also have a variable `tag` with the tag of the pointer we are operating on (loading, or storing, or casting to a raw pointer, ...).

对于这些操作中的每一个，我们都迭代所有受影响的位置；让我们调用类型为`MemoyByte`的循环变量`Loc`。我们还有一个变量`tag`，其中包含我们正在操作的指针的标记(加载、存储或转换为原始指针等)。

Moreover, we have a boolean variable `in_unsafe_cell` indicating whether, according to the type of the pointer, the location we are currently working on is covered by an [`UnsafeCell`](https://doc.rust-lang.org/stable/std/cell/struct.UnsafeCell.html).
(This realizes the conditions checking whether we have interior mutability or not.)
For example, in `&Cell<i32>`, all 4 locations are inside an `UnsafeCell`.
However, in `&(i32, Cell<i32>)`, only the last 4 of the 8 covered locations are inside an `UnsafeCell`.

此外，我们有一个布尔变量`in_unsafe_cell`，表示根据指针的类型，我们当前正在处理的位置是否被`UnSafeCell`覆盖。(这实现了检查我们是否具有内部可变性的条件。)例如，在`&Cell<I32>`中，所有4个位置都在一个`UnSafeCell`中。然而，在`&(I32，Cell<I32>)`中，只有8个覆盖位置中的最后4个在`UnSafeCell`中。

Finally, given a reference type, a tag, and whether we are inside an `UnsafeCell`, we can compute the matching `Borrow`:
Mutable references use `Mut(Uniq(tag))`, shared references in an `UnsafeCell` use `Mut(Raw)` and other shared references use `Frz(tag)`.
We use `bor` to refer to the `Borrow` of the pointer we are working on.

最后，给定引用类型、标签以及我们是否在`UnSafeCell`中，我们可以计算匹配的`Borrow`：可变引用使用`Mut(uniq(Tag))`，`UnSafeCell`中的共享引用使用`Mut(Raw)`，其他共享引用使用`Frz(Tag)`。我们用`bor`来表示我们正在处理的指针的`borrow`。

Now we can look at what happens for each operation.

现在，我们可以看看每个操作发生了什么。

* Using a raw pointer directly is desugared to creating a shared reference (when reading) or a mutable reference (when writing), and using that. The appropriate steps below apply.
* Any time we use a (mutable or shared) reference to access memory, and any time we pass a reference to "the outside world" (passing it to a function, storing it in memory, returning it to our caller; also below structs or enums but not below unions or pointer indirectons), we reactivate.
  * `loc.reactivate(borrow)`.
* Any time a *new* reference is created (any time we run an expression `&mut foo` or `&foo`), we (re)borrow.
  * Bump up the clock, and remember the old time as `new_tag`.
    
    直接使用原始指针被简化为创建共享引用(在读取时)或可变引用(在写入时)，并使用它。下面的适当步骤适用。任何时候我们使用(可变的或共享的)引用来访问内存，以及任何时候我们传递对“外部世界”的引用(将其传递给函数、存储在内存中、返回给我们的调用者；也在结构或枚举之下，但不在联合或指针间接之下)，我们重新激活`loc.reactive(借入)`。每当创建新的引用(任何时候我们运行表达式`&mut foo`或`&foo`)时，我们(重新)借用。增加时钟，并将旧时间记住为`new_tag`。
  
  * Compute `new_bor` from `new_tag` and the type of the reference being created.
    
    根据`new_tag`和要创建的引用的类型计算`new_bor`。
  
  * `if loc.check(new_bor) {`
    
    \`如果Loc.check(New_Bor){`
    
    * The new borrow is already active! This can happen because a mutable reference can be shared multiple times. We do not do anything else.
      As a special exception, we do *not* reactivate `bor` even though it is "used", because that would unfreeze the location!
    `} else {`
    
    新借入已处于活动状态！之所以会发生这种情况，是因为可变引用可以多次共享。我们不做其他任何事情。作为特殊例外，我们不会重新激活`bor`，即使它是“已使用”的，因为这将解冻位置！`}否则{`
    
    * We might be creating a reference to a local variable. In that case, `loc.reset()`. Otherwise, `reactivate(bor)`.
    * `initiate(new_bor)`
    `}`
    
    我们可能正在创建对局部变量的引用。在这种情况下，请使用`Loc.Reset()`。否则，`reactive(Bor)`.`Initiate(New_Bor)``}`
  
  * Use `new_tag` for the new reference.
    
    使用`new_tag`作为新引用。

* Any time a reference is passed to us from "the outside world" (as function argument, loaded from memory, or returned from a callee; also below structs or enums but not below unions or pointer indirectons), we retag.
  * Bump up the clock, and remember the old time as `new_tag`.
  * Compute `new_bor` from `new_tag` and the type of the reference being created.
  * `reactivate(bor)`.
  * If this is a function argument coming in: `loc.borrows.push(FnBarrier(stack_height))`.
  * `initiate(new_bor)`. Note that this is a NOP if `new_bor` is already active -- in particular, if the location is frozen and this is a shared reference with interior mutability, we do *not* push anything on top of the barrier. This is important, because any reactivation that unfreezes this location must lead to UB while the barrier is still present.
  * Change reference tag to `new_tag`.
* Any time a raw pointer is created from a reference, we might have to do a raw reborrow.
  * `reactivate(bor)`.
  * `initiate(Mut(Raw))`. This is a NOP when coming from a shared reference.
* Any time a function returns, we have to clean up the barriers.
  * Iterate over all of memory and remove the matching `FnBarrier`. This is where the "stack" becomes a bit of a lie, because we also remove barriers from the middle of a stack.<br>
    This could be optimized by adding an indirection, so we just have to record somewhere that this function call has ended.
* Any time memory is deallocated, this counts as mutation, so the usual rules for that apply. After that, the stacks of the deallocated bytes must not contain any barriers.

If you want to test your own understanding of "Stacked Borrows", I invite you to go back to \[Section 2.2 of "Types as Contracts"\]({% post_url 2017-07-17-types-as-contracts %}#22-examples) and look at the three examples here.
Ignore the `Validate` calls, that part is no longer relevant.
These are examples of optimizations we would like to be valid, and in fact all three of them are still valid with "Stacked Borrows".
Can you argue why that is the case?

任何时候从“外部世界”传递给我们的引用(作为函数参数，从内存加载，或从被调用者返回；也在结构或枚举下面，但不在联合或指针间接下面)，我们都会重新记录。调高时钟，并记住旧时间为`new_tag`。从`new_tag`计算`new_bor`和正在创建的引用的类型。`reactive(Bor)`。如果这是传入的函数参数：`loc.borrows.push(FnBarrier(stack_height))`.`initiate(new_bor)`.请注意，如果`new_bor`已经处于活动状态，则这是一个NOP--特别是，如果位置被冻结，并且这是具有内部可变性的共享引用，则我们不会在屏障之上推送任何内容。这一点很重要，因为任何解冻该位置的重新激活都必须在障碍仍然存在的情况下导致UB。将引用标签更改为`reborrow.`reactivate(bor)`.`initiate(Mut(Raw))`._tag`。每次从引用创建原始指针时，我们都可能需要执行原始新标记这是一个来自共享引用的NOP。每当函数返回时，我们都必须清除屏障。在所有内存上迭代并移除匹配的`FnBarrier`。这就是“堆栈”变得有点像谎言的地方，因为我们还清除了堆栈中间的障碍。这可以通过添加一个间接地址来优化，所以我们只需要在某个地方记录这个函数调用已经结束。任何时候释放内存，这都被算作突变，所以通常的规则适用于此。在此之后，释放的字节堆栈不能包含任何屏障。如果您想测试自己对“堆叠借阅”的理解，我邀请您回到[“类型作为合同”的第2.2节]({%post_url 2017-07-17-Types-as-Contracts%}#22-Examples)，并查看这里的三个示例。忽略`Valiate`调用，该部分不再相关。这些都是我们希望有效的优化示例，事实上，这三个优化在“堆叠借阅”中仍然有效。你能解释一下为什么会是这样吗？

## Summary

## 摘要

I have described (yet) another Rust memory model that defines when a reference may be used to perform which memory operations.
The main design constraint of this model is that lifetimes should not matter for program execution.
To my own surprise, the model actually ended up being fairly simple, all things considered.

我已经描述了另一个Rust内存模型，该模型定义了何时可以使用引用来执行哪些内存操作。此模型的主要设计约束是生命周期不应对程序执行产生影响。令我惊讶的是，考虑到所有因素，这个模型实际上最终相当简单。

I think I covered most of the relevant features, though I will have to take a closer look at two-phase borrows and see if they need some further changes to the model.

我想我已经涵盖了大部分相关功能，不过我将不得不更仔细地研究两阶段借贷，看看他们是否需要对模型进行一些进一步的更改。

Of course, now the big question is whether this model actually "works" -- does it permit all the code we want to permit (does it even permit all safe code), and does it rule out enough code such that we can get useful optimizations?
I hope to explore this question further in the following weeks by implementing a dynamic checker to test the model on real code.
It is just easier to answer these questions when you do not have to *manually* reevaluate all examples after every tiny change.
However, I can always use more examples, so if you think you found some interesting or surprising corner case, please let me know!

当然，现在最大的问题是这个模型是否真的“有效”--它是否允许我们想要允许的所有代码(它甚至允许所有安全代码)，以及它是否排除了足够的代码，以便我们可以获得有用的优化？我希望在接下来的几周里通过实现一个动态检查器来进一步探讨这个问题，以在实际代码上测试该模型。当您不必在每个微小的更改后手动重新评估所有示例时，回答这些问题就更容易了。但是，我总是可以使用更多的例子，所以如果你认为你发现了一些有趣的或令人惊讶的角落案例，请让我知道！

As always, if you have any questions or comments, feel free to [ask in the forums](https://internals.rust-lang.org/t/stacked-borrows-an-aliasing-model-for-rust/8153).

像往常一样，如果您有任何问题或意见，请随时在论坛上提问。