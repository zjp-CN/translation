# 异步析构函数、异步通用性和保证完成的 Future

> src: 《 [Async destructors, async genericity and completion futures](https://sabrinajewson.org/blog/async-drop) 》
>
> by Sabrina Jewson in 2022-03-24


本文的主要重点是尝试设计一个系统来支持 Rust 编程语言中的异步析构函数，弄清楚它们的确切语义并解决在此过程中遇到的任何问题。

副作用是，它还设计了一种称为“异步通用性” (async genericity) 的语言功能，该功能可以使用相同的代码库支持阻塞和异步代码，并设计一个把保证完成的
Futures (completion-guaranteed Futures) 添加到该语言中的系统。

## 什么需要异步析构函数？

异步析构函数 (async drop) 在较高级别上允许类型在被 drop 时运行其中包含 `.await` 的代码。

这使得用于清理的代码实际上能执行 I/O，从而在正确清理资源的程度上提供了更大的自由度。

一个值得注意的用例是实施 TLS 协议，其中：

> 每方在关闭连接的写端时，**必须**发送一个“通知关闭” (close_notify) 的警报，除非它已经发送了某种错误警报。
>
> src: [RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446#section-6.1)

为了确保始终满足此要求，TLS 实现应该能够在 drop `TlsStream` 类型时发送此警报 —— 如果所有 I/O 都是异步完成的，则需要异步析构函数。

目前，这种清理通常由 [`poll_shutdown`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html#tymethod.poll_shutdown) 和
[`poll_close`](https://docs.rs/futures-io/0.3/futures_io/trait.AsyncWrite.html#tymethod.poll_close) 
等方法管理：如果使用者希望干净地清理该类型，则可以由调用者选择调用异步函数。然而，这种方法有几个局限性：

* 没有办法静态地保证该方法不会被调用两次，这取决于使用者。
* 根本没有办法静态地保证该方法被调用 —— 它很容易被忘记。
* 在每个值的生命周期结束时调用它是很麻烦的样板，并且理想情况下是没有必要的。
* 它实际仅适用于实现 `AsyncWrite` 的类型。如果您的类型实际上不是字节流，那就太糟糕了。

显然我们需要一个比这更好的解决方案。因此，让我们看一些实际示例，看看我们需要哪些功能来改善这种情况。

## Future 取消后的 async drop

让我们从这个简单的函数开始：

```rust,ignore
async fn wait_then_drop_stream(_stream: TlsStream) {
    time::sleep(Duration::from_secs(10)).await;
} 
```

它是一个异步函数，它获取 `TlsStream` 的所有权，休眠 10 秒，然后在最后隐式删除它。我们想要这个函数最明显的特点是，
TLS 流应该在 10 秒后执行正常的 `close_notify`。

然而，还有一个些许微妙但同样重要的一点：因为在 Rust 中，每个 Future 都可以在 `.await` 点隐式地取消，所以如果
Future 被取消，同样的优雅关闭也应该发生。例如，假设该函数的使用方式如下：

```rust,ignore
let handle = task::spawn(wait_then_drop_stream(some_tls_stream));
time::sleep(Duration::from_secs(5)).await;
handle.cancel(); 
```

仅仅因为我们取消了整个任务，并不意味着我们突然想要绕开定期的正常关闭并让 TLS 流以未知的方式完成 —— 事实上，我们几乎从不想这样。

因此，我们需要一种方法来注册 Future 被取消后发生的异步操作，以支持在那里运行优雅的关闭代码。我们该怎么做呢？

事实证明，使用该语言中的异步析构函数，那会变得非常容易：由于取消 Future 是通过调用其析构函数，向 Future 发出信号的，因此 Future 
本身可以简单地拥有一个异步析构函数并在其中运行清理代码。

其精确语义的工作方式与当今同步 drop 的工作方式非常相似：以相反的顺序删除每个局部变量（关键在于包括 `_stream` 变量）。

我们必须回答的第二个问题是，当异步析构函数 _本身_ 被取消时会发生什么 —— 例如，您可能正在删除 
TLS 流，但同时您的任务突然终止。为了演示这个问题，看一下这个函数：

```rust,ignore
async fn assign_stream(target: &mut TlsStream, source: TlsStream) {
    *target = source; // Async destructor is implicitly called!
    println!("1");
    async { println!("2") }.await;
    println!("3");
    yield_now().await;
    println!("4");
} 
```

它将`source` TLS 流分配给 `target` TLS 流（在此过程中删除旧的 `source` ），然后打印出数字 1 到 4。

在正常情况下，此任务只会从上到下运行并始终打印出每个数字；但当涉及取消时，事情就变得更加复杂。

如果在将 `source` 分配给 `target` 期间发生取消，则该语言现在必须决定如何处理其余代码 - 是否应该运行到最后？应该立即退出吗？它应该只运行其中的_一部分_吗？

这里值得讨论的选项主要有三类：“立即终止”设计、“永不终止”设计和“延迟终止”设计。每一种都有优点和缺点，下面将详细探讨。

### “立即终止”设计

在这些设计下，上面代码中的四个打印都不能保证运行。

如果分配被终止，它将尽快退出 Future，同时执行最少量的清理（即只运行析构函数而不执行其他操作）。

此设计有三种变体，在需要指定 `.await` 时略有不同：

1.  有时 `.await`：在此设计下， `=` 永远不需要 `.await` ，而异步函数调用则始终需要 `.await`。

    这基本上保持了原来的样子：没有引入特殊的新语法，也没有进行重大的破坏性更改。
    
    为了感受一下它的样子，以下是一个使用它实现的重要的“现实世界”异步函数：
    
    ```rust,ignore
    async fn handle_stream(mut stream: TlsStream) -> Result<()> {
        loop {
            match read_message(&mut stream).await? {
                Message::Redirect(address) => {
                    stream = connect(address).await?;
                    // The below line isn't guaranteed to run even if
                    // redirection succeded, since the future could be
                    // cancelled during the drop of the old `TlsStream`.
                    log::info!("Redirected");
                }
                Message::Exit => break, 
            }
        }
    } 
    ```

    它确实引入了一个脚枪，因为控制流在哪些点可以退出函数将不再明显。它也可以被认为是不一致的，因为某些挂起点需要
    `.await` 而其他挂起点则不需要，尽管事实上这两种挂起点之间没有有意义的语义差异。
    
2.  从不 `.await`：为了解决这种不一致问题，此设计完全删除了 `.await` ，使所有取消点完全不可见。调整我们之前的示例，它看起来像：
    
    ```rust,ignore
    async fn handle_stream(mut stream: TlsStream) -> Result<()> {
        loop {
            match read_message(&mut stream)? {
                Message::Redirect(address) => {
                    stream = connect(address)?;
                    log::info!("Redirected");
                }
                Message::Exit => break,
            }
        }
    }
    ```
    
    除了删除 `.await` 的技术问题（它是递归完成的吗？它是否使实现 `Future` 成为一个重大改变？异步块是否变得多余？等等）和向后兼容性
    / 使用者流失问题，这与之前的脚枪一样，但结果变得极端 —— 现在基本上不可能仔细管理取消可能发生的位置，并且大多数使用者最终不得不将取消更多地视为
    `pthread_kill` 而不是有用的控制流构造。
    
3.  始终 `.await`：另一方面，这种设计使得 `.await` 在任何地方都是强制性的。使用异步析构函数对值进行赋值必须使用新的
    `=.await` 运算符而不是普通的 `=` 来完成，并且值不能隐式超出范围，而必须由使用者显式调用 `drop` 。再次回到`handle_stream`示例：
    
    ```rust,ignore
    async fn handle_stream(mut stream: TlsStream) -> Result<()> {
        loop {
            match read_message(&mut stream).await? {
                Message::Redirect(address) => {
                    stream =.await /* 👈 */ connect(address).await?;
                    log::info!("Redirected");
                }
                Message::Exit => break,
            }
        }
        drop(stream).await; // 👈
    }
    ```
    
    这是这三个选项中唯一可以明确避免“隐式取消”的选项，但它仍然不理想，因为它最终会引入新的奇怪的语法，并使编写异步代码变得相当冗长。
    

这三种方案最终都有相当明显的缺点 —— 从根本上来说，它与当前的异步语法和模型非常不兼容。

因此，如果支持终止是如此棘手，那么我们是否可以通过完全避免它来回避这个问题呢？

### “永不终止”设计

这种设计完全消除了语言中的隐式取消。Future 将与同步函数非常相似，从上到下线性运行，而不会导致调用者提前退出（当然，panics 仍然会导致提前退出发生）。

这意味着在本节开头所示的 `assign_stream` 函数中，所有 `1` 、`2` 、`3` _和_ `4` 都保证会打印，因为代码执行在任何时候都不允许停止。

如果您想了解更多相关信息， [Carl Lerche 之前已经提出过这种方法](https://carllerche.netlify.app/2021/06/17/six-ways-to-make-async-rust-easier/)。

与“立即终止”非常相似，它具有三个子设计：“始终 `.await`”、“有时 `.await`”和“从不 `.await`”，具体取决于 `.await`被认为是必要的情况。

那里列出的许多相同的论点都适用，尽管不再存在由隐式潜在取消点引起的脚枪问题，因此这主要是权衡一致性、破坏和新语法的问题。

这是另一种高度一致的方法，但它的主要缺点是丢弃了非常有用的工具，即隐式取消上下文。

虽然取消绝对可以作为库功能实现（请参阅 [`CancellationToken`](https://docs.rs/tokio-util/0.7/tokio_util/sync/struct.CancellationToken.html)
和 [`StopToken`](https://docs.rs/stop-token/0.7/stop_token/struct.StopToken.html)
），并且我希望它成为需要它的用例的一个选项，但大多数时候拥有隐式上下文要有用得多，因为它更简洁，需要使用的样板代码也更少。

我不想看到原本绝对可靠的函数变得容易出错，或者需要付出巨大的迁移努力来向每个函数添加取消 Token 参数。

Carl Lerche 的一个论点是在一个示例代码片段中， 取消 Future 与 `select!` 相结合其实是一种脚枪。

但正如 Yoshua Wuyts 在 [Futures Concurrency III](https://blog.yoshuawuyts.com/futures-concurrency-3/)
中指出的那样，此类代码中的主要问题是 `select!` 混乱的语义而不是取消 Future 的行为。

最终，我认为取消的问题不足以保证将其从语言中删除。尽管这种方法的一致性及其与阻塞代码的并行性很好，但取消仍然有用，并且有一些方法可以将其与不引入脚枪的异步析构函数结合起来。

注意，即使使用其他选项，如果需要这样的语义，向语言中添加异步析构函数将使创建一个以“无取消”模式执行 Future 的组合器是容易的
—— 有关更多信息，请参阅 [附录 D](#uncancellable-futures)。

### “延迟终止”设计

与前两种设计不同，这些方法试图完全接受分配和超出范围（不需要 `.await` ）和调用 async 函数（需要 `.await`）之间的语法差异。

当调用者尝试在前一个操作期间取消 Future 时，Future 实际上会在之后继续运行一小段时间，直到它能够到达之后一个操作并正确退出。

这立即解决了困扰“立即终止”设计的主要问题，而无需走向永不终止的极端：没有脚枪，因为取消点永远不会隐式引入，没有添加新语法，也没有重大的破坏性，并且现在有一个明确的原因
_为什么_ `=` 不需要 `.await` 但调用函数需要。

然而，它并不完美。它有效地引入了两种不同类型的挂起点，它们的行为截然不同，“立即终止”和“从不终止”设计不存在这种不一致。

此外，这意味着如果您围绕 `=` 运算符调用包装函数或手动调用 `drop`，它与使用内置语言行为的语义略有不同，因为它改变了挂起点的类型。

对于大多数使用者来说，这可能是意料之外且不直观的。

此设计有三种方式，具体取决于代码何时停止运行：

1. 在第一次 `.await` 之前终止：`=` 发生取消等操作后，代码将继续运行，直到下一个 `.await` 发生的点，此时外部 Future 将立即退出，甚至无需轮询一次内部
   Future。在 `assign_stream` 示例中，这意味着保证打印 `1` ，但不打印之后的所有内容。

2. 在第一次 `.await` 之后终止：与前一个一样，但 Future 将被轮询一次（只会丢弃其结果并退出外部 Future）。在我们的示例中，这意味着保证打印
   `1` 和 `2`，但不能打印除此之外的任何内容。

3. 第一次挂起时终止：外部 Future 将在第一次调用 `.await` 被轮询、返回 `Poll::Pending` 的时候终止。在示例代码中，这将强制打印所有 `1` 、`2` 和 `3` ，但不打印
   `4` 因为 `yield_now()` 会导致出现挂起点。这与今天取消 Future 的工作方式最相似，因为如果没有挂起点，目前似乎无法发生取消（对于上述建议，取消仍然不能发生，但它似乎因为
   `async {}.await` 而可能退出控制流）。从 Future 的角度来看，这就像调用者刚刚 `.await` 然后稍后尝试取消一样。

尽管它们看起来非常相似，但前两种方法进行了极其微妙但非常重要的范式转变： `.await` 的含义从“可能挂起” (might suspend) 
运算符更改为“可能停止” (might halt) 或“可能终止” (might abort) 运算符，因为 `async {}.await;`
现在能够导致计算突然停止。

这是一个很小的差异，但最终会带来很大的问题，因为我们现在必须回答一大堆新问题：

* 如果 `.await` 只是关于取消，我们是否应该允许省略它来调用异步函数，同时禁止取消？
* 是否应该允许使用 `.await` 调用 _同步_ 函数以在它们周围引入取消点？
* 是否应该引入普通的 `await;` 语句来引入这些取消点，相当于 `async {}.await;` 吗？

换句话说，这会变成如下表格，空的格子存在明显的漏洞：

|                      | 调用者无法取消 | 调用者可以取消 |
|:--------------------:|:--------------:|:--------------:|
| **被调用者无法取消** |     `foo()`    |        ?       |
| **被调用者可以取消** |        ?       |  `foo().await` |

我认为这不是我们想要的情况。第三种方法通过将终止机会与暂停点联系起来，消除了对该表中第二列的需要，从而消除了这些漏洞，从而完全避免了整个情况。

此外，第三种方式并不是重大更改，因为不必条以前那些依赖无法终止的 `async` 操作的立即完成的代码。

从技术上讲，无论哪种方式它仍然不会破坏，因为现有代码不使用异步析构函数，但它允许程序员保持他们的心理模型，这也很重要。

由于所有这些原因，我赞成使用在第一次挂起点终止 (abort-at-first-suspend) 的延迟设计：它需要很少的迁移工作，避免了脚枪，而且我认为这对使用者来说并不太令人惊讶。

本文的其余部分将在假设选择了这种设计。

## 同步函数中的 async drop

也许任何异步 drop 设计必须面对的最困难的问题是，当具有异步析构函数的类型在同步上下文中被删除时会发生什么。

考虑这段代码：

```rust,ignore
fn sync_drop_stream(_stream: TlsStream)  {} 
```

该同步函数采用 TLS 流作为参数。它必须对给定的流执行某些操作，因为它拥有所有权，并且没有返回值将其传递回调用者，但它不能使用常规的异步 drop，因为它是同步函数。

那么它能做什么呢？在 [withoutboats 关于这个主题的博客文章](https://without.boats/blog/poll-drop/#the-non-async-drop-problem) 中，假设了两种选择：

> 1. 像所有其他类型一样，调用非异步析构函数。
> 2. 向运行时引入某种执行器（可能只是 `block_on` ）以作为 drop glue 的一部分进行调用。

对我来说，这两种解决方案似乎都很糟糕。对于方案 2， 由于 Boats 所概述的原因，显然是行不通的，但我认为方案 1 远比它看起来的更像是一把脚枪。

标准库中的许多函数本质上都是禁区，因此您不仅无法在编写良好的代码中获得它们的人体工程学，而且在 TLS 流上简单地调用
[`Option::insert`](https://doc.rust-lang.org/stable/std/option/enum.Option.html#method.insert) 之类的函数将很容易创建充满错误的代码。

我的替代解决方案是完全禁止该代码编译。对于要在同步上下文中 drop 的类型，它必须实现某个 trait，而 `TlsStream` 和类似类型不会实现这个 trait。

因此，除非在 `TlsStream` 上显式使用 `close_unclean` 方法，否则完全不可能从任何地方导致不干净的 TLS 关闭，从而消除这一类错误。

这种方法并非没有困难 ） 事实上，它比其他方法有更多的困难，本文的大部分内容将只是致力于解决这些问题。

但最终，我确实相信为了更强大的静态保证，这是一个更好的解决方案。

## panic 检查

我提议在编译时禁止在同步上下文中异步 drop 的类型。看起来很容易吧？

编译器只需检测何时为每个值运行析构函数，如果无效则抛出错误。

```rust,ignore
// Error
fn bad(stream: TlsStream) {
    println!("{:?}", stream.protocol_version());
}

// OK
fn good(stream: TlsStream) -> TlsStream {
    println!("{:?}", stream.protocol_version());
    stream
} 
```

但事情没那么简单。

因为在程序中的几乎每个点，线程都可能出现 panic，如果发生这种情况，可能会开始发生展开 (unwinding)，如果发生 _这种_ 
情况，您需要删除作用域中的所有局部变量，但只有当它们具有同步析构函数！

因此，编译器真的必须在同步上下文中禁止使用异步析构函数的 _任何_ 值，因为 panic 总是会发生并把事情搞砸。

```rust,ignore
// Error
fn bad(stream: TlsStream) -> TlsStream { stream } 
```

但这也行不通。在许多情况下，在同步上下文中使用具有异步析构函数的类型是绝对必要的，例如 `TlsStream::close_unclean` 
函数需要 `self` 作为参数或 `block_on` 函数需要 Future 作为参数。

编译器实际需要强制执行的内容可以稍微宽松一些：无法同步 drop 的值保留在范围内，但不会发生可能发生 panic 的操作。

这里的“可能发生 panic 的操作”包括调用任何函数或触发任何运算符重载。它只是不包括简单的事情，例如构造结构体或元组、访问类型的字段（不重载
`Deref` ）、模式匹配、返回或任何其他内置的简单操作。

```rust,ignore
// Error
fn bad(stream: TlsStream) -> TlsStream {
    println!("{:?}", stream.protocol_version());
    stream
}

// OK
fn good(stream: TlsStream) -> TlsStream { stream } 
```

该规则相当受限，但实际上提供了处理这种情况所需的所有工具。

与 `ManuallyDrop` 结合使用时特别有效：因为 `ManuallyDrop` 会跳过运行类型的析构函数，因此即使内部类型没有同步 drop，它也始终能够同步
drop。因此，只要在获取第一个可能出现 panic 操作的值上对其调用 `ManuallyDrop::new`，编译器将允许执行您喜欢的任何操作，因为 drop
该值的责任实际上已转移给 _您_，如果你想 drop 这个值的话。

更重要的是，`ManuallyDrop::new` 本身不必使用任何编译器魔法来实现，因为它所做的只是执行一个结构表达式并返回它，它可以很好地通过 panic 检查。

## 异步中的 unwinding

我们已经了解了同步上下文中的展开是什么样子，接下来让我们看看异步上下文中的展开是什么样子的。

它应该更容易，因为这次我们实际上可以等待每个值的析构函数。

```rust
async fn unwinds(_stream: TlsStream) {
    panic!();
} 
```

基于完全禁止不优雅关闭 TLS 流的原则，Future 将捕获这种 panic，然后像通常那样异步 drop 范围内的所有内容，然后最终将 panic 传播给调用者是有意义的。

与同步代码一样，在执行这些异步 drop 时，[`std::thread::panicking`](https://doc.rust-lang.org/stable/std/thread/fn.panicking.html)
将返回 `true` ，并且类似的，再次 panic 将导致终止。

实际上，在 Future 中存储正在进行的 panic 很容易：只需选择性地存储一个指针，即由 `catch_unwind` 返回的 `Box<dyn Any + Send>`，准备稍后传递给 `resume_unwind`。

不幸的是，这些函数 [在 `no_std` 环境中尚不可用](https://github.com/rust-lang/rfcs/issues/2810)，因此目前编译器可能必须使用诸如终止或泄漏值之类的解决方法，或者可能在
`#![no_std]` 上完全禁止实现异步析构函数。如果这个问题得到解决，就有可能将处理 panic 改进为更有用的东西。

然而，这种方法有一个大问题，那就是展开安全 (unwind safety)。

展开安全是指代码中的 panic 会导致共享数据结构进入逻辑上无效的状态，因此每当您有机会在 panic 后观察世界时，都应该检查您是否知道这种情况可能会发生。

这是由两个 trait [`UnwindSafe`](https://doc.rust-lang.org/stable/std/panic/trait.UnwindSafe.html) 和
[`RefUnwindSafe`](https://doc.rust-lang.org/stable/std/panic/trait.RefUnwindSafe.html) 控制的，它们提供了必要的基础设施来在编译时进行这些检查。

实施起来很简单，但这个提案就会打破这个概念：

```rust,ignore
#[derive(Clone, Copy, PartialEq, Eq)]
enum State { Valid, Invalid }

let state = Cell::new(State::Valid);

let task = pin!(async {
    let stream = some_tls_stream;
    state.set(State::Invalid);
    panic!();
    state.set(State::Valid);
});
let _ = task.poll(&mut cx);

// Now the task is panicking and polling the TLS stream...

// But we can observe the invalid state!
assert_eq!(state.get(), State::Invalid); 
```

那么我们该怎么办呢？好吧，我们有几个选择：

1. 要求异步上下文中的所有局部变量都是 `UnwindSafe` 。这将阻止上述代码编译，因为 `&Cell<T>` 是 `!UnwindSafe`。
2. 让编译器生成的 `async {}` 类型仅在 `Self: UnwindSafe` 时实现 `Future` 。这与第一个选项基本相同，只是在编译过程后期导致错误。
3. 完全忽略展开安全，它已经有点无用了，因为 `std::thread::spawn` 不需要 `F: UnwindSafe` 并且它已经可以用来见证损坏的不变量。整个系统绝对是
   `std` 中更令人困惑和更难理解的部分之一，它通常相当于对所有内容都使用 `AssertUnwindSafe`，直到 rustc 满意为止，而实际上并没有考虑其含义。
4. 异步 panic 总是会导致局部变量运行同步 drop。这将强制在类型上选择使用同步 drop，而这种类型甚至可能没有逻辑意义，并且处理异步 panic 将永久地以次优方式完成。

就我个人而言，我非常赞成选项 3 —— 完全忽略展开安全。

我想不出它什么时候实际上对我有用或防止了错误，但当然您的经验可能会有所不同（[我知道 `rust-analyzer` 已被 unwind safety 至少保护过一次](https://github.com/rust-lang/chalk/issues/260)）。

我也对选项 1 持开放态度，尽管它最终可能会很痛苦。

## `poll_drop_ready`

在现已关闭的 [RFC 2958](https://github.com/rust-lang/rfcs/pull/2958) 中，withoutboats 提出了以下用于实现异步析构函数的设计：

```rust,ignore
trait Drop {
    fn drop(&mut self);
    
    fn poll_drop_ready(&mut self, cx: &mut Context<'_>) -> Poll<()> {
        Poll::Ready(())
    }
} 
```

在这种设计下，drop 一个类型将是一件简单的事情，只需在 Future `poll` 函数内转发给 `poll_drop_ready`，直到它返回 `Poll::Ready(())` 并且可以继续执行。

类型需要在类型本身内部保存用于运行析构函数所需的所有状态。

但这种设计有一个 _主要_ 缺点，我到目前为止还没有有人提到：它破坏了 `Vec` 的三指针大小的布局保证。

问题是 `Vec` 在被破坏时需要按顺序删除它的每个元素。因此，使用像 `poll_drop_ready` 这样的方法，它需要跟踪到目前为止在 `Vec` 
自身内部已经销毁了多少元素，因为在销毁期间不允许引入任何新的外部状态。它不能使用任何现有字段来执行此操作，`ptr` 、`len` 和 `capacity`
都必须保留， 因此唯一的其他选择是添加新字段，但 [Rust 已经保证](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#guarantees) `Vec` 永远不会这样做。

这并不是说没有潜在的解决方案，例如将 `Vec` 的异步放置代码硬编码到语言中，或者仅将其设置为四个 `usize` 大小用于异步 drop 类型。但这两种方式都是不正式的手段，在我看来，这只勉强缓解设计中一个基本的问题。

那么我们如何避免这种情况呢？好吧，我们必须允许类型在其异步析构函数中保存 **新** 状态。

这样的设计 [被 withoutboats 拒绝了](https://without.boats/blog/poll-drop/#the-destructor-state-problem)，原因有两点：

1. 由此产生的 Future 可能是意想不到的 `!Send`。
2. 它不能很好地处理 trait objects。

我认为第一个问题没有特别糟糕，即使类型的异步析构函数最终成为 `!Send` ，它只是构成该类型的公共 API 的一部分，类似于类型本身的
`Send` 方式。在通用上下文中，由于 `Send` 实现无论如何都会泄漏，所以析构函数的 `Send` 性质也可能会泄漏：如果使用者希望最终的结果是
`Send` 则需要由使用者提供带有 `Send` 析构函数的类型。

Trait 对象肯定会带来更大的挑战：由于新状态的大小是可变的，因此不可能像我们通常使用非类型擦除类型那样将其栈分配到任何地方。

但这不是一个需要立即解决的问题：暂时可以禁止具有异步析构函数的 `dyn` trait 对象，并可能在以后填补这个空白。 

由于使用者始终可以为此功能创建使用者空间的解决方法，因此并不需要紧急地立即尝试稳定解决方案。

此外，因为这是所有异步 traits 共有的问题，而不仅仅在异步析构函数上，如果为这些 traits 找到通用解决方案，它最终也会解决这个问题。

<div id="function-implicit-bounds"></div>

## 函数的 implicit bounds

现在我们需要开始考虑异步 drop 如何在通用代码中工作。特别是，泛型参数何时会强制要求类型支持或不支持同步 drop？

在当前版次中，保持向后兼容性至关重要。因此，我们不能突然强制在任何现有的同步或异步的函数或实现上要求 `T: ?Drop`，因为它们很可能依赖于同步 drop。

如果 API 完全支持异步 drop，那么必须明确选择添加它（[稍后会详细介绍](#relaxed-drop-bounds)）。

没有选择添加它的所有通用参数和关联类型将默认在每个上下文中需要同步 drop。

为了说明这是如何工作的，下面是使用隐式约束注释的 `Option` 的 `FromIterator` 实现：

```rust,ignore
impl<A, V> FromIterator<Option<A>> for Option<V>
where
  // A: Drop,
  // V: Drop,
  V: FromIterator<A>,
{
  fn from_iter<I>(iter: I) -> Self
  where
    I: IntoIterator<Item = Option<A>>,
    // I: Drop,
    // No `I::IntoIter: Drop` bound is implied here since
    // that's provided by the IntoIterator trait already.
  {
    iter.into_iter().scan((), |_, item| item).collect()
  }
}
```

旁注：我使用 `T: Drop` 语法来表示“T 支持同步 drop”。

不幸的是，这与`T: Drop`当前的含义 _不符_，也不表示“类型 [`needs_drop`](https://doc.rust-lang.org/stable/std/mem/fn.needs_drop.html)
”；相反，只有当存在类型 T 的实际的 `impl Drop` 块时才满足，这使得 trait 绑定在任何实际代码中完全无用。但现在让我们忽略这一点并假设更合理的含义。

在考虑下一个版次时，我们会获得更多的自由，并且可以开始将这些限制的默认值放宽到更常用的东西。只要标准库提供了一组足够的实用程序来处理异步删除类型，迁移就应该是轻松的。

让我们看几个简单的例子来尝试找出这些默认值实际上应该是什么。

```rust,ignore
fn sync_drops_a_value<T>(v: T)  {}
fn sync_takes_a_ref<T>(v: &T)  {}
fn sync_drops_a_clone<T: Clone>(v: &T)  { v.clone(); }
async fn async_drops_a_value<T>(v: T)  {} 
```

`sync_drops_a_value` 和 `sync_drops_a_clone` 可能应该按原样编译，并且不适用于异步 drop 类型。

类似地， `async_drops_a_value` 显然应该与异步 drop 类型一起使用，因为异步上下文中当然会支持异步析构函数。

乍一看， `sync_takes_a_ref` 似乎也能容易处理，毕竟，它并没有试图 drop 任何东西，但实际上它不能，因为编译器不应该检查它的函数体来确定它是否真的做了类似
`sync_drops_a_clone` 的事情。虽然这种情况很不幸，但也不全是坏事，因为事实证明，在大多数情况下，额外的限制并不重要，因为使用者通常可以添加对该类型的额外引用来弥补缺失。

```rust,ignore
fn takes_a_ref<T /* implied to require not-async-drop */>(val: &T) { /* ... */ }

let stream: TlsStream = /* ... */;
takes_a_ref(&stream); // doesn't work, since TlsStream is async-drop
takes_a_ref(&&stream); // does work, since &TlsStream is not async-drop
```

通常，双引用的功能与单引用完全等效，因此这不应该是一个太大的问题。随着旧 API 逐渐迁移到新语法，它变得越来越不统一。

因此，在下一版次之后，所有同步函数都将通过 `T: Drop` 隐式绑定每个通用参数，并且所有异步函数将使用异步 drop 绑定。

虽然这并不能 100% 覆盖所需的行为，但它覆盖了大多数情况，这就是默认所需的全部内容 —— 可以在任何需要的地方使用显式约束。

Inherent 函数遵循大致相同的想法。考虑这个例子：

```rust,ignore
struct Wrapper<T>(T);

impl<T> Wrapper<T> {
    fn some_sync_method(self) {}
    fn ref_method(&self) {}
    async fn some_async_method(self) {}
}
```

当所有隐式约束变成显式时，它看起来像这样：

```rust,ignore
struct Wrapper<T>(T);

impl<T> Wrapper<T> {
    fn some_sync_method(self) where T: Drop {}
    fn ref_method(&self) where T: Drop {}
    async fn some_async_method(self)  where T: AsyncDrop {}
} 
```

不过，还有一个小小的补充：由于经常需要定义多个不关心 drop 的同步方法，因此可以在 `impl`
块上指定宽松的边界，并将其应用于其中的每个函数。这对于定义许多 `Option` 方法很有用：

```rust,ignore
impl<T: ?Drop> Option<T> {
    pub fn is_some(&self) -> bool { /* ... */ }
    pub fn is_none(&self) -> bool { /* ... */ }
    pub fn as_ref(&self) -> Option<&T> { /* ... */ }
    pub fn as_mut(&mut self) -> Option<&mut T> { /* ... */ }
    // et cetera
}
```


稍后将详细讨论具体语法的选择。

<div id="drop-supertrait"></div>

## Drop supertrait

今天以下代码编译：

```rust,ignore
pub trait Foo {
    fn consumes_self(self)  {}
} 
```

如果任何声明的 trait 并不意味着 `Drop` 作为 supertrait，那么我们将进行重大更改，因为不再保证 `self` 可以像这样被删除。

最终，我想遵循 `Sized` 的路径，并且 _永远不会_ 暗示 `Foo: Drop` ，以便上面的代码需要显式的 `where Self: Drop` 绑定，但在那之前该代码必须像这样脱糖：

```rust,ignore
pub trait Foo: Drop {
    fn consumes_self(self) {}
} 
```

一切都可以再次编译。

我们也有可能在当前版次中引入一些更复杂的规则，例如“只有在有任何默认方法时才隐含 Drop supertrait”；但它们只会在少数情况下有所帮助，而且说服使用者使用下一版次会更容易。

<div id="async-genericity"></div>

## 异步通用性

仅就目前的建议而言，虽然将支持异步 drop，但会相当不方便，因为现有的标准库 API 几乎没有会支持它。

只是为了展示使用起来有多么困难，这里有一些不适用于异步 drop 类型的函数：

* `Option::insert`：因为它可以 drop `Option` 中的旧值。
* 许多 `HashMap` 函数：`insert` 、`entry` 等，因为它们调用使用者提供的泛型方法，而这些方法总是会出现 panic。
* `Vec::push`：因为它是同步的，如果 `Vec` 的长度超过 `isize::MAX`，可能会出现 panic。
* `Box::new`：因为分配可能会出现 panic。

一种可能的选择是引入每个同步函数的异步版本。在处理异步 drop 类型时，您可以调用 `vec.push_async(item).await;` 而不是
`vec.push(item);`，以及 `Box::new_async(value).await` 而不是 `Box::new(value)`。

然而，这将使标准库的 API 面积增加近一倍，并导致大量代码重复。这显然是不可取的，那么我们该怎么办呢？

一个潜在的前进方向是称为异步重载 (async overloading) 的功能，[先前由 Yoshua Wuyts 提出](https://blog.yoshuawuyts.com/async-overloading/)。

这个想法是同步函数可以被异步函数重载，允许 `Vec::push_async` 和 `Vec::push` 有效地共享相同的命名空间，并根据上下文选择正确的函数。

虽然这确实相当巧妙地解决了双重 API 表面的第一个问题，但它并没有解决代码重复的第二个问题：人们仍然需要为同一算法的异步和同步实现编写两份几乎相同的代码副本。

它也有自己的问题，例如需要一种好方法来强制从多种可能性中选择一个特定的重载。

我的另一种想法是我将其称为异步通用性 (async genericity)。

与具有不同主体的两个独立函数的异步重载不同，在异步通用性下，一个函数的异步和同步等效项共享一个适用于两者的主体。
然后，编译器可以将其单态化为两个单独的函数，就像对泛型参数所做的那样。将根据给定通用参数实现的 trait 在调用站点选择正确的版本。在某种程度上，它是无颜色的异步。

## 来自 `const` 的灵感

我想从 [`const fn` 的工作中](https://github.com/rust-lang/rust/issues/67792) 获得灵感，它面临着与我们现在面临的类似问题：

如何编写一个适用于多种模式（异步/同步、常量/非常量）的函数？

一个简单的例子是 `drop`：

```rust,ignore
const fn drop<T: ~const Drop>(_x: T) {} 
```

该函数可以被视为“扩展”成两个单独的函数：

```rust,ignore
const fn drop_const<T: const Drop>(_x: T)  {}
fn drop_non_const<T>(_x: T)  {} 
```

根据 `T` 是否可以在 `const` 上下文中被 drop，在调用站点编译器将选择正确的一个。 

`const Drop` 是编译器生成的 `Drop` trait，它具有与 `Drop` 相同的方法，但转换为 `const fn`。

这个 `const` 修饰符实际上可以应用于任何 trait 以自动使其成为 `const`，比如 `const Iterator` 、`const Add` 等等。

您可以在 [其 pre-RFC](https://internals.rust-lang.org/t/pre-rfc-revamped-const-trait-impl-aka-rfc-2632/15192) 中阅读更多相关信息，我不会在这里详细介绍。

我将使用它作为异步泛型设计的起点。它可能看起来像这样：

```rust,ignore
~async fn drop<T>(_x: T)  {} 
```

`T: ~async Drop` 约束是隐含的，就像 `T: async Drop` 在普通 `async fn` 中隐含的方式一样。它“扩展”成：

```rust,ignore
async fn drop_async<T>(_x: T)  {}
fn drop_sync<T>(_x: T)  {} 
```

如果有多个通用参数，例如：

```rust,ignore
~async fn drop_pair<A, B>(_: A, _: B)  {} 
```

仅当 _所有_ 参数都实现 trait 的同步版本时，同步版本才是可能的。

```rust,ignore
// `A: async Drop, B: async Drop`
async fn drop_pair_async<A, B>(_: A, _: B) {}

// `A: Drop, B: Drop`
fn drop_pair_sync<A, B>(_: A, _: B) {}
```

如果在 `A: Drop` 但 `B: async Drop` 处调用该函数，则将选择异步版本，因为 `A: Drop` 已暗示 `A: async Drop`。

如果 `~async fn` 声明时 _没有_ 带有 `~async` 绑定的泛型参数，那么它实际上完全等同于同步函数，并且可能应该被 rustc 警告。

需要注意的一个重要方面是 `async` 在某种程度上与 `const` 相反。

虽然非 `const` 函数始终可以替换掉 `const` 函数，但 `async` 则相反：`async` 函数始终可以替换同步函数，但反之则不然。

这意味着虽然 `const Trait` 是 `Trait` 的 subtrait（实现 `const Trait` 的类型比 `Trait` 少），但是 `async Trait` 是 `Trait`
的 supertrait（实现 `async Trait` 的类型比 `Trait` 多）。

或者换句话说， `const Trait: Trait: async Trait`。

该系统的另一个重要影响是，与 `const` 不同，将实现从 `async Trait` 升级到 `Trait`
是一个重大更改，因为这些方法现在默认是同步的而不是异步的，因此您以前使用 `.await` 的所有地方都将获得错误。

当然，实际的用例数量普遍增加，而不是减少（将其传递给接受 `async Trait` 的函数仍然有效，并且这些方法仍然需要 `.await`
），但直接调用者需要修改其代码才能编译它。然而，这不应该是一个大问题，因为通常预先知道某些东西是否需要异步。

另一种选择是将 `async Trait` 和 `Trait` 视为两个完全独立的 trait，两者之间没有内在联系。

这样做的优点是可以防止在编译时，在异步函数中使用 `std::fs::File` 之类的错误（因为 `std::fs::File` _不会_ 实现 `async Read`），但总的来说，我认为这不值得:

1. 无论如何，使用者最终可能调用一个具体的阻塞函数而犯错误，例如 `Path` 上的 `.metadata()` 或 `std::thread::sleep`。它只能帮助预防少数情况。
2. 这并不总是一个错误；有时，在异步上下文中运行阻塞代码 _很有_ 用，例如，如果想要在阻塞工作线程上混合异步和阻塞函数调用。
3. 有时一个操作是否会 _真正_ 阻塞只能动态地知道，例如从 TCP 流中读取：如果流处于
   [非阻塞模式](https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html#method.set_nonblocking)（这是标准库明确支持的用例），那么从异步代码调用它应该没问题。
4. 默认情况下，像 `Vec<u8>` 这样的类型（其 `Write` 实现既不是异步也不是阻塞，因此可以在两种上下文中使用）最终将完全同步。为了支持两者，必须编写样板代码来分别实现
   `async Trait` 和 `Trait`，或者我们必须引入 _另一个_ 新的语法来共享实现。
    
    当考虑 `Drop` 时，情况会变得更糟。每个实现该 trait 的非泛型类型都必须迁移到这个新语法，甚至可以在异步上下文中使用（或者我们可以把
    `Drop` 特殊化以具有共享实现，但我不能想出一个强有力的理由来说明为什么 `Drop` 应该与其他事物区别对待）。
5. 将 trait 分开反而会增加系统整体的复杂性。
    
<div id="relaxed-drop-bounds"></div>

## 放宽 Drop bounds

我们在 [上一节](#function-implicit-bounds) 中介绍了隐式默认的 `Drop` 约束；现在我们有了一些异步 drop（`async Drop`）的实际语法，问题是如何为允许它的函数放宽这些约束。

我首先想在本节中介绍一个新概念： `?Drop` 约束。该约束可以被认为是添加隐式约束之前的初始约束，并且它对类型支持 drop 的程度绝对没有任何要求。

在任何情况下，在 `async Drop` 上都不需要此约束，因为最不“可 drop”的类型是 `async Drop`，应用它只会夺走实现者的能力，而不会向调用者提供任何能力。

但它仍然很重要，因为它让根本不关心传递 panic 检查的同步函数（`mem::replace` 、`any::type_name` 、`Option::map` 等）避免在签名中写通用的 `async`。

当它们实际上根本不异步 drop 类型时，声明 `<T: async Drop>` 或其他东西会感觉很奇怪。

它还允许将来扩展到更多类型的 drop，这[可能是有用的](#linear-types)。

所有函数对泛型参数都有比 `?Drop` 更强的默认约束，并且可以与 Rust 中的其他隐式约束相同的方式放宽到 `?Drop`：通过在参数列表或在 
where 子句中中添加 `?Drop` 作为 trait bound。

与 `Sized` 一样，它只接受简单的情况，因此 `?Drop` 不能用作 supertrait（[Drop 是默认值](#drop-supertrait)）
或用作字面类型参数以外的类型的约束。

这里有一个轻微的不一致之处：即使隐含的约束实际上不是 `Drop`，也会使用 `?Drop`，因为它实际上可能是 `async Drop`；所以在某种程度上，如果外部函数是
`async`，它实际上应该是 `?async Drop`；而如果外部函数是同步的，则应该是 `?Drop` 。但由于 `?Drop` 更短、更一致且明确，所以没有充分的理由不使用它。

当放宽到比默认弱但强于 `?Drop` 约束时，（特别是在同步函数中将它们设置为 `async Drop` ）最明显的选择是直接支持 trait 名称：使用
`T: async Drop` 来支持 `T` 不实现任何 `Drop` subtraits（`Drop` 、`const Drop`），但要求它实现 `async Drop`。

然而，这种方法最终会带来很大的问题，因为 `?Drop` 的独特语法使其只能支持少数特殊情况，而 `async Drop
`也是一个与其他任何情况一样的 trait，因此必须像其他任何情况一样在一般情况下得到支持。

这意味着，在更复杂的情况下（例如，通过 supertrait 隐含它，或通过应用于另一种类型的 `where` 子句中的约束传递），
`T: async Drop` 也会隐式放宽 `Drop` 约束，从而导致行为不一致和令人困惑的语义。

相反，Rust 应该采取一致的方法，_允许_（但可能警告）同步函数上的 `T: async Drop` 等边界，但不给它们任何效果，除非它们 _也_ 与 `?Drop` 一致。

由于 `Drop` 意味着 `async Drop`，因此在同步函数中添加 `async Drop` 是一个同义反复，只有去掉初始 `Drop` 约束才有意义。

这种方法的唯一问题是它很冗长： `T: ?Drop + async Drop` 表达一个概念相当拗口。

Rust 可能会引入一些语法糖来使其更短，唯一的困难是实际语法是什么，同时保持清晰和明确。我非常乐意接受这里的建议。

<div id="synchronous-opt-out"></div>

## 选择退出同步

虽然在大多数情况下，`const Trait` 中的每个方法都可以盲目地转化成 `const`。

但对于 `async Trait`，最终却效果不佳。特别是，无论外部 trait 是否被视为异步，有很多方法都可以从始终同步中受益，例如：

* `Iterator::size_hint` 和 `ExactSizeIterator::len`：这些方法应该是 O(1) 并且不执行 I/O，因此没有理由让它们成为 `async`。
* `Iterator::{step_by, chain, zip, map, filter, enumerate, ...}` ：这些函数只是构造一个类型并返回它，这里没有异步。
* `Read::{by_ref, bytes, chain, take}`：仅仅是构造类型的简单函数而已。
* `BufRead::consume`： `BufRead` 完成的任何 I/O 都应该发生在 `fill_buf` 中，并且所有 `consume` 应该做的就是移动几个数字。因此，它应该始终是同步的。

因此，显然 trait 定义需要能够控制其 `async` 形式的外观。

由 Rust 编译器选择任何一个作为默认都是一个坏主意，因为即使不考虑 `async` 代码，只需编写一个 trait，您就已经选择并稳定了 `async` API。

另外，并不像许多 trait 需要有异步等价物，主要只是 `Iterator` 、 I/O trait、函数和 `Drop` 很重要。因此，我认为最好让 trait 声明者选择添加 `async Trait` 支持。

声明这些 trait 语法比如有 `trait ~async Foo` 、`~async trait Foo` 或 `async trait Foo`。

我没有强烈的偏好，现在将使用第一个。为了将这些 trait 的方法在有些条件下声明为异步，实际上可以从通用异步函数借用相同的
`~async` 语法，`Self` 将被视为具有 `~async Trait` 约束的另一个通用参数。这在函数和 trait 之间产生了很好的相似性，如下所示：

```rust,ignore
// What you write
~async fn f<T: ~async Trait>() { /* ... */ }

trait ~async Trait { ~async fn f(); }

// What it "expands" to
async fn f_async<T: async Trait>() { /* ... */ }
fn f_sync<T: Trait>() { /* ... */ }

trait async Trait { async fn f(); }
trait Trait { fn f(); }
```

由于这些函数实际上只是常规的 `~async` 函数，因此它们还与通用参数交互：

```rust,ignore
trait ~async Trait {
    ~async fn f<T: ~async Read>(val: T);
}

// What it "expands" to
trait async Trait {
    async fn f_async<T: async Read>(val: T);
}
trait Trait {
    async fn f_async<T: async Read>(val: T);
    fn f_sync<T: Read>(val: T);
}

// A synchronous implementation
impl Trait for () {
    ~async fn f<T: ~async Read>(val: T) {}
}
// An asynchronous implementation
impl async Trait for u32 {
    async fn f<T: async Read>(val: T) {}
}
// A generic implementation
impl<T: ~async Trait> ~async Trait for &T {
    ~async fn f<T: ~async Read>(val: T) {}
}
```

就像常规的 `~async` 函数一样，仅当 _所有_ 泛型参数（此处为 `T` 和 `Self`）同步地实现该特征时，同步版本才存在。

最后要注意的是 `~async Trait` 中的关联类型将具有隐式 `~async Drop` 约束：当 trait 是 `async Trait` 时，它们允许是
`async Drop`，但当它是同步 `Trait` 时，它们需要是 `Drop`。这应该遵循使用者大多数时候想要的规则。

最后，我将为您提供一段带注释的片段，说明添加了 `async` 支持后 `Iterator` 特征的外观：

```rust,ignore
pub trait ~async Iterator {
    type Item;

    ~async fn next(&mut self) -> Option<Self::Item>;

    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }

    ~async fn fold<B, F>(mut self, init: B, f: F) -> B
    where
        Self: Sized,
        // `fold` always drops `Self` at the end so this bound is required.
        Self: ~async Drop,
        F: ~async FnMut(B, Self::Item) -> B,
        // We can't relax B's bound because it's dropped in the event that
        // `self.next()` panics.
    {
        let mut accum = init;
        // `.await` is required in both cases because it could be a cancellation
        // point.
        while let Some(x) = self.next().await {
            accum = f(accum, x).await;
        }
        accum
    }

    fn map<B, F>(self, f: F) -> Map<Self, F>
    where
        Self: Sized,
        // Even a synchronous iterator's `map` accepts an `async FnMut` here,
        // without the tilde. This is because every `FnMut` is also an
        // `async FnMut`, so `async FnMut` is the strictly more general bound.
        // The tilde is only necessary when the function effectively needs to
        // specialize on the synchronous case to not be async, but that's not
        // necessary here since `map` isn't ever async anyway.
        F: async FnMut(Self::Item) -> B,
        // The default bounds are overly restrictive, so we relax them.
        F: ?Drop,
        B: ?Drop,
    {
        Map::new(self, f)
    }

    // et cetera
}
```

与当前添加新的 `Stream` / `AsyncIterator` Trait 的设计相比，这具有以下优点：

* 对于像 `fold` 这样的函数，我们不必在异步回调和同步回调之间做出决定（目前
  [futures-util](https://docs.rs/futures-util/0.3/futures_util/stream/trait.StreamExt.html#method.fold) 和
  [tokio-stream](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.fold) 对此意见不一）。
* 我们没有两个单独的函数 `.map` 和 `.then` 分别用于同步和异步。
* 同步迭代器上可以调用使用异步函数的 `.map`，并自动将同步迭代器转换为异步迭代器。
* 不需要额外的转换函数，例如 `.into_stream()` 或 `.into_async_iter()`。
* 现有的迭代器，例如 `slice::Iter` 将自动实现新的 `async Iterator` 特征。

<div id="async-traits-and-backwards-compatibility"></div>

## 异步 traits 和向后兼容性

如果您仔细查看上面对`Iterator`的定义，会发现它实际上与 `Iterator` 的当前定义不向后兼容。

问题在于，今天我们可以覆盖像 `fold` 这样的函数，这些函数的功能不如 `~async` 版本强大。例如：

```rust,ignore
impl Iterator for Example {
    type Item = ();

    fn next(&mut self) -> Option<Self::Item> { Some(()) }

    fn fold<B, F>(mut self, mut accum: B, f: F) -> B
    where
        F: FnMut(B, Self::Item) -> B,
    {
        loop { accum = f(accum, ()) }
    }
}
```

根据我对 `Iterator` 的新定义，该代码需要像这样重写：

```rust,ignore
impl Iterator for Example {
    type Item = ();

    fn next(&mut self) -> Option<Self::Item> { Some(()) }

    ~async fn fold<B, F>(mut self, mut accum: B, f: F) -> B
    where
        F: ~async FnMut(B, Self::Item) -> B,
    {
        loop { accum = f(accum, ()).await }
    }
}
```

迭代器本身仍然不是异步的，但此更改还允许使用异步回调来调用 `fold` ，即使底层迭代器仍然是同步的。

不幸的是，由于 Rust 的向后兼容性保证，我们不能让第一个版本停止编译。

即使是一个版次也无法解决这个问题，因为问题不仅仅是语法问题。

我认为没有一种合理的方法可以以某种方式修复 `fold` 本身，此时它的签名实际上已确定。

但我们 _可以_ 向它添加一个 `where Self: Iterator<Item = Self::Item>` 约束，然后将通用版本命名为新名称 `fold_async` 。

由于 `fold_async` 比 `fold` 更通用，因此 `fold` 的默认实现可以直接转发给它。

所以 `Iterator` 的定义实际上看起来更像是这样的：

```rust,ignore
pub trait ~async Iterator {
    type Item;

    ~async fn next(&mut self) -> Option<Self::Item>;

    fn fold<B, F>(mut self, init: B, f: F) -> B
    where
        Self: Iterator<Item = Self::Item> + Sized + Drop,
        F: FnMut(B, Self::Item) -> B,
    {
        self.fold_async(init, f)
    }

    ~async fn fold_async<B, F>(mut self, init: B, f: F) -> B
    where
        Self: Sized + ~async Drop,
        F: ~async FnMut(B, Self::Item) -> B,
    {
        let mut accum = init;
        while let Some(x) = self.next().await {
            accum = f(accum, x).await;
        }
        accum
    }

    // et cetera
}
```

尽管它看起来与根本没有异步通用性非常相似，但它仍然比没有更好，因为：

1. 覆盖 `fold_async` 也可以有效地覆盖 `fold`，它们能够共享一个实现。
2. 异步和同步迭代器共享 `fold` 和 `fold_async` 的定义。

在我看来，这使得该功能仍然值得，即使我们必须在 `Iterator` 中插入一些技巧以避免破坏兼容性。

不幸的是，`fold` 并不是唯一需要这种处理的方法，可能许多其他方法也需要这种处理。

据我所知，这包括（仅在标准库中）：`chain` 、 `zip` 、 `map` 、 `for_each` 、 `filter` 、 `filter_map` 、 `skip_while` 、 `take_while` 、
`map_while` 、 `scan` 、 `flat_map` 、 `flatten` 、 `inspect` 、 `collect` 、 `partition` 、 `try_fold` 、 `try_for_each` 、 `reduce` 、
`all` 、 `any` 、 `find` 、 `find_map` 、 `position` 、 `rposition` 、 `sum` 、 `product` 、 `cmp` 、 `partial_cmp` 、 `eq` 、 `ne` 、
`lt` 、 `le` 、 `gt` 、 `ge` 、 `DoubleEndedIterator::try_rfold` 、 `DoubleEndedIterator::rfold` 、 `DoubleEndedIterator::rfind` 和 `Read::chain`。

如果 `async Clone` 或 `async Ord` 成为现实，这个列表将会变得更长。

有点遗憾的是，像 `map` 和 `Read::chain` 这样的函数必须有异步版本，因为无论如何都不会有人覆盖 `map` 。

但由于 _技术上_ 可行，Rust 已经承诺不会破坏该代码，因此现在无法放宽该函数的签名。

但是谁知道呢，也许如果我们得到一个低百分比的 Crater 运行回归，它会让人们相信这是可以接受的破损，并且列表可以缩短到更易于管理的 `for_each` 、
`partition`、 `try_fold`、 `try_for_each`、 `reduce` 、 `all`、 `any`、 `find` 、 `find_map` 、 `position` 、 `rposition` 、 `cmp` 、
`partial_cmp` 、 `eq` 、 `ne` 、 `lt` 、 `le` 、 `gt` 、 `ge` , `DoubleEndedIterator::try_rfold` 、 `DoubleEndedIterator::rfold` 和 `DoubleEndedIterator::rfind`。

我肯定宁愿这样做，因为坦率地说，如果你覆盖 `map`，那么你应该得到你得到的东西。

在该组中， `collect` 、 `sum` 和 `product` 是特别有趣的三个，因为它们的 `_async` 版本（如果我们接受技术上的重大更改，则它们的正常版本）不能使用标准的
`FromIterator` 、 `Product` 和 `Sum` trait，因为这些 traits 当前是硬编码的，仅适用于同步迭代器。所以我们必须使用旧版本的 blanket 实现来创建这些 traits 的新 `*Async` 版本：

```rust,ignore
// Not sure how useful `~async` is here; it would only be needed for collections
// that actually perform async work themselves while collecting as opposed to
// just potentially-asynchronously receiving the items and then synchronously
// collecting them.
//
// This is not true of any existing `FromIterator` or `FromStream`
// implementation currently, but there may still be use cases - who knows.
pub trait ~async FromAsyncIterator<A>: Sized {
    ~async fn from_async_iter<T: ~async IntoIterator<Item = A>>(iter: T) -> Self;
}
impl<T: FromAsyncIterator<A>, A> FromIterator<A> for T {
    fn from_iter<T: IntoIterator<Item = A>>(iter: T) -> Self {
        Self::from_async_iter(iter)
    }
}
```

`Sum` 和 `Product` 具有类似的代码。与 `Iterator::fold` 不同，由于 `from_iter` 、 `sum` 和 `product`
不是默认实现的方法，我们不能只是向 `FromIterator` trait 本身添加一个新的 `from_async_iter` 函数；需要一种全新的 trait。

<div id="trait-impl-implicit-bounds"></div>

## Trait impl implicit bounds  

[之前](#function-implicit-bounds)，我讨论过在 inherent impl 块内部，对外部类型泛型的隐式 `Drop` 约束如何根据其异步性单独应用于每个方法，并且块本身不会对类型强制执行任何约束。

不幸的是，在考虑 trait 实现时，我们没有那么幸运：trait 要么被实现，要么没有，并且我们不能将自己的约束应用于单个 item。

但是，我们 _确实_ 知道整个 trait 是否应该被视为异步 ，无论它是作为 `async Trait` 还是 `Trait` 实现的。

因此，我们可以将该属性转发为默认的 `Drop` 约束，这应该是使用者大多数时候想要的。

当然，对于（希望）罕见的 _不需要_ 的情况，他们总是可以覆盖它。

最明显的情况是当实现一个不是 `async Trait` trait 但仍然具有异步方法的 trait 时（即没有同步等价物的异步 trait），那么 Drop 约束最终会过于严格：

```rust,ignore
trait ExampleTrait {
    async fn foo<V>(&self, value: V);
}

struct Wrapper<T>(T);

impl<T> ExampleTrait for Wrapper<T>
where
    // overly-restrictive implied bound: `T: Drop`
{
    async fn foo<V>(&self, value: V)
    where
        // implied bound: `V: async Drop` (since it's declared
        // on the function and not on the impl block)
    {
        todo!()
    }
}
```

但幸运的是，这种代码不会太常见，因为理想情况下，使用者无论如何都应该将大多数代码编写为通用异步代码。

上述规则的一个有趣的副作用是在如下代码中：

```rust,ignore
struct Wrapper<T>(T);

impl<T /* implied Drop bound */> Drop for Wrapper<T> {
    fn drop(&mut self) {
        println!("I am being dropped");
    }
}
```

尽管不明显，但此代码无法编译，因为类型的 `Drop` 实现具有比类型本身更严格的 trait bound，而这是不允许的。

但由于看起来这段代码应该编译，我发现引入一个特殊情况并简单地让编译器转发约束到类型本身的隐式 `T: Drop` 是可以接受的，但前提是专门存在`Drop`实现。

无论哪种方式，该类型都不适用于`async Drop`类型，修复方法如下：

```rust,ignore
struct Wrapper<T>(T);

impl<T: ?Drop> Drop for Wrapper<T> {
    fn drop(&mut self) {
        println!("I am being dropped");
    }
}
```

<div id="async-closures"></div>

## 异步闭包

> 译者注：[RFC 3668: async closures](https://rust-lang.github.io/rfcs/3668-async-closures.html) 即将稳定。

使用闭包支持异步通用性（由 `Option::map` 和 `Iterator::fold` 等函数所需）需要 `async {Fn, FnMut, FnOnce}` 作为 trait 存在。

这似乎有点无用，因为我们已经有了返回 Future 的函数，但事实证明，拥有单独的 `async` 函数 traits 确实有好处，特别是在使用闭包时：它使生命周期更容易管理，因为返回的
Future 将能够借用闭包和参数 —— 这在当前的设计中是不可能的。

然而，为了使 `async Fn` traits 有用，它们必须由相关的函数和闭包实际实现。目前，人们通过返回 Futures ( `|| async {}`
) 的闭包来支持异步回调，并且 `async fn` 也被脱糖为这种形式的函数。

尝试改变前者的行为并不是一个好主意，因为这将需要一个黑魔法编译器特殊处理仅返回 Future 的闭包，但值得庆幸的是，我们保留了一些非常适合此用例的语法：异步闭包（ `async || {}` ）。

如果它们要计算为实现 `async Fn` 而不是 `Fn` 的闭包类型，则它们可以毫无问题地传递到 `Option::map` 等异步通用函数中。

```rust,ignore
// Gives an `Option<T>`, since the async `map` is used.
let output = some_option.map(async |value| process(value).await).await;

// Gives an `Option<impl Future<Output = T>>`, since the sync `map` is used.
let output = some_option.map(|value| async { process(value).await });
```

添加这个的不太好的一面是使用 `async fn` ：我们必须在将当前的脱糖系统保留为简单的 `-> impl Future` 函数和实现 `async Fn` trait 之间做出选择。

前者向后兼容且更透明（因为这些函数可以完全在使用者空间中复制），但后者与异步泛型函数具有更好的互操作性。我倾向于选择后一种设计，但这是一个不幸的决定。

请注意，不可能 _同时_ 实现 `async Fn` 和 `Fn` ，因为实现 `Fn` 已经意味着将 `async Fn` 实现为从不等待的异步函数；我们最终会得到
`async Fn` 的冲突实现，一种异步计算为 `T`，另一种立即计算为 `impl Future<Output = T>`。为了避免编译错误，我们必须选择一个并丢弃另一个。

<div id="conclusion"></div>

## 结论

在这篇文章中，我们勾画出了异步 drop 的潜在设计，并在此过程中弄清楚了许多细节和复杂性。

不幸的是，这个提议并不小，但是它在异步析构函数之外确实有很多普遍的用处（特别是 `~async` 对于这么多代码来说是非常好的），并且如果我们要最小化脚枪，那么很多东西都是必要的。

作为迄今为止我们探索过的一切的总结：

1. 我们在同步函数和泛型中，弄清楚了取消、 panic 和赋值期间涉及异步 drop 的边缘情况的语义。
2. 我们探索了一个基于 destructor Futures 而不是`poll_drop_ready` 的异步析构函数系统。
3. 我们探索了一种支持通用代码的机制，无论它是否为 `async`。
4. 我们假设什么最适合用作函数中的默认泛型 drop bounds，以及如何在必要时放松和加强它们。
5. 我们考虑了异步通用性将如何影响函数和闭包。

这篇文章并不试图提供异步 drop 的最终设计，仍然存在许多悬而未决的问题（例如 `UnwindSafe` 、 `?Drop`
语法、 `![no_std]`支持）和可能未知的未知数。

但它确实试图正确探索一种特定的设计，以评估其复杂性、可行性和实用性。

在所有可能的选择中，我认为这是一个很有前途的选择，并且绝对有可能以某种形式实施。

非常感谢 Yoshua Wuyts 为我校对了这篇文章！

<div id="completion-futures"></div>

## 附录 A: Completion futures

听起来它的功能不多，但 completion futures 实际上非常有用：

* 它们让 `spawn` 和 `spawn_blocking` 这些函数不会将 future 的生命周期限制为 `'static`。
* 它们能够围绕基于完成的 API（例如 `io_uring` 、 IOCP 和 libusb ）创建零成本包装器。
* 它们可以与 C++ futures 实现更好的互操作性，默认情况下具有此保证。

我之前 [为此编写了一个库](https://github.com/SabrinaJewson/completion)，但它非常有限，因为它从根本上需要依赖
`unsafe`，几乎每次使用它时都会感染到 `unsafe`，这确实不理想。

但事实证明，使用像这篇文章提出的异步析构函数设计，以更强大的方式和最小的 `unsafe` 来支持它们要容易得多。

解决方案是向核心库添加一个新 trait：

```rust,ignore
pub unsafe auto trait Leak {} 
```

作为 auto trait，它被每种类型实现，但除了特殊 `core::marker::PhantomNoLeak` marker 和任何传递地包含该标记的类型。

`Leak` 代表通过 [`mem::forget`](https://doc.rust-lang.org/stable/std/mem/fn.forget.html) 、引用循环或类似的东西安全地泄漏类型实例的能力。

如果一个类型选择不实现它，则可以保证从创建开始，如果该类型的生命周期结束，它的 `Drop` 或 `async Drop` 实现将会运行。

标准库将拥有所有“泄漏的” API，如 `Arc` 、 `Rc` 、 `ManuallyDrop` 和 `MaybeUninit` 要求在内部类型上实现 `Leak`，以避免安全代码能够绕过限制。

除此之外，大多数其他 API 将支持 `Leak` 和 `!Leak` 类型，因为它们将运行内部值的析构函数。

这就是支持 completion futures 所需的全部。

执行 `io_uring` I/O 操作的 future 可以通过在创建时提交操作并等待其在 drop 时完成来实现，并且 `!Leak` 保证意味着消除
`io_uring` 库当前必须解决的 [释放后使用问题](https://github.com/spacejam/rio/issues/30)。

这是一个非常强大的功能，甚至比我以前的基于 `unsafe` 的实现更强大。

因为它保证不会从创建中泄漏，而不仅仅是从第一次轮询中泄漏，所以作用域任务甚至不需要定义特殊的作用域（类似于 [crossbeam](https://docs.rs/crossbeam/0.8/crossbeam/fn.scope.html) ）。

相反，像这样的 API 就可以工作：

```rust,ignore
pub async fn spawn<'a, R, F>(f: F) -> JoinHandle<'a, R>
where
    F: Future<Output = R> + Send + 'a,
    R: Send,
{ /* ... */ }
```

它也会对同步代码产生影响，因为 [`thread::spawn`](https://doc.rust-lang.org/stable/std/thread/fn.spawn.html) 可以以类似的方式扩展：

```rust,ignore
pub fn spawn_scoped<'a, R, F>(f: F) -> JoinHandle<'a, R>
where
    F: FnOnce() -> R + Send + 'a,
    R: Send,
{ /* ... */ }
```

这将允许您编写从栈上借用的代码而不会出现问题：

```rust,ignore
let message = "Hello World".to_owned();

// Synchronous code
let thread_1 = thread::spawn_scoped(|| println!("{message}"));
let thread_2 = thread::spawn_scoped(|| println!("{message}"));
thread_1.join().unwrap();
thread_2.join().unwrap();

// Asynchronous code
let task_1 = task::spawn(async { println!("{message}") }).await;
let task_2 = task::spawn(async { println!("{message}") }).await;
task_1.await.unwrap();
task_2.await.unwrap();
```

很整洁，对吧？

与许多事情一样，它需要跨越一个版次才能完全实现：在当前版本中，每个通用参数仍然必须暗示
`T: Leak`，但在未来版次中可以放宽为 `T: ?Leak`，允许一小部分 API 在其签名中声明 _可以_ 泄漏值（`Arc` 、 `Rc` 、
`mem::forget` 、 `ManuallyDrop`等），并且大多数 API 默认情况下限制较少。

<div id="weakly-async-functions"></div>

## 附录 B: Weakly async functions  

在当前的设计中，最终会出现大量具有特定属性的函数，如果它们处理的类型是 `async Drop`，那么它们需要是
`async fn`，唯一的原因是该属性的类型出现在它们范围内时，可以 panic。

我在 [异步通用性](#async-genericity) 部分的开头列出了一些，包括 `HashMap::{insert, entry}` 、 `Vec::push` 和 `Box::new`。

但这里有一个特别相关的 `task::spawn` （见于各种运行时： [tokio](https://docs.rs/tokio/1/tokio/task/fn.spawn.html) 、
[async-std](https://docs.rs/async-std/1/async_std/task/fn.spawn.html) 、
[glommio](https://docs.rs/glommio/0.7/glommio/fn.spawn_local.html) 、 [smol](https://docs.rs/smol/1/smol/fn.spawn.html) ）。

在所有这些运行时中， `task::spawn` 能够在执行 future 之前发生 panic，如果运行时未运行，通常会发生这种情况，但理论上如果分配失败或存在其他一些随机系统错误，也可能会发生这种情况。

问题在于，仅仅因为这一小边缘情况（以及假设他们希望支持 `async Drop` futures）， `task::spawn` 被迫成为一个完整的 `async fn`，即使它 _本身_ 不执行任何 `async` 工作。

这对于作为函数的 `task::spawn` 来说尤其糟糕，因为它很容易让那些正在迁移代码的人陷入困境。

例如，在此之前，此代码将与 `other_work()` 并行运行任务：

```rust,ignore
let task = task::spawn(some_future);
other_work().await;
task.await;
```

实现重大更改后，它将运行 `other_work ()` 并等待它完成， _然后_ 执行任务，甚至不等待它完成！

当然，除非删除任务句柄将被更改为隐式 join 任务，这 _可能_ 是一个更好的整体设计。但这一点仍然成立，因为它并不像人们期望的那样并行运行。

修复版本如下所示：

```rust,ignore
let task = task::spawn(some_future).await;
other_work().await;
task.await; 
```

但考虑到旧版本甚至没有编译失败，这不是一个理想的情况。此外，处理另一个 future 看起来确实很奇怪。

我提出的解决这个问题的方法是，在语言中添加一种新类型的函数，称为“弱异步函数” (weakly async functions)，它介于异步函数和同步函数之间。

我们在这里用 `[async] fn` 表示它，但语法显然适合再合计合计。这个想法是这样的：

* `[async] fn` 要么同步完成，要么异步 panic。
* 因为它们必须同步完成，所以它们不能被取消，因此它们不需要 `.await`，这可以隐式进行。
* 因为它们是异步 panic 的，所以它们绕过了 panic 检查，并被允许在潜在的 panic 点上拥有带有异步析构函数的类型（但不允许 drop 它们，除非通过 panic 来 drop 它们）。
* 它们可以调用常规的 `fn` 和其他 `[async] fn`，但不能调用`async fn`。
* 不能从同步函数内调用它们。
* 它们不允许递归，就像 `async fn` 一样。
* 从 `[async] fn` 转换为常规 `fn` 并不是重大更改。

这样， `task::spawn` （以及一堆其他函数，如 `Box::new` 、 `Box::pin` 、 `Vec::push` 、 `Result::unwrap` 等）将避免在使用
`async Drop` 类型调用时需要 `.await`。这解决了上述问题，同时也有助于代码的简洁性。

`task::spawn`定义如下：

```rust,ignore
pub [async] fn spawn<O, F>() -> JoinHandle<O>
where
    F: Future<Output = O> + Send + ?Drop + async Drop + 'static,
    O: Send,
```

在异步上下文中，只需使用 `task::spawn(future)` 即可调用，无需等待。

当在通用代码中时， `[async]` 将被视为 `~async fn`，可以处于的另一种状态，这意味着实际上有三种方法可以使用这些函数。

另外函数还有 `~[async] fn` 形式，它可以是 `fn` 或 `[async] fn`，但不是 `async fn`。

您还需要一种特殊的约束来表示“当函数是同步时 `Drop` 和 当函数是 `async` 或 `[async]` 时 `async Drop`，因为该函数不会 drop
这个类型的值，除非它 panic”。

现在，我将使用极其冗长的 `~[async] async Drop` 形式来表示这一点，但如果实际上添加了此功能，则可能必须选择更好、更集思广益的语法。

这个特性允许我们通用地定义 `Vec::push`：

```rust,ignore
impl<T> Vec<T> {
    ~[async] fn push(&mut self, item: T)
    where
        T: ?Drop + ~[async] async Drop,
    {
        /* ... */
    }
}

// "Expanded" version
impl<T> Vec<T> {
    fn push_sync(&mut self, item: T)
    where
        T: Drop,
    {
        /* ... */
    }
    ~[async] fn push_weak_async(&mut self, item: T)
    where
        T: ?Drop + async Drop,
    {
        /* ... */
    }
}
```

请记住，此函数可以 drop `item` ，因此不能完全同步，但也不会 drop `item` 除非它 panic，因此也不应该完全 `async`。

因此，它使用中间的方式，当它是 `[async] fn` 时支持 `async Drop` （因此也支持 `[async] Drop` ），当它是 `fn` 时支持 `Drop`。

与 completion futures 不同，我不太确定这是否是一个好主意，或者是否有其他更简单的替代方案。

但我确实认为这里确实存在一个需要以某种方式解决的问题，对我来说这似乎是最好的方法。

<div id ="linear-types"></div>

## 附录 C: Linear types

我觉得我必须至少提到一次线性类型 (linear types)，因为关于线性类型的讨论已经有很多了。

线性类型被定义为“必须仅使用一次的类型”。事实证明这个定义有点模糊，因为它可以指两件事：

1. 没有任何 `Drop` 实现且必须显式处理的类型，但可能会通过 [`mem::forget`](https://doc.rust-lang.org/stable/std/mem/fn.forget.html) 等函数泄漏。
2. 确实具有析构函数的类型可能会隐式超出范围，但不能通过像 [`mem::forget`](https://doc.rust-lang.org/stable/std/mem/fn.forget.html)
   这样的函数泄漏（因此保证它们能够在超出范围之前运行代码）。

前者是线性类型的更常见定义，并允许类型强制其使用者更明确地了解它们被销毁时会发生什么。

我没有这方面的建议，但只是巧合地提出了 `?Drop` 约束功能确实面向 Future 支持此类线性类型，虽然我个人认为它们不值得添加，但它们的可行性已经增加作为副作用。

后一个定义是由上述 [completion futures](#completion-futures) 提案实现的。

在某种程度上，它不是真正的线性类型，但它是唯一一种能够提供零成本 `io_uring` 和作用域任务等实际好处的类型。

集成到现有的 Rust 代码中也容易得多，这些代码往往严重依赖现有的析构函数，但不太依赖于安全可泄漏的值。

<div id="uncancellable-futures"></div>

## 附录 D: Uncancellable futures  

我之前反对 [Carl Lerche 的建议，让所有异步函数不可取消](https://carllerche.netlify.app/2021/06/17/six-ways-to-make-async-rust-easier/)，而赞成为
`.await`定义一致的语义而不是删除它。

然而，这些类型的功能并非完全不可能。这样的功能仍然肯定存在，首先作为使用者空间内的组合器：

```rust,ignore
pub async fn must_complete<F: Future>(fut: F) -> F::Output {
    MustComplete(fut).await
}

#[pin_project(PinnedDrop)]
struct MustComplete<F: Future>(#[pin] F);

impl<F: Future + ?Drop + async Drop> Future for MustComplete<F> {
    type Output = F::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut task::Context<'_>) -> Poll<Self::Output> {
        self.project().0.poll(cx)
    }
}

#[pinned_drop]
impl<F: Future> async PinnedDrop for MustComplete<F> {
    async fn drop(self: Pin<&mut Self>) {
        self.project().0.await;
    }
}
```

可以像这样使用：

```rust,ignore
must_complete(async {
    some_very_important_work().await;
    that_must_not_be_interrupted().await;
})
.await;
```

它也可以作为一种语言功能存在，如果需要的话，还可以删除 `.await` 。

无论哪种方式，效果都是相同的：该提案可以轻松地编写保证没有取消点的 futures。

就我个人而言，我认为这个用例不够常见，不足以保证语言功能，但它仍然绝对值得考虑。

