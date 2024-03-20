# Actors with tokio

> 原文：《[Actors with tokio]》 by tokio 维护者 [Alice Ryhl] 

[Alice Ryhl]: https://github.com/Darksonn 
[Actors with tokio]: https://github.com/Darksonn/ryhl.io/blob/master/content/blog/actors-with-tokio/index.md

本文介绍如何使用 tokio 直接构建 actors，而不使用 Actix 等任何 actor 库。事实证明，这很容易做到，但是有一些细节你应该知道：

1. 在哪里放置 `tokio::spawn` 调用
2. 对比使用 `run` 方法的结构体与单纯的函数
3. actor 的句柄 (handle)
4. 背压 (backpressure) 和有界通道 (bounded channel)
5. 优雅地关闭

本文中概述的技术应该适用于任何 executor，但为了简单起见，我们只讨论 tokio。

这里与 tokio 教程中的 [spawning] 和 [channel] 章节有一些重叠，我建议你也阅读这些章节。

[spawning]: https://tokio.rs/tokio/tutorial/spawning
[channel]: https://tokio.rs/tokio/tutorial/channels

## 什么是 actor

在我们讨论如何写 actor 之前，需要知道 actor 是什么。

actor 背后的基本思想是产生一个独立于程序其他部分、执行某些工作的独立任务。通常，这些 actor
通过使用消息传递通道与程序的其余部分进行通信。由于每个 actor 都独立运行，因此使用这种设计的程序自然是并行的。

actor 的一个常见用例是将你想要共享的某些资源的独占所有权分配给 actor，然后让其他任务通过与 actor 对话来间接访问该资源。

例如，如果你正在实现聊天服务器，则可以为每个连接生成一个任务，并在其他任务之间生成一个在聊天消息之间发送聊天消息的主任务。

这很有用，因为主任务可以避免处理网络 IO，而连接任务可以专门专注于处理网络 IO。

本文与 [这个演讲](https://www.youtube.com/watch?v=fTXuGRP1ee4) 内容一致。

## 典型步骤

actor 分为两个部分：任务和句柄。任务是独立生成的 tokio 任务，它实际执行 actor 的职责，句柄是允许你与该任务进行通信的结构。

考虑一个简单的 actor。 actor 在内部存储某种唯一 ID 的计数器，其基本结构如下所示：

```rust,ignore
use tokio::sync::{oneshot, mpsc};

struct MyActor {
    receiver: mpsc::Receiver<ActorMessage>,
    next_id: u32,
}
enum ActorMessage {
    GetUniqueId {
        respond_to: oneshot::Sender<u32>,
    },
}

impl MyActor {
    fn new(receiver: mpsc::Receiver<ActorMessage>) -> Self {
        MyActor {
            receiver,
            next_id: 0,
        }
    }
    fn handle_message(&mut self, msg: ActorMessage) {
        match msg {
            ActorMessage::GetUniqueId { respond_to } => {
                self.next_id += 1;

                // The `let _ =` ignores any errors when sending.
                //
                // This can happen if the `select!` macro is used
                // to cancel waiting for the response.
                let _ = respond_to.send(self.next_id);
            },
        }
    }
}

async fn run_my_actor(mut actor: MyActor) {
    while let Some(msg) = actor.receiver.recv().await {
        actor.handle_message(msg);
    }
}
```

现在我们有了 actor 本身，我们还需要一个 actor 的句柄。句柄是其他代码可以用来与 actor 对话的对象，也是使 actor 存活的原因。

句柄看起来是这样的：

```rust,ignore,ignore
#[derive(Clone)]
pub struct MyActorHandle {
    sender: mpsc::Sender<ActorMessage>,
}

impl MyActorHandle {
    pub fn new() -> Self {
        let (sender, receiver) = mpsc::channel(8);
        let actor = MyActor::new(receiver);
        tokio::spawn(run_my_actor(actor));

        Self { sender }
    }

    pub async fn get_unique_id(&self) -> u32 {
        let (send, recv) = oneshot::channel();
        let msg = ActorMessage::GetUniqueId {
            respond_to: send,
        };

        // Ignore send errors. If this send fails, so does the
        // recv.await below. There's no reason to check for the
        // same failure twice.
        let _ = self.sender.send(msg).await;
        recv.await.expect("Actor task has been killed")
    }
}
```

[完整示例 (playground)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=1e60fb476843fb130db9034e8ead210c)

让我们仔细看看本例中的不同部分。

`ActorMessage` 枚举定义了我们可以向 actor 发送的消息类型。通过使用枚举，我们可以拥有许多不同的消息类型，并且每种消息类型都可以有自己的一组参数。这里使用
[`oneshot`] 通道向发送方返回值，该通道是一个消息传递通道，只允许发送一条消息。

在 `handle_message` 方法中匹配 actor 结构上的枚举。但这并不是构造它的唯一方式，你还可以在 `run_my_actor` 函数中匹配枚举。

然后，此匹配中的每个分支都可以在 actor 对象上调用各种方法，如 `get_unique_id`。

发送消息时出错。在处理通道时，并非所有错误都是致命的。因此，该示例有时会使用
` let _ = ` 忽略错误。通常，如果接收器已被丢弃，则在通道上的 `send` 操作失败。

第一处是 actor 中响应我们收到的消息的行。如果接收方对操作的结果不再感兴趣，例如发送消息的任务可能已被终止，则可能会发生这种情况。

关闭 actor。我们可以通过查看接收消息的失败来检测 actor 何时应该关闭。在示例中，这发生在以下 while 循环：

```rust,ignore,ignore
while let Some(msg) = actor.receiver.recv().await {
    actor.handle_message(msg);
}
```

当 `receiver` 的所有发送者都被 drop 时，我们知道将永远不会再收到另一条消息，因此可以关闭 actor。此时，调用
`.recv()` 将返回 `None`，并且由于它与模式 `Some(msg)` 不匹配，while 循环退出，函数返回。

`MyActorHandle` 派生实现了 `Clone` trait。之所以能做到这一点，是因为 `mpsc`
意味着它是多生产者、单消费者的通道。由于该通道允许多个生产者，我们可以自由克隆 actor 的句柄，来允许从多个地方与它对话。

[`oneshot`]: https://docs.rs/tokio/1/tokio/sync/oneshot/index.html
[`mpsc`]: https://docs.rs/tokio/1/tokio/sync/mpsc/index.html

## `run` 方法

上面的示例使用了一个顶级函数 `run_my_actor`，该函数没有在任何结构上定义，就像 tokio 生成 (spawn) 一个任务。

但是许多人发现直接在 `MyActor` 结构上定义一个 `run` 方法并把它生成任务会更自然。

这当然也是可以的，但我这么做的原因是，它更自然地不会给你带来很多生命周期问题。

为了理解为什么，我准备了一个例子，来说明不熟悉这种模式的人经常会遇到什么。

```rust,ignore,ignore
impl MyActor {
    fn run(&mut self) {
        tokio::spawn(async move {
            while let Some(msg) = self.receiver.recv().await {
                self.handle_message(msg);
            }
        });
    }

    pub async fn get_unique_id(&self) -> u32 {
        let (send, recv) = oneshot::channel();
        let msg = ActorMessage::GetUniqueId {
            respond_to: send,
        };

        // Ignore send errors. If this send fails, so does the
        // recv.await below. There's no reason to check for the
        // same failure twice.
        let _ = self.sender.send(msg).await;
        recv.await.expect("Actor task has been killed")
    }
}

// 并且不把 MyActorHandle 分离出来
```

这里的两个故障来源是：

1. 在 `run` 内部调用 `tokio::spawn` 调用
2. actor 和句柄处于同一个的结构

问题一：因为 `tokio::spawn` 函数要求参数为 `'static`。这意味着新任务必须拥有它内部的所有东西。而 `run`
借用了 `self`，这意味着它无法将 `self` 的所有权让给新任务。

问题二：因为 Rust 执行的是单一所有权原则，如果你将 actor 和句柄一起放到单个结构中，那么至少从编译器的角度来看，
你为每个句柄提供了对 actor 任务所拥有的字段的访问权限。例如，`next_id` 整数应该只属于 actor 的任务，并且不能从任何句柄直接访问。

这意味着有一个版本是有效的。通过修复上述两个问题，得到以下结果：

```rust,ignore
impl MyActor {
    async fn run(&mut self) {
        while let Some(msg) = self.receiver.recv().await {
            self.handle_message(msg);
        }
    }
}

impl MyActorHandle {
    pub fn new() -> Self {
        let (sender, receiver) = mpsc::channel(8);
        let actor = MyActor::new(receiver);
        tokio::spawn(async move { actor.run().await }); // 注意这里

        Self { sender }
    }
}
```

这与顶级函数的工作方式相同。

要注意，严格来说，尽管可以编写一个在 `run` 中调用 `tokio::spawn` 的版本，但我不推荐这种方法。

## 话题衍生

我在本文中用作示例的 actor 是消息使用请求-响应 (request-response) 情况下的范例，但你不必这样做。

在这一节中，我将给你一些启发，告诉你如何改变这个想法。

### 不响应消息

以响应通过 `oneshot` 通道发送的消息情况作为例子，但你并不总是需要响应消息。

此时，只要不在消息枚举中包含 `oneshot` 通道就没问题了。当通道中有空间 (space) 时，这甚至允许你在消息处理之前从发送方返回。

你仍应确保使用有界通道，以便通道中等待的消息数量不会无限制地增长。

在某些情况下，这将意味着发送仍需要为一个异步函数，以处理 `send` 操作需要在通道中等待更多空间的情况。

然而，除了使用 `send` 异步方法之外，还有一种替代方法。使用 `try_send` 方法，通过简单杀死 actor 来处理发送失败。

这在这种情况有用：actor 管理 `TcpStream` 时，转发你所发送的任何消息；如果写入 `TcpStream` 跟不上，你可能需要关闭连接。

### 单 actor 多句柄

如果某个 actor 需要从不同的地方发送消息，你可以使用多个句柄结构来强制某些消息只能从某些位置发送。

[`tokio::select!`]: https://docs.rs/tokio/1/tokio/macro.select.html

此时，你仍然可以在内部重用相同的 `mpsc` 通道，其中的枚举包含所有可能的消息类型。

如果你确实想单独使用多个通道，actor 可以使用 [`tokio::select!`] 来一次接收多个通道。

```rust,ignore
loop {
    tokio::select! {
        Some(msg) = chan1.recv() => {
            // handle msg
        },
        Some(msg) = chan2.recv() => {
            // handle msg
        },
        else => break,
    }
}
```

要注意如何处理关闭通道，因为此时，它们的 `recv` 方法会立即返回 `None`。

幸运的是， `tokio::select!` 宏让你通过提供模式 `Some(msg)` 来处理这种情况：
- 如果只有一个通道关闭，则该分支被禁用，而另一个通道仍从接收；
- 当两个分支都关闭时，else 分支运行并使用 `break` 退出循环。

### 向其他 actors 发送消息

让 actors 向其他 actors 发送信息并没有问题。要做到这一点，你只需将其他 actor 的句柄交给一个 actor 即可。

如果你的 actor 形成一个循环，你需要小心一点，由于保持彼此的句柄结构，最后一个发送者永远不会被丢弃，从而防止关机。

要处理这种情况，可以让其中一个 actor 有两个句柄结构，它们有单独的 `mpsc` 通道，但有一个 `tokio::select!`，如下所示：

```rust,ignore
loop {
    tokio::select! {
        opt_msg = chan1.recv() => {
            let msg = match opt_msg {
                Some(msg) => msg,
                None => break,
            };
            // handle msg
        },
        Some(msg) = chan2.recv() => {
            // handle msg
        },
    }
}
```

如果 `chan1` 关闭，则上述循环将始终退出，即使 `chan2` 仍处于开启状态。如果 `chan2` 是 actor 循环的一部分，这就打破了循环，让 actors 停止。

另一种方法是简单地对循环中的一个 actor 调用 [`abort`]。

[`abort`]: https://docs.rs/tokio/1/tokio/task/struct.JoinHandle.html#method.abort

### 多 actors 共享一个句柄

就像每个 actor 可以有多个句柄一样，每个句柄也可以有多个 actor。

最常见的例子是在处理像 `TcpStream` 这样的连接时，通常会产生两个任务：一个用于读取，一个用于写入。你可以使读写任务尽可能简单
—— 他们唯一的工作就是做 IO。读取器任务只会将它接收到的任何消息发送到其他任务，通常是另一个 
actor，而写入器任务只会将它接收到的任何消息转发到连接。

这种模式非常有用，因为它把与执行 IO 
带来的复杂性分离开，这意味着程序的其余部分可以假装向连接写入某些内容是立即发生的，尽管实际的写入发生在
actor 处理消息的一段时间之后。

## 当心循环

我已经在“向其他 actors 发送信息”那部分谈到了一些循环，在那里我讨论了形成循环的 actors 如何关闭。

然而，关闭并不是循环可能导致的唯一问题，因为循环还可能导致死锁，其中循环中的每个 actor 都在等待下一个 actor
接收消息，但那下一个 actor 直到它的下一个 actor 接收到消息才会收到该消息，依此类推。

为了避免这种死锁，你必须确保容量受限的通道中不存在循环。原因是有界通道上的 `send`
方法不会立即返回。

而 `send` 方法总是立即返回的通道不会计入这种循环，从而这样的 `send` 不会导致死锁。

注意，这意味着 `Oneshot` 通道不可能是死锁循环的一部分，因为它们的 `send`
方法总是立即返回。还要注意，如果你使用 `try_send` 而不是 `send` 来发送消息，这也不可能是死锁循环的一部分。

感谢 [Matklad] 指出了循环和死锁的问题。

[matklad]: https://matklad.github.io

