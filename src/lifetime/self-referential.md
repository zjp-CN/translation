# 自引用与生命周期

> Stackoverflow: [Why can't I store a value and a reference to that value in the same struct?](https://stackoverflow.com/a/32300133/15448980)
>
> 由 [@Shepmaster] 提供解答。

[@Shepmaster]: https://github.com/shepmaster

一个自引用 (self-referential) 数据结构的核心特点是：数据及其引用存放在一处。

## 典型错误

在 Rust 中，你通常无法编译如下自引用结构体 `Combined`：

```rust
struct Parent {
    count: u32,
}

struct Child<'a> {
    parent: &'a Parent,
}

struct Combined<'a> {
    parent: Parent,
    child: Child<'a>,
}

impl<'a> Combined<'a> {
    fn new() -> Self {
        let parent = Parent { count: 42 };
        let child = Child { parent: &parent };

        Combined { parent, child } // E0515 + E0505
    }
}
```

点击右上角运行按钮会看到两个编译错误：
* error[E0515]: cannot return value referencing local variable `parent`
* error[E0505]: cannot move out of `parent` because it is borrowed

要完全理解这里的错误，则必须思考这些值在内存中是如何表示的[^mem-layout]，以及移动这些值时会发生什么。

[^mem-layout]: 译者注：我强烈推荐一个介绍 Rust 内存布局的基础向视频
《[Visualizing memory layout of Rust's data types](https://www.youtube.com/watch?v=rDoqT-a6UFg)》，其内容是每个 Rust 程序员应该知晓的。

让我们用一些假想的内存地址来标注 `Combated::new` 内部的内存情况，这些地址表示值的位置：

```rust,ignore
let parent = Parent { count: 42 };
// `parent` 位于 0x1000 处，并占据 4 个字节，其值为 42
let child = Child { parent: &parent };
// `child` 位于 0x1010 处，并占据 4 个字节（假设在 32 位的机器上），其值为 0x1000
         
Combined { parent, child }
// 这个返回值位于 0x2000 处，并占据 8 个字节
// `parent` 被移动至 0x2000，那么 `child` 呢？
```

`child` 该怎么办？如果它的值像 `parent` 那样移动，那么它将引用不再保证包含有效值的内存，因为任何其他代码都可以值存储在内存地址 0x1000。

访问该内存时假设是一个整数，则可能会导致崩溃和/或安全错误，这是 Rust 可以防止的主要一种错误。

这正是生命周期阻止的问题。生命周期是一些元数据，它让你和编译器知道一个值在其 **<u>当前内存位置将有效多长时间</u>**。

这是一个重要的区别，因为这是 Rust 新人经常犯的错误：生命周期**不是**从一个对象被创建到被销毁之间的时间段！

打个比方，你可以这样想：在一个人的一生中，他会居住在许多不同的地方，每个地方都有不同的地址。

而 Rust 的生命周期关注的是你（指值） **现在** 居住的地址，而不是你未来什么时候会死（虽然说死亡也会改变你的地址）。

每次移动值，都会涉及不同的生命周期，因为原地址不再有效。

同样重要的几点是：
* 生命周期 **不会** 改变你的代码；
* 是 **你的代码控制生命周期**，而不是生命周期控制你的代码；
* 一句精辟的话是：“生命周期是 **描述性的**，而 *不是规束性的*”。 (lifetimes are **descriptive**, not *prescriptive*)

用一些行号来理解 `Combated::new` 中的生命周期：

```rust
{                                          // 0
    let parent = Parent { count: 42 };     // 1
    let child = Child { parent: &parent }; // 2
                                           // 3
    Combined { parent, child }             // 4
}                                          
```

`parent` 的具体生命周期是从 1 到 4，包括两端（用 `[1, 4]` 表示）。`child` 的具体生存期为 `[2, 4]`，返回值的具体生存期为`[4，5]`。

有的具体的生命周期可能从零开始，比如函数参数、在块之外的东西。

需要注意的是，`child` 本身的生命周期是 `[2, 4]`，但它引用的是一个生命周期为 `[1, 4]` 的值。只要
**引用的值在被引用的值无效之前无效**，就可以这样做。

当试图从块中返回 `child` 时，就会出现问题：这将“过度延长”其生命周期，让其超出自然长度。

先从简单的情况开始思考，对于下面的方法签名：

```rust,ignore
impl Parent {
    fn child(&self) -> Child { /* ... */ }
}
```

它使用了生存周期省略来避免写出显式的泛型生命周期参数。它等价于：

```rust,ignore
impl Parent {
    fn child<'a>(&'a self) -> Child<'a> { /* ... */ }
}
```

在这两种写法都会使用 `self` 的具体生命周期进行参数化，从而返回一个 `child` 结构体。

换句话说，`child` 实例包含对创建它的 `parent` 的引用，因此不能比 `parent` 的实例存活更长时间。

这也让我们认识到，原先的函数创建确实出了问题：

```rust,ignore
// 对一般的函数
fn make_combined<'a>() -> Combined<'a> { /* ... */ }

// 实现中的函数
impl<'a> Combined<'a> {
    fn new() -> Combined<'a> { /* ... */ }
}
```

在这两种情况都没有通过函数参数提供生命周期参数，这意味着 `Combined` 将被参数化不受任何约束的生命周期，它可以是调用者想要的任何生命期。

但这是无意义的，因为调用者可以指定 `'static`，但无法满足该条件。

## 修复代码

### 避免自引用

最简单做法是也是最推荐的解决方案:不要试图将这些东西放在同一结构中。

把数据和对它的引用分开：将原数据（非引用）的类型放在一个结构中，然后提供返回引用或包含引用的数据结构的方法。

有一种特殊情况：过度跟踪生命周期，这发生当把数据存放在 heap 上。例如，当使用 `Box<T>`
时，被移动的结构体中包含了指向堆的指针，且指针指向的值保持不变，但指针本身的地址将移动。在实践中，这并不重要，因为你总是跟随指针。

### 使用解决自引用的库

一些库提供了解决这种情况的方法，但它们要求基础的地址不能被移动。这不让改变 vec，因为可能导致堆上存放的值被重新分配和移动。

* [rental](https://crates.io/crates/rental) 不再维护
* [owning_ref](https://crates.io/crates/owning_ref)
* [ouroboros](https://crates.io/crates/ouroboros)

rental 解决自引用问题的例子（都是 stackoverflow 链接）：

* [Is there an owned version of String::chars?](https://stackoverflow.com/q/47193584/155423)
* [Returning a RWLockReadGuard independently from a method](https://stackoverflow.com/q/50496879/155423)
* [How can I return an iterator over a locked struct member in Rust?](https://stackoverflow.com/q/51664098/155423)
* [How to return a reference to a sub-value of a value that is under a mutex?](https://stackoverflow.com/q/40095383/155423)
* [How do I store a result using Serde Zero-copy deserialization of a Futures-enabled Hyper Chunk?](https://stackoverflow.com/q/43702185/155423)
* [How to store a reference without having to deal with lifetimes?](https://stackoverflow.com/q/49300618/155423)

### 使用 `Rc` 或 `Arc`

你可以把数据放入某种引用计数类型，比如 [`Rc`](https://doc.rust-lang.org/std/rc/struct.Rc.html)
或 [`Arc`](https://doc.rust-lang.org/std/sync/struct.Arc.html)。

## 自动更新引用？

在将 `parent` 移到结构体中后，为什么编译器不能把 `parent` 的新地址的引用赋给结构中的 `child`？

虽然理论上这样做是可能的，但这样做会带来大量的复杂性和开销。

每次移动对象时，编译器都需要插入代码来“修复”引用。这意味着复制结构体不再是只移动一些位的非常低廉的操作，甚至可能意味着这样的代码很昂贵，这取决于假想的优化器有多好：

```rust,ignore
let a = Object::new();
let b = a;
let c = b;
```

程序员可以通过创建仅在调用它们时才接受适当引用的方法来选择何时发生，而不是强制在每次移动时都发生这种情况。

## 一种可行但无用做法

在某种特定的情况下，你可以创建一个引用其自身的类型。不过，你需要使用类似 `Option` 的代码，并且分两步完成：

```rust
#[derive(Debug)]
struct WhatAboutThis<'a> {
    name: String,
    nickname: Option<&'a str>,
}

fn main() {
    let mut tricky = WhatAboutThis {
        name: "Annabelle".to_string(),
        nickname: None,
    };
    tricky.nickname = Some(&tricky.name[..4]);

    println!("{:?}", tricky);
}
```

从某种意义上说，这确实是可行的，但创建的值高度受制：它永远也不能被移动。尤其是，它不能从函数返回，也不能按值传递给任何对象。

以下构造函数具有与上述生命周期相同的问题：

```rust
fn creator<'a>() -> WhatAboutThis<'a> {  }
```

如果你尝试使用方法执行相同的代码，则需要诱人但最终无用的 `&'a self`，此时，代码将受到更多限制，并且在第一次方法调用后收到借用错误：

```rust,editable
#[derive(Debug)]
struct WhatAboutThis<'a> {
    name: String,
    nickname: Option<&'a str>,
}

impl<'a> WhatAboutThis<'a> {
    fn tie_the_knot(&'a mut self) {
       self.nickname = Some(&self.name[..4]); 
    }
}

fn main() {
    let mut tricky = WhatAboutThis {
        name: "Annabelle".to_string(),
        nickname: None,
    };
    tricky.tie_the_knot();

    // cannot borrow `tricky` as immutable because it is also borrowed as mutable
    // println!("{:?}", tricky);
}
```

另见：

* [Cannot borrow as mutable more than once at a time in one code - but can in another very similar](https://stackoverflow.com/q/31067031/155423)

## `Pin` 怎么样？

[`Pin`](https://doc.rust-lang.org/std/pin/struct.Pin.html) 稳定于 Rust 1.33，在[模块文档](https://doc.rust-lang.org/std/pin/index.html)中有这样的内容：

> 这种情况的一个主要示例是创建自引用结构体，因为移动带有指向自身的指针的对象将使指针无效，这可能会导致未定义的行为。

需要注意的是，“自引用”并不一定意味着使用引用。实际上，自引用结构的[示例](https://doc.rust-lang.org/std/pin/index.html#example-self-referential-struct)特别说明：

> 我们不能用普通引用告知编译器这一点，因为这种模式不能用通常的借用规则来描述。
>
> 相反，我们使用一个原始指针，因为我们知道该指针不为空，且指向字符串。

从 Rust 1.0 开始，就存在为此行为使用裸指针的功能。实际上，owning-ref 和 rental 就是使用裸指针。

添加 `Pin` 的唯一一件事是声明它是一种常见的方式来保证给定的值不会被移动。

另见：

* [How to use the Pin struct with self-referential structures?](https://stackoverflow.com/q/49860149/155423)
