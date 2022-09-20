---

## layout: post
title: 'Dyn async traits, part 8: the soul of Rust'
date: 2022-09-18 13:49 -0400

## 版面：帖子标题：《Dyn Async Characters，Part 8：The灵魂of Rust》日期：2022-09-18 13：49-0400

In the last few months, Tyler Mandry and I have been circulating a [“User’s Guide from the Future”](https://hackmd.io/@nikomatsakis/SJ2-az7sc) that describes our current proposed design for async functions in traits. In this blog post, I want to deep dive on one aspect of that proposal: how to handle dynamic dispatch. My goal here is to explore the space a bit and also to address one particularly tricky topic: how explicit do we have to be about the possibility of allocation? This is a tricky topic, and one that gets at that core question: what is the soul of Rust?

在过去的几个月里，泰勒·曼德里和我一直在分发一份《来自未来的用户指南》，其中描述了我们目前为特征中的异步功能提出的设计方案。在这篇博客文章中，我想深入探讨该建议的一个方面：如何处理动态调度。我在这里的目标是探索一下这个空间，并解决一个特别棘手的话题：我们必须对分配的可能性有多明确？这是一个棘手的话题，也是一个切中核心问题的话题：铁锈的灵魂是什么？

### The running example trait

### 正在运行的示例特征

Throughout this blog post, I am going to focus exclusively on this example trait, `AsyncIterator`:

在这篇博客文章中，我将专门关注这个示例特征，`AsyncIterator`：

```rust
trait AsyncIterator {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}
```

And we’re particularly focused on the scenario where we are invoking `next` via dynamic dispatch:

我们特别关注通过动态调度调用`next`的场景：

```rust
fn make_dyn<AI: AsyncIterator>(ai: AI) {
    use_dyn(&mut ai); // <— coercion from `&mut AI` to `&mut dyn AsyncIterator`
}

fn use_dyn(di: &mut dyn AsyncIterator) {
    di.next().await; // <— this call right here!
}
```

Even though I’m focusing the blog post on this particular snippet of code, everything I’m talking about is applicable to any trait with methods that return `impl Trait` (async functions themselves being a shorthand for a function that returns `impl Future`).

尽管我的博客文章将重点放在这段特定的代码上，但我所讨论的所有内容都适用于具有返回`impl Trait`的方法的任何特征(异步函数本身是返回`impl Future`的函数的简写)。

The basic challenge that we have to face is this:

我们必须面对的基本挑战是：

* The caller function, `use_dyn`, doesn’t know what impl is behind the `dyn`, so it needs to allocate a fixed amount of space that works for everybody. It also needs some kind of vtable so it knows what `poll` method to call.
* The callee, `AI::next`, needs to be able to package up the future for its `next` function in some way to fit the caller’s expectations.

The [first blog post in this series](https://smallcultfollowing.com/babysteps/blog/2021/09/30/dyn-async-traits-part-1/)[^2020] explains the problem in more detail.

调用方函数`use_dyn`不知道`dyn`后面的impl是什么，所以它需要分配一个固定大小的空间，每个人都可以使用。它还需要某种vtable，这样它才知道要调用什么`poll`方法。被调用方`AI：：next`需要能够以某种方式为其`next`函数打包未来，以满足调用方的期望。本系列的第一篇博客文章更详细地解释了这个问题。

[^2020]: Written in Sep 2020, egads!

写于2020年9月，天哪！

### A brief tour through the options

### 简要介绍这些选项

One of the challenges here is that there are many, many ways to make this work, and none of them is “obviously best”. What follows is, I think, an exhaustive list of the various ways one might handle the situation. If anybody has an idea that doesn’t fit into this list, I’d love to hear it.

这里面临的挑战之一是，有很多很多方法可以做到这一点，但没有一种方法“显然是最好的”。我认为，以下是一个人可能处理这种情况的各种方法的详尽清单。如果有人有不符合这个清单的想法，我很乐意听听。

**Box it.** The most obvious strategy is to have the callee box the future type, effectively returning a `Box<dyn Future>`, and have the caller invoke the `poll` method via virtual dispatch. This is what the [`async-trait`](https://crates.io/crates/async-trait) crate does (although it also boxes for static dispatch, which we don’t have to do).

把它装上盒子。最明显的策略是让被调用方框为Future类型，有效地返回一个`Box<dyn Future>`，并让调用方通过虚拟分派调用`poll`方法。这就是‘Async-Trait`板条箱’所做的(尽管它也对静态分派进行装箱，这是我们不必做的)。

**Box it with some custom allocator.** You might want to box the future with a custom allocator.

用一些定制的分配器把它装箱。您可能希望使用自定义分配器来装箱未来。

**Box it and cache box in the caller.** For most applications, boxing itself is not a performance problem, unless it occurs repeatedly in a tight loop. Mathias Einwag pointed out if you have some code that is repeatedly calling `next` on the same object, you could have that caller cache the box in between calls, and have the callee reuse it. This way you only have to actually allocate once.

将其装箱并将其缓存在调用方中。对于大多数应用程序来说，装箱本身并不是性能问题，除非它在一个紧密的循环中重复出现。Mathias Einwag指出，如果你有一些代码在同一个对象上重复调用`next`，你可以让调用者在调用之间缓存盒子，并让被调用者重复使用它。这样，您只需实际分配一次。

**Inline it into the iterator.** Another option is to store all the state needed by the function in the `AsyncIter` type itself. This is actually what the existing `Stream` trait does, if you think about it: instead of returning a future, it offers a `poll_next` method, so that the implementor of `Stream` effectively *is* the future, and the caller doesn’t have to store any state. Tyler and I worked out a more general way to do inlining that doesn’t require user intervention, where you basically wrap the `AsyncIterator` type in another type `W` that has a field big enough to store the `next` future. When you call `next`, this wrapper `W` stores the future into that field and then returns a pointer to the field, so that the caller only has to poll that pointer. **One problem with inlining things into the iterator is that it only works well for `&mut self` methods**, since in that case there can be at most one active future at a time. With `&self` methods, you could have any number of active futures.

将其内联到迭代器中。另一种选择是将函数需要的所有状态存储在`AsyncIter`类型本身中。这实际上就是现有的`Stream`特征所做的，如果你仔细想想的话：它不是返回未来，而是提供了一个`polnext`方法，所以`Stream`的实现者实际上就是未来，调用者不必存储任何状态。泰勒和我设计了一种更通用的内联方法，这种方法不需要用户干预，基本上是将`AsyncIterator‘类型包装在另一种类型`W’中，该类型的字段足够大来存储`Next‘的未来。当您调用`next`时，这个包装器`W`将未来存储到该字段中，然后返回指向该字段的指针，因此调用者只需轮询该指针。将对象内联到迭代器的一个问题是，它只适用于`&mut self‘方法，因为在这种情况下，一次最多只能有一个活动的未来。使用`&self‘方法，您可以拥有任意数量的活跃期货。

**Box it and cache box in the callee.** Instead of inlining the entire future into the `AsyncIterator` type, you could inline just one pointer-word slot, so that you can cache and reuse the `Box` that `next` returns. The upside of this strategy is that the cached box moves with the iterator and can potentially be reused across callers. The downside is that once the caller has finished, the cached box lives on until the object itself is destroyed.

将其装箱并将其缓存在被调用方中。不是将整个未来内联到`AsyncIterator`类型中，而是只内联一个指针字槽，这样就可以缓存和重用`next`返回的`Box`。此策略的优点是缓存的框随迭代器移动，并且有可能在调用方之间重复使用。缺点是，一旦调用方完成调用，缓存的框就会一直存在，直到对象本身被销毁。

**Have caller allocate maximal space.** Another strategy is to have the caller allocate a big chunk of space on the stack, one that should be big enough for every callee. If you know the callees your code will have to handle, and the futures for those callees are close enough in size, this strategy works well. Eric Holk recently released the \[stackfuture crate\] that can help automate it. **One problem with this strategy is that the caller has to know the size of all its callees.**

让调用方分配最大空间。另一种策略是让调用者在堆栈上分配一大块空间，该空间应该足够大，供每个被调用者使用。如果您知道您的代码将必须处理的被调用方，并且这些被调用方的未来在大小上足够接近，则此策略效果很好。埃里克·霍克最近发布了[StackFuture板条箱]，它可以帮助实现自动化。这种策略的一个问题是，调用方必须知道其所有被调用方的大小。

**Have caller allocate some space, and fall back to boxing for large callees.** If you don’t know the sizes of all your callees, or those sizes have a wide distribution, another strategy might be to have the caller allocate some amount of stack space (say, 128 bytes) and then have the callee invoke `Box` if that space is not enough.

让调用者分配一些空间，并为较大的被调用者回退到拳击。如果您不知道所有被调用者的大小，或者这些大小分布很广，另一种策略可能是让调用者分配一定数量的堆栈空间(例如，128字节)，然后在空间不足的情况下让被调用者调用`Box`。

**Alloca on the caller side.** You might think you can store the size of the future to be returned in the vtable and then have the caller “alloca” that space — i.e., bump the stack pointer by some dynamic amount. Interestingly, this doesn’t work with Rust’s async model. Async tasks require that the size of the stack frame is known up front. 

呼叫方的Alloca。您可能认为您可以在vtable中存储要返回的未来的大小，然后让调用者“alloca”获得该空间-即，将堆栈指针增加一些动态量。有趣的是，这不适用于Rust的异步模型。异步任务要求预先知道堆栈帧的大小。

**Side stack.** Similar to the previous suggestion, you could imagine having the async runtimes provide some kind of “dynamic side stack” for each task.[^ada] We could then allocate the right amount of space on this stack. This is probably the most efficient option, but it assumes that the runtime is able to provide a dynamic stack. Runtimes like [embassy](https://github.com/embassy-rs/embassy) wouldn’t be able to do this. Moreover, we don’t have any sort of protocol for this sort of thing right now. Introducing a side-stack also starts to “eat away” at some of the appeal of Rust’s async model, which is [designed to allocate the “perfect size stack” up front](https://without.boats/blog/futures-and-segmented-stacks/) and avoid the need to allocate a “big stack per task”.[^heap]

侧面堆叠。与前面的建议类似，您可以想象让异步运行时为每个任务提供某种“动态侧堆栈”。然后，我们可以在此堆栈上分配适当数量的空间。这可能是最有效的选择，但它假定运行库能够提供动态堆栈。像EMBASSY这样的运行时无法做到这一点。此外，我们目前还没有针对这类事情的任何类型的协议。引入侧堆栈也开始“蚕食”Rust的异步模型的一些吸引力，该模型旨在预先分配“完美大小的堆栈”，并避免“每个任务分配一个大堆栈”的需要。

[^heap]: Of course, without a side stack, we are left using mechanisms like `Box::new` to cover cases like dynamic dispatch or recursive functions. This becomes a kind of pessimistically sized segmented stack, where we allocate for each little piece of extra state that we need. A side stack might be an appealing middle ground, but because of cases like `embassy`, it can’t be the only option.

当然，如果没有侧堆栈，我们只能使用`Box：：new`这样的机制来覆盖动态调度或递归函数等情况。这变成了一种悲观大小的分段堆栈，我们为所需的每一小块额外状态进行分配。侧面堆栈可能是一个很吸引人的中间立场，但由于像`embassys‘这样的情况，它不可能是唯一的选择。

[^ada]: I was intrigued to learn that this is what Ada does, and that Ada features like returning dynamically sized types are built on this model. I’m not sure how [SPARK](https://www.adacore.com/about-spark) and other Ada subsets that target embedded spaces manage that, I’d like to learn more about it.

我很感兴趣地了解到这就是Ada所做的，并且像返回动态大小类型这样的Ada功能都构建在这个模型上。我不确定Spark和其他针对嵌入式空间的Ada子集是如何做到这一点的，我想了解更多。

### Can async functions used with dyn be “normal”?

### 与dyn一起使用的异步功能可以是“正常的”吗？

One of my initial goals for async functions in traits was that they should feel “as natural as possible”. In particular, I wanted you to be able to use them with dynamic dispatch in just the same way as you would a synchronous function. In other words, I wanted this code to compile, and I would want it to work even if `use_dyn` were put into another crate (and therefore were compiled with no idea of who is calling it):

对于特征中的异步功能，我最初的目标之一是让它们感觉“尽可能自然”。特别是，我希望您能够像使用同步函数一样将它们与动态分派一起使用。换句话说，我希望编译这段代码，并且即使将`use_dyn`放入另一个箱子中(因此编译时不知道是谁在调用它)，我也希望它能够工作：

```rust
fn make_dyn<AI: AsyncIterator>(ai: AI) {
    use_dyn(&mut ai);
}

fn use_dyn(di: &mut dyn AsyncIterator) {
    di.next().await;
}
```

My hope was that we could make this code work *just as it is* by selecting some kind of default strategy that works most of the time, and then provide ways for you to pick other strategies for those code where the default strategy is not a good fit. The problem though is that there is no single default strategy that seems “obvious and right almost all of the time”…

我的希望是，我们可以通过选择某种在大多数情况下都有效的默认策略来使该代码正常工作，然后为那些默认策略不太适合的代码选择其他策略。但问题是，没有哪一种默认策略似乎在几乎所有时间都是“显而易见和正确的”…

|Strategy	战略|Downside	负面影响|
|--------|--------|
|Box it (with default allocator)	将其装箱(使用默认分配器)|requires allocation, not especially efficient	需要分配，不是特别高效|
|Box it with cache on caller side	在呼叫方端使用缓存将其装箱|requires allocation	需要分配|
|Inline it into the iterator	将其内联到迭代器中|adds space to 	增加空间到`AI`, doesn’t work for 	\`Ai`，不适用于`&self`	\`自我`|
|Box it with cache on callee side	在被呼叫方一侧使用缓存将其装箱|requires allocation, adds space to 	需要分配，添加空间到`AI`, doesn’t work for 	\`Ai`，不适用于`&self`	\`自我`|
|Allocate maximal space	分配最大空间|can’t necessarily use that across crates, requires extensive interprocedural analysis	不一定能跨板条箱使用，需要广泛的程序间分析|
|Allocate some space, fallback	分配一些空间，后备|uses allocator, requires extensive interprocedural analysis or else random guesswork	使用分配器，需要广泛的过程间分析或随机猜测|
|Alloca on the caller side	呼叫方的Alloca|incompatible with async Rust	与异步铁锈不兼容|
|Side-stack	侧堆叠|requires cooperation from runtime and allocation	需要来自运行时和分配的协作|

### The soul of Rust

### 铁锈之魂

This is where we get to the “soul of Rust”. Looking at the above table, the strategy that seems the closest to “obviously correct” is “box it”. It works fine with separate compilation, fits great with Rust’s async model, and it matches what people are doing today in practice. I’ve spoken with a fair number of people who use async Rust in production, and virtually all of them agreed that “box by default, but let me control it” would work great in practice.

这就是我们到达“铁锈之魂”的地方。看看上面的表格，看起来最接近“明显正确”的策略是“把它放在盒子里”。它在单独编译的情况下工作得很好，非常适合Rust的异步模型，而且它符合今天人们在实践中所做的事情。我与相当多在生产中使用Async Rust的人交谈过，几乎所有人都同意“默认情况下，但让我控制它”在实践中会很好地工作。

And yet, when we floated the idea of using this as the default, Josh Triplett objected strenuously, and I think for good reason. Josh’s core concern was that this would be crossing a line for Rust. Until now, there is no way to allocate heap memory without some kind of explicit operation (though that operation could be a function call). But if we wanted make “box it” the default strategy, then you’d be able to write “innocent looking” Rust code that nonetheless *is* invoking `Box::new`. In particular, it would be invoking `Box::new` each time that `next` is called, to box up the future. But that is very unclear from reading over `make_dyn` and `use_dyn`.

然而，当我们提出将此作为默认设置的想法时，乔希·特里普莱特强烈反对，我认为这是有充分理由的。乔希的核心担心是，这对拉斯特来说是越界了。到目前为止，如果没有某种显式操作(尽管该操作可能是函数调用)，就无法分配堆内存。但是，如果我们希望将“box it”设置为默认策略，那么您可以编写“看起来很无辜”的Rust代码，但它仍然会调用`Box：：New`。具体地说，它将在每次调用`next`时调用`Box：：new`，以装箱未来。但从阅读`make_dyn`和`use_dyn`来看，这一点非常不清楚。

As an example of where this might matter, it might be that you are writing some sensitive systems code where allocation is something you always do with great care. It doesn’t mean the code is no-std, it may have access to an allocator, but you still would like to know exactly where you will be doing allocations. Today, you can audit the code by hand, scanning for “obvious” allocation points like `Box::new` or `vec![]`. Under this proposal, while it would still be *possible*, the presence of an allocation in the code is much less obvious. The allocation is “injected” as part of the vtable construction process. To figure out that this will happen, you have to know Rust’s rules quite well, and you also have to know the signature of the callee (because in this case, the vtable is built as part of an implicit coercion). In short, scanning for allocation went from being relatively obvious to requiring a PhD in Rustology. Hmm.

例如，您可能正在编写一些敏感的系统代码，在这些代码中，分配始终是您非常小心地处理的事情。这并不意味着代码是非标准的，它可能可以访问分配器，但是您仍然想知道您将在哪里进行分配。如今，您可以手动审核代码，扫描`Box：：new`或`vec！[]`等“明显”的分配点。在这一提议下，虽然仍有可能，但代码中存在分配的情况就不那么明显了。作为vtable构造过程的一部分，该分配被“注入”。要确定这是否会发生，您必须非常了解Rust的规则，并且还必须知道被调用者的签名(因为在本例中，vtable是作为隐式强制的一部分构建的)。简而言之，扫描分配从相对明显地变成了需要Rustology的博士学位。嗯。

On the other hand, if scanning for allocations is what is important, we could address that in many ways. We could add an “allow by default” lint to flag the points where the “default vtable” is constructed, and you could enable it in your project. This way the compiler would warn you about the possible future allocation. In fact, even today, scanning for allocations is actually much harder than I made it ought to be: you can easily see if your function allocates, but you can’t easily see what its callees do. You have to read deeply into all of your dependencies and, if there are function pointers or `dyn Trait` values, figure out what code is potentially being called. With compiler/language support, we could make that whole process much more first-class and better.

另一方面，如果扫描分配是重要的，我们可以通过许多方式解决这个问题。我们可以添加一个“默认允许”的lint来标记构造“默认vtable”的点，并且您可以在项目中启用它。这样，编译器就会警告您未来可能的分配。事实上，即使在今天，扫描分配实际上也比我应该做的要难得多：您可以很容易地看到您的函数是否进行了分配，但您不能轻松地看到它的被调用者做了什么。您必须深入阅读您的所有依赖项，如果有函数指针或`dyn Trait`值，则找出可能被调用的代码。有了编译器/语言支持，我们可以使整个过程变得更一流和更好。

In a way, though, the technical arguments are besides the point. “Rust makes allocations explicit” is widely seen as a key attribute of Rust’s design. In making this change, we would be tweaking that rule to be something like ”Rust makes allocations explicit *most of the time*”. This would be harder for users to understand, and it would introduce doubt as whether Rust *really* intends to be the kind of language that can replace C and C++[^coro].

然而，在某种程度上，技术上的争论并不是重点。“Rust使分配显式”被广泛视为Rust设计的一个关键属性。在做出这一改变时，我们将把这一规则调整为类似于“铁锈在大部分时间内明确分配”的规则。这会让用户更难理解，也会引起人们的怀疑，比如Rust是否真的打算成为一种可以取代C和C++的语言。

[^coro]: Ironically, C++ itself inserts implicit heap allocations to help with coroutines!

具有讽刺意味的是，C++本身插入了隐式堆分配来帮助执行协程！

### Looking to the Rustacean design principles for guidance

### 以Rustacean的设计原则为指导

Some time back, Josh and I drew up a draft set of design principles for Rust. It’s interesting to look back on them and see what they have to say about this question:

不久前，乔希和我为Rust起草了一套设计原则草案。回顾他们，看看他们对这个问题有什么看法，这是很有趣的：

* ⚙️ Reliable: "if it compiles, it works"
* 🐎 Performant: "idiomatic code runs efficiently"
* 🥰 Supportive: "the language, tools, and community are here to help"
* 🧩 Productive: "a little effort does a lot of work"
* 🔧 Transparent: "you can predict and control low-level details"
* 🤸 Versatile: "you can do anything with Rust"

Boxing by default, to my mind, scores as follows:

⚙️Reliable：“如果它编译了，它就能工作”🐎Performant：“惯用代码高效运行”🥰Support：语言、工具和社区在这里帮助“🧩Productive：”一点努力就能做很多工作“🔧透明：”你可以预测和控制低级别的细节“🤸多功能：”你可以用铁锈做任何事情“在我看来，默认情况下，得分如下：

* **🐎 Performant: meh.** The real goal with performant is that the cleanest code also runs the *fastest*. Boxing on every dynamic call doesn’t meet this goal, but something like “boxing with caller-side caching” or “have caller allocate space and fall back to boxing” very well might.
* **🧩 Productive: yes!** Virtually every production user of async Rust that I’ve talked to has agreed that having code box by default would (but giving the option to do something else for tight loops) would be a great sweet spot for Rust.
* **🔧 Transparent: no.** As I wrote before, understanding when a call may box now requires a PhD in Rustology, so this definitely fails on transparency.

(The other principles are not affected in any notable way, I don't think.)

🐎表演者：嗯。Performant的真正目标是最干净的代码也运行得最快。在每个动态调用上装箱并不能达到这个目标，但是像“使用调用方缓存进行装箱”或“让调用方分配空间并回退到装箱”这样的事情很有可能达到这个目标。🧩Productive：是的！实际上，与我交谈过的每一位异步Rust的生产用户都同意，默认设置代码框(但允许选择做其他事情来处理循环)将是Rust的最佳选择。🔧透明：不。正如我之前所写的那样，现在理解一次通话可能需要Rustology的博士学位，所以这在透明度方面肯定是失败的。(我认为，其他原则没有受到任何显著的影响。)

### What the “user’s guide from the future” suggests

### 《未来用户指南》的建议

These considerations led Tyler and I to a different design. In the [“User’s Guide From the Future”](https://hackmd.io/@nikomatsakis/SJ2-az7sc) document from before, you’ll see that it does not accept the running example just as is. Instead, if you were to compile the example code we’ve been using thus far, you’d get an error:

这些考虑导致泰勒和我进行了不同的设计。在前面的“User‘s Guide from the Future”文档中，您将看到它并不完全接受正在运行的示例。相反，如果您编译我们到目前为止一直在使用的示例代码，您会得到一个错误：

```
error[E0277]: the type `AI` cannot be converted to a
              `dyn AsyncIterator` without an adapter
 --> src/lib.rs:3:23
  |
3 |     use_dyn(&mut ai);
  |                  ^^ adapter required to convert to `dyn AsyncIterator`
  |
  = help: consider introducing the `Boxing` adapter,
    which will box the futures returned by each async fn
3 |     use_dyn(&mut Boxing::new(ai));
                     ++++++++++++  +
```

As the error suggests, in order to get the boxing behavior, you have to opt-in via a type that we called `Boxing`[^bikeshed]:

如错误所示，为了获得装箱行为，您必须通过我们称为`Boxing`的类型选择加入：

[^bikeshed]: Suggestions for a better name very welcome.

对于更好的名字的建议非常受欢迎。

```rust
fn make_dyn<AI: AsyncIterator>(ai: AI) {
    use_dyn(&mut Boxing::new(ai));
    //          ^^^^^^^^^^^
}

fn use_dyn(di: &mut dyn AsyncIterator) {
    di.next().await;
}
```

Under this design, you can only create a `&mut dyn AsyncIterator` when the caller can verify that the `next` method returns a type from which a `dyn*` can be constructed. If that’s not the case, and it’s usually not, you can use the `Boxing::new` adapter to create a `Boxing<AI>`. Via some kind of compiler magic that *ahem* we haven’t fully worked out yet[^pay-no-attention], you could coerce a `Boxing<AI>` into a `dyn AsyncIterator`.

在这种设计下，只有当调用方可以验证`next`方法返回一个可以构造`dyn*`的类型时，才能创建`&mut dyn AsyncIterator`。如果不是这样，通常也不是这样，您可以使用`Boxing：：new`适配器来创建`Boxing<AI>`。通过某种我们还没有完全弄明白的编译器魔术，你可以强迫一个`Boxing<AI>`变成一个`dyn AsyncIterator‘。

[^pay-no-attention]: Pay no attention to the compiler author behind the curtain. 🪄 🌈 Avert your eyes!

不要理会幕后的编译器作者。🪄🌈把你的眼睛移开！

**The details of the `Boxing` type need more work[^UMFchange], but the basic idea remains the same: require users to make *some* explicit opt-in to the default vtable strategy, which may indeed perform allocation.**

\`Boxing`类型的细节需要做更多的工作，但基本思想保持不变：要求用户显式地选择加入默认的vtable策略，这可能确实会执行分配。

[^UMFchange]: e.g., if you look closely at the [User's Guide from the Future](https://hackmd.io/@nikomatsakis/SJ2-az7sc), you'll see that it writes `Boxing::new(&mut ai)`, and not `&mut Boxing::new(ai)`. I go back and forth on this one.

例如，如果你仔细查看来自未来的用户指南，你会发现它写的是`Boxing：：new(&mut ai)`，而不是`&mut Boxing：：new(Ai)`。在这个问题上，我反复考虑。

### How does `Boxing` rank on the design principles?

### 《拳击》在设计原则上排名如何？

To my mind, adding the `Boxing` adapter ranks as follows…

在我看来，添加`Boxing`适配器的排名如下…

* **🐎 Performant: meh.** This is roughly the same as before. We’ll come back to this.
* **🥰 Supportive: yes!** The error message guides you to exactly what you need to do, and hopefully links to a well-written explanation that can help you learn about why this is required.
* **🧩 Productive: meh.** Having to add `Boxing::new` call each time you create a `dyn AsyncIterator` is not great, but also on-par with other Rust papercuts.
* **🔧 Transparent: yes!** It is easy to see that boxing may occur in the future now.

This design is now transparent. It’s also less productive than before, but we’ve tried to make up for it with supportiveness. “Rust isn’t always easy, but it’s always helpful.”

🐎表演者：嗯。这与之前大致相同。我们会回到这个话题的。🥰Support：是的！该错误消息将指导您准确地执行所需操作，并希望链接到一个写得很好的解释，帮助您了解为什么需要这样做。🧩Productive：Meh。每次创建‘dyn AsyncIterator’时都要添加`Boxing：：New`调用，这并不是很好，但也与其他铁锈剪纸不相上下。🔧透明：是的！现在很容易看出拳击可能会在未来发生。这种设计现在是透明的。它的生产力也低于以前，但我们试图用支持来弥补这一点。“生锈并不总是那么容易，但它总是有帮助的。”

### Improving performance with a more complex ABI

### 通过更复杂的ABI提高性能

One thing that bugs me about the “box by default” strategy is that the performance is only “meh”. I like stories like `Iterator`, where you write nice code and you get tight loops. It bothers me that writing “nice” async code yields a naive, middling efficiency story.

“默认设置框”策略让我感到困扰的一件事是，它的性能只有“meh”。我喜欢《迭代器》这样的故事，在这种故事中，你写出了很好的代码，你得到了紧凑的循环。编写“好的”异步代码会产生一个天真的、中等效率的故事，这让我感到不安。

That said, I think this is something we could fix in the future, and I think we could fix it backwards compatibly. The idea would be to extend our ABI when doing virtual calls so that the caller has the *option* to provide some “scratch space” for the callee. For example, we could then do things like analyze the binary to get a good guess as to how much stack space is needed (either by doing dataflow or just by looking at all implementations of `AsyncIterator`). We could then have the caller reserve stack space for the future and pass a pointer into the callee — the callee would still have the *option* of allocating, if for example, there wasn’t enough stack space, but it could make use of the space in the common case.

也就是说，我认为这是我们未来可以修复的东西，我认为我们可以向后兼容地修复它。我们的想法是在进行虚拟调用时扩展我们的ABI，以便调用者可以选择为被调用者提供一些“临时空间”。例如，然后我们可以分析二进制文件，以很好地猜测需要多少堆栈空间(通过执行数据流或仅通过查看`AsyncIterator‘的所有实现)。然后，我们可以让调用方为将来保留堆栈空间，并将一个指针传递给被调用方-被调用方仍有分配的选项，例如，如果没有足够的堆栈空间，但在常见情况下它可以利用这些空间。

Interestingly, I think that if we did this, we would also be putting some pressure on Rust’s “transparency” story again. While Rust’s leans heavily on optimizations to get performance, we’ve generally restricted ourselves to simple, local ones like inlining; we don’t require interprocedural dataflow in particular, although of course it helps (and LLVM does it). But getting a good estimate of how much stack space to reserve for potential calleees would violate that rule (we’d also need some simple escape analysis, as I describe in [Appendix A](#Appendix-A-futures-that-escape-the-stack-frame)). All of this adds up to a bit of ‘performance unpredictability’. Still, I don’t see this as a big problem, particularly since the fallback is just to use `Box::new`, and as we’ve said, for most users that is perfectly adequate.

有趣的是，我认为，如果我们这样做，我们也会再次对拉斯特的“透明度”故事施加一些压力。虽然Rust在很大程度上依赖优化来获得性能，但我们通常将自己限制在简单的本地优化，如内联；我们不需要特别的过程间数据流，尽管它当然有帮助(LLVM就是这样做的)。但是，很好地估计为潜在的被调用者保留多少堆栈空间将违反该规则(我们还需要一些简单的转义分析，如我在附录A中所描述的)。所有这一切加在一起，就是有点“性能的不可预测性”。尽管如此，我不认为这是一个大问题，特别是因为后备只是使用`Box：：new`，正如我们已经说过的，对于大多数用户来说，这是完全足够的。

### Picking another strategy, such as inlining

### 选择另一种策略，例如内联

Of course, maybe you don’t want to use `Boxing`. It would also be possible to construct other kinds of adapters, and they would work in a similar fashion. For example, an inlining adapter might look like:

当然，也许你不想用`拳击‘。还可以构建其他类型的适配器，它们将以类似的方式工作。例如，内联适配器可能如下所示：

```rust
fn make_dyn<AI: AsyncIterator>(ai: AI) {
    use_dyn(&mut InlineAsyncIterator::new(ai));
    //           ^^^^^^^^^^^^^^^^^^^^^^^^
}
```

The `InlineAsyncIterator<AI>` type would add the extra space to store the future, so that when the `next` method is called, it writes the future into its own fields and then returns it to the caller. Similarly, a cached box adapter might be `&mut CachedAsyncIterator::new(ai)`, only it would use a field to cache the resulting `Box`.

\`InlineAsyncIterator<AI>`类型将添加额外的空间来存储未来，因此当调用`next`方法时，它将未来写入自己的字段中，然后将其返回给调用方。同样，缓存的Box适配器可能是`&mut CachedAsyncIterator：：new(Ai)`，只是它会使用一个字段来缓存结果`Box`。

You may have noticed that the inline/cached adapters include the name of the trait. That’s because they aren’t relying on compiler magic like Boxing, but are instead intended to be authored by end-users, and we don’t yet have a way to be generic over any trait definition. (The proposal as we wrote it uses macros to generate an adapter type for any trait you wish to adapt.) This is something I’d love to address in the future. [You can read more about how adapters work here.](https://rust-lang.github.io/async-fundamentals-initiative/explainer/inline_async_iter_adapter.html)

您可能已经注意到，内联/缓存适配器包括特征的名称。这是因为它们不依赖于像Boxing那样的编译器魔力，而是打算由最终用户创作，而我们还没有办法在任何特征定义上实现泛型。(我们编写的提案使用宏为您希望适应的任何特性生成适配器类型。)这是我未来很乐意解决的问题。您可以在此处阅读有关适配器如何工作的更多信息。

### Conclusion

### 结论

OK, so let’s put it all together into a coherent design proposal:

好的，让我们把所有这些放到一个连贯的设计方案中：

* You cannot coerce from an arbitrary type `AI` into a `dyn AsyncIterator`. Instead, you must select an adaptor:
  * Typically you want `Boxing`, which has a decent performance profile and “just works”.
  * But users can write their own adapters to implement other strategies, such as `InlineAsyncIterator` or `CachingAsyncIterator`.
* From an implementation perspective:
  * When invoked via dynamic dispatch, async functions return a `dyn* Future`. The caller can invoke `poll` via virtual dispatch and invoke the (virtual) drop function when it’s ready to dispose of the future.
  * The vtable created for `Boxing<AI>` will allocate a box to store the future `AI::next()` and use that to create the `dyn* Future`.
  * The vtable for other adapters can use whatever strategy they want. `InlineAsyncIterator<AI>`, for example, stores the `AI::next()` future into a field in the wrapper, takes a raw pointer to that field, and creates a `dyn* Future` from this raw pointer.
* Possible future extension for better performance:[^tmandry]
  * We modify the ABI for async trait functions (or any trait function using return-position impl trait) to allow the caller to optionally provide stack space. The `Boxing` adapter, if such stack space is available, will use it to avoid boxing when it can. This would have to be coupled with some compiler analysis to figure out how much to stack space to pre-allocate.

[^tmandry]: I should clarify that, while Tyler and I have discussed this, I don't know how he feels about it. I wouldn't call it 'part of the proposal' exactly, more like an extension I am interested in.

不能将任意类型的`AI`强制为`dyn AsyncIterator`。相反，你必须选择一个适配器：通常你想要`Boxing`，它有不错的性能配置文件，并且“刚刚好”。但用户也可以编写自己的适配器来实现其他策略，如`InlineAsyncIterator`或`CachingAsyncIterator`。从实现的角度来看：当通过动态调度调用时，异步函数返回`dyn*Future`。调用者可以通过虚拟分派调用`poll`，并在准备好处置未来时调用(虚拟)Drop函数。为`Boxing<AI>`创建的vtable将分配一个框来存储未来的`ai：：Next()`，并用它来创建`dyn*Future`。其他适配器的vtable可以使用他们想要的任何策略。例如，`InlineAsyncIterator<AI>`将未来的`AI：：Next()`存储到包装器的一个字段中，获取指向该字段的原始指针，并从该原始指针创建一个`dyn*Future`。为了更好的性能，可能的未来扩展：我们修改了异步特征函数(或使用返回位置Iml特征的任何特征函数)的ABI，以允许调用者有选择地提供堆栈空间。如果堆栈空间可用，`Boxing`适配器将在可能的情况下使用它来避免装箱。这必须与一些编译器分析相结合，以确定要预先分配的堆栈空间有多少。我应该澄清一点，尽管泰勒和我已经讨论过这一点，但我不知道他对此有何感想。确切地说，我不会将其称为“提议的一部分”，而更像是我感兴趣的延期。

This lets us express virtually any pattern. Its even *possible* to express side-stacks, if the runtime provides a suitable adapter (e.g., `TokioSideStackAdapter::new(ai)`), though if side-stacks become popular I would rather consider a more standard means to expose them.

这让我们几乎可以表达任何模式。如果运行时提供合适的适配器(例如，`TokioSideStackAdapter：：new(Ai)`)，甚至可以表示侧堆栈，尽管如果侧堆栈变得流行，我宁愿考虑一种更标准的方法来公开它们。

The main downsides to this proposal are:

这项提议的主要缺点是：

* Users have to write `Boxing::new`, which is a productivity and learnability hit, but it avoids a big hit to transparency. Is that the right call? I’m still not entirely sure, though my heart increasingly says yes. It’s also something we could revisit in the future (e.g., and add a default adapter).
* If we opt to modify the ABI, we’re adding some complexity there, but in exchange for potentially quite a lot of performance. I would expect us not to do this initially, but to explore it as an extension in the future once we have more data about how important it is. 

There is one pattern that we can’t express: “have caller allocate maximal space”. This pattern *guarantees* that heap allocation is not needed; the best we can do is a heuristic that *tries* to avoid heap allocation, since we have to consider public functions on crate boundaries and the like. To offer a guarantee, the argument type needs to change from `&mut dyn AsyncIterator` (which accepts any async iterator) to something narrower. This would also support futures that escape the stack frame (see [Appendix A](#Appendix-A-futures-that-escape-the-stack-frame) below). It seems likely that these details don’t matter, and that either inline futures or heuristics would suffice, but if not, a crate like [stackfuture](https://github.com/microsoft/stackfuture) remains an option.

用户必须写`Boxing：：new`，这会影响工作效率和易学性，但它避免了对透明度的重大影响。这是正确的决定吗？我仍然不完全确定，尽管我的心越来越多地同意了。这也是我们将来可以重新考虑的事情(例如，添加默认适配器)。如果我们选择修改ABI，我们会在那里增加一些复杂性，但以潜在的相当大的性能为交换。我希望我们一开始不会这样做，但一旦我们有了更多关于它有多重要的数据，就会将其作为未来的扩展进行探索。有一种模式我们无法表达：“让调用者分配最大的空间”。该模式保证不需要堆分配；我们所能做的最多是尝试避免堆分配的启发式方法，因为我们必须考虑机箱边界上的公共函数等等。为了提供保证，参数类型需要从`&mut dyn AsyncIterator`(它接受任何异步迭代器)更改为更窄的类型。这也将支持对堆栈帧进行转义的期货(参见下面的附录A)。这些细节似乎并不重要，内联期货或启发式方法就足够了，但如果不是这样，像StackFuture这样的板条箱仍然是一个选择。

### Comments?

### 有什么评论吗？

Please leave comments in [this internals thread](https://internals.rust-lang.org/t/blog-series-dyn-async-in-traits-continues/17403). Thanks!

请在此内部帖子中留下评论。谢谢!

### Appendix A: futures that escape the stack frame

### 附录A：转义堆栈帧的期货

In all of this discussion, I’ve been assuming that the async call was followed closely by an await. But what happens if the future is not awaited, but instead is moved into the heap or other locations?

在所有这些讨论中，我一直假设异步调用之后紧跟着等待。但是，如果未来没有被等待，而是被移到堆或其他位置，会发生什么呢？

```rust
fn foo(x: &mut dyn AsyncIterator<Item = u32>) -> impl Future<Output = Option<u32>> + ‘_ {
    x.next()
}
```

For boxing, this kind of code doesn’t pose any problem at all. But if we had allocated space on the stack to store the future, examples like this would be a problem. So long as the scratch space is optional, with a fallback to boxing, this is no problem. We can do an escape analysis and avoid the use of scratch space for examples like this.

对于拳击，这种代码根本不会带来任何问题。但如果我们在堆栈上分配空间来存储未来，这样的示例将是一个问题。只要暂存空间是可选的，并后备到装箱，这就不成问题。对于这样的例子，我们可以进行转义分析，避免使用暂存空间。

### Footnotes

### 脚注