This article is about building actors with Tokio directly, without using any
actor libraries such as Actix. This turns out to be rather easy to do, however
there are some details you should be aware of:

本文介绍如何使用Tokio直接构建参与者，而不使用Actix等任何参与者库。事实证明，这很容易做到，但是有一些细节您应该知道：

1. Where to put the `tokio::spawn` call.
1. Struct with `run` method vs bare function.
1. Handles to the actor.
1. Backpressure and bounded channels.
1. Graceful shutdown.

The techniques outlined in this article should work with any executor, but for
simplicity we will only talk about Tokio.  There is some overlap with the
[spawning](https://tokio.rs/tokio/tutorial/spawning) and [channel chapters](https://tokio.rs/tokio/tutorial/channels) from the Tokio tutorial, and I recommend also
reading those chapters.

在哪里放置`tokio：：spawn`调用。使用`run`方法的Struct与Bare函数。到参与者的句柄。反压力和绑定的通道。正常关闭。本文中概述的技术应该适用于任何Executor，但为了简单起见，我们只讨论Tokio。与Tokio教程中的产卵和通道章节有一些重叠，我建议您也阅读这些章节。

Before we can talk about how to write an actor, we need to know what an actor
is. The basic idea behind an actor is to spawn a self-contained task that
performs some job independently of other parts of the program. Typically these
actors communicate with the rest of the program through the use of message
passing channels. Since each actor runs independently, programs designed using
them are naturally parallel.

在我们讨论如何写演员之前，我们需要知道演员是什么。参与者背后的基本思想是产生一个独立于程序其他部分执行某些工作的独立任务。通常，这些参与者通过使用消息传递通道与程序的其余部分进行通信。由于每个演员都独立运行，因此使用他们设计的程序自然是并行的。

A common use-case of actors is to assign the actor exclusive ownership of some
resource you want to share, and then let other tasks access this resource
indirectly by talking to the actor. For example, if you are implementing a chat
server, you may spawn a task for each connection, and a master task that routes
chat messages between the other tasks. This is useful because the master task
can avoid having to deal with network IO, and the connection tasks can focus
exclusively on dealing with network IO.

参与者的一个常见用例是将您想要共享的某些资源的独占所有权分配给参与者，然后让其他任务通过与参与者对话来间接访问该资源。例如，如果您正在实现聊天服务器，则可以为每个连接派生一个任务，并在其他任务之间生成一个在聊天消息之间发送聊天消息的主任务。这很有用，因为主任务可以避免必须处理网络IO，而连接任务可以专门专注于处理网络IO。

## The Recipe

## 食谱

An actor is split into two parts: the task and the handle. The task is the
independently spawned Tokio task that actually performs the duties of the actor,
and the handle is a struct that allows you to communicate with the task.

演员分为两个部分：任务和句柄。该任务是独立生成的Tokio任务，它实际执行参与者的职责，句柄是允许您与该任务进行通信的结构。

Let's consider a simple actor. The actor internally stores a counter that is
used to obtain some sort of unique id. The basic structure of the actor would be
something like the following:

让我们来考虑一个简单的演员。参与者在内部存储用于获取某种唯一ID的计数器。参与者的基本结构如下所示：

```rust
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

Now that we have the actor itself, we also need a handle to the actor. A handle
is an object that other pieces of code can use to talk to the actor, and is also
what keeps the actor alive.

现在我们有了演员本身，我们还需要一个演员的句柄。句柄是其他代码段可以用来与参与者对话的对象，也是使参与者存活的原因。

The handle will look like this:

手柄将如下所示：

```rust
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

[full example](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=1e60fb476843fb130db9034e8ead210c)

完整示例

Let's take a closer look at the different pieces in this example.

让我们仔细看看本例中的不同部分。

**`ActorMessage.`** The `ActorMessage` enum defines the kind of messages we can
send to the actor. By using an enum, we can have many different message types,
and each message type can have its own set of arguments. We return a value to
the sender by using an [`oneshot`](https://docs.rs/tokio/1/tokio/sync/oneshot/index.html) channel, which is a message passing channel
that allows sending exactly one message.

\`ActorMessage.``ActorMessage`枚举定义了我们可以向参与者发送的消息类型。通过使用枚举，我们可以拥有许多不同的消息类型，并且每种消息类型都可以有自己的一组参数。我们使用`ones hot`通道向发送方返回值，该通道是一个消息传递通道，允许只发送一条消息。

In the example above, we match on the enum inside a `handle_message` method on
the actor struct, but that isn't the only way to structure this. One could also
match on the enum in the `run_my_actor` function. Each branch in this match
could then call various methods such as `get_unique_id` on the actor object.

在上面的示例中，我们匹配参与者结构上的`handleMessage`方法中的枚举，但这并不是构造它的唯一方式。还可以匹配`run_my_actor`函数中的枚举。然后，此匹配中的每个分支都可以在参与者对象上调用各种方法，如`get_Unique_id`。

**Errors when sending messages.** When dealing with channels, not all errors are
fatal.  Because of this, the example sometimes uses `let _ =` to ignore errors.
Generally a `send` operation on a channel fails if the receiver has been
dropped.

发送消息时出错。在处理通道时，并非所有错误都是致命的。因此，该示例有时会使用`let_=`忽略错误。通常，如果接收器已被丢弃，则在通道上的`send`操作失败。

The first instance of this in our example is the line in the actor where we
respond to the message we were sent. This can happen if the receiver is no
longer interested in the result of the operation, e.g. the task might that sent
the message might have been killed.

在我们的示例中，第一个实例是参与者中响应我们收到的消息的行。如果接收方对操作的结果不再感兴趣，例如发送消息的任务可能已被终止，则可能会发生这种情况。

**Shutdown of actor.** We can detect when the actor should shut down by looking
at failures to receive messages. In our example, this happens in the following
while loop:

演员的停业。我们可以通过查看接收消息的失败来检测参与者何时应该关闭。在我们的示例中，这发生在以下While循环中：

```rust
while let Some(msg) = actor.receiver.recv().await {
    actor.handle_message(msg);
}
```

When all senders to the `receiver` have been dropped, we know that we will never
receive another message and can therefore shut down the actor. When this
happens, the call to `.recv()` returns `None`, and since it does not match the
pattern `Some(msg)`, the while loop exits and the function returns.

当“接收者”的所有发送者都被删除时，我们知道我们将永远不会再收到另一条消息，因此可以关闭参与者。发生这种情况时，对`.recv()`的调用返回`None`，并且由于它与模式`Some(Msg)`不匹配，While循环退出，函数返回。

**`#[derive(Clone)]`** The `MyActorHandle` struct derives the `Clone` trait. It
can do this because [`mpsc`](https://docs.rs/tokio/1/tokio/sync/mpsc/index.html) means that it is a multiple-producer,
single-consumer channel. Since the channel allows multiple producers, we can
freely clone our handle to the actor, allowing us to talk to it from multiple
places.

\`#[派生(Clone)]``MyActorHandle`结构派生`Clone`特征。它之所以能做到这一点，是因为`mpsc`意味着它是一个多生产者、单消费者的渠道。由于该频道允许多个制作人，我们可以自由克隆我们的演员句柄，允许我们从多个地方与它交谈。

## A run method on a struct

## 结构上的Run方法

The example I gave above uses a top-level function that isn't defined on any
struct as the thing we spawn as a Tokio task, however many people find it more
natural to define a `run` method directly on the `MyActor` struct and spawn
that. This certainly works too, but the reason I give an example that uses a
top-level function is that it more naturally leads you towards the approach that
doesn't give you lots of lifetime issues.

我上面给出的示例使用了一个顶级函数，该函数没有在任何结构上定义为我们作为Tokio任务派生的东西，但是许多人发现直接在`MyActor`结构上定义一个`run`方法并派生它会更自然。这当然也是有效的，但我给出一个使用顶级函数的例子的原因是，它更自然地将您引向不会给您带来很多生命周期问题的方法。

To understand why, I have prepared an example of what people unfamiliar with the
pattern often come up with.

为了理解为什么，我准备了一个例子，说明不熟悉这种模式的人经常会想到什么。

```rust
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

... and no separate MyActorHandle
```

The two sources of trouble in this example are:

本例中的两个故障来源是：

1. The `tokio::spawn` call is inside `run`.
1. The actor and the handle are the same struct.

The first issue causes problems because the `tokio::spawn` function requires the
argument to be `'static`. This means that the new task must own everything
inside it, which is a problem because the method borrows `self`, meaning that it
is not able to give away ownership of `self` to the new task.

\`tokio：：spawn`调用在`run`内部。参与者和句柄是相同的结构。第一个问题导致问题，因为`tokio：：spawn`函数需要参数为`‘static`。这意味着新任务必须拥有它内部的所有东西，这是一个问题，因为该方法借用了`self`，这意味着它无法将`self`的所有权让给新任务。

The second issue causes problems because Rust enforces the single-ownership
principle. If you combine both the actor and the handle into a single struct,
you are (at least from the compiler's perspective) giving every handle access to
the fields owned by the actor's task. E.g. the `next_id` integer should be owned
only by the actor's task, and should not be directly accessible from any of the
handles.

第二个问题引发了问题，因为Rust执行的是单一所有权原则。如果您将参与者和句柄组合到单个结构中，则(至少从编译器的角度来看)您为每个句柄提供了对参与者任务所拥有的字段的访问权限。例如，`Next_id`整数应该只属于参与者的任务，并且不能从任何句柄直接访问。

That said, there is a version that works. By fixing the two above problems, you
end up with the following:

这就是说，有一个版本是有效的。通过修复上述两个问题，您将得到以下结果：

```rust
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
        tokio::spawn(async move { actor.run().await });

        Self { sender }
    }
}
```

This works identically to the top-level function. Note that, strictly speaking,
it is possible to write a version where the `tokio::spawn` is inside `run`, but
I don't recommend that approach.

这与顶级函数的工作方式相同。请注意，严格来说，可以编写`tokio：：spawn`位于`run`中的版本，但我不推荐这种方法。

## Variations on the theme

## 主题的变奏曲

The actor I used as an example in this article uses the request-response
paradigm for the messages, but you don't have to do it this way. In this section
I will give some inspiration to how you can change the idea.

我在本文中用作示例的参与者对消息使用请求-响应范例，但您不必这样做。在这一节中，我将给你一些启发，告诉你如何改变这个想法。

### No responses to messages

### 没有回复消息

The example I used to introduce the concept includes a response to the messages
sent over a `oneshot` channel, but you don't always need a response at all. In
these cases there's nothing wrong with just not including the `oneshot` channel
in the message enum. When there's space in the channel, this will even allow you
to return from sending before the message has been processed.

我用来介绍这个概念的例子包括对通过‘one hot`通道发送的消息的响应，但您并不总是需要响应。在这些情况下，只要不在消息枚举中包含`ones hot`通道就没什么错了。当通道中有空间时，这甚至允许您在消息处理之前从发送返回。

You should still make sure to use a bounded channel so that the number of
messages waiting in the channel don't grow without bound. In some cases this
will mean that sending still needs to be an async function to handle the cases
where the `send` operation needs to wait for more space in the channel.

您仍应确保使用有界通道，以便通道中等待的消息数量不会无限制地增长。在某些情况下，这将意味着发送仍然需要是一个异步函数，以处理`send`操作需要在通道中等待更多空间的情况。

However there is an alternative to making `send` an async method. You can use
the `try_send` method, and handle sending failures by simply killing the actor.
This can be useful in cases where the actor is managing a `TcpStream`,
forwarding any messages you send into the connection. In this case, if writing
to the `TcpStream` can't keep up, you might want to just close the connection.

然而，除了将`send`设置为异步方法之外，还有一种替代方法。您可以使用`try_send`方法，通过简单杀死参与者来处理发送失败。这在参与者管理`TcpStream`，转发您发送到连接中的任何消息的情况下非常有用。在这种情况下，如果写入`TcpStream`跟不上，您可能需要关闭连接。

### Multiple handle structs for one actor

### 一个参与者的多个句柄结构

If an actor needs to be sent messages from different places, you can use
multiple handle structs to enforce that some message can only be sent from some
places.

如果某个参与者需要从不同的位置发送消息，您可以使用多个句柄结构来强制某些消息只能从某些位置发送。

When doing this you can still reuse the same `mpsc` channel internally, with an
enum that has all the possible message types in it. If you *do* want to use
separate channels for this purpose, the actor can use [`tokio::select!`](https://docs.rs/tokio/1/tokio/macro.select.html) to
receive from multiple channels at once.

这样做时，您仍然可以在内部重用相同的`mpsc`通道，其中的枚举包含所有可能的消息类型。如果您确实想单独使用多个频道，操作者可以使用`tokio：：SELECT！`一次接收多个频道。

```rs
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

You need to be careful with how you handle when the channels are closed, as
their `recv` method immediately returns `None` in this case. Luckily the
`tokio::select!` macro lets you handle this case by providing the pattern
`Some(msg)`. If only one channel is closed, that branch is disabled and the
other channel is still received from. When both are closed, the else branch runs
and uses `break` to exit from the loop.

关闭通道时需要注意如何处理，因为在这种情况下，它们的`recv`方法会立即返回`None`。幸运的是，`tokio：：select！`宏允许您通过提供模式`Some(Msg)`来处理这种情况。如果只有一个通道关闭，则该分支被禁用，而另一个通道仍从接收。当两个分支都关闭时，Else分支运行并使用`Break`退出循环。

### Actors sending messages to other actors

### 演员向其他演员传递信息

There is nothing wrong with having actors send messages to other actors. To do
this, you can simply give one actor the handle of some other actor.

让演员向其他演员发送信息并没有什么错。要做到这一点，您只需将其他演员的句柄交给一个演员即可。

You need to be a bit careful if your actors form a cycle, because by holding on
to each other's handle structs, the last sender is never dropped, preventing
shutdown. To handle this case, you can have one of the actors have two handle
structs with separate `mpsc` channels, but with a `tokio::select!` that looks
like this:

如果你的参与者形成一个循环，你需要小心一点，因为通过保持彼此的句柄结构，最后一个发送者永远不会被丢弃，从而防止关机。要处理这种情况，可以让其中一个参与者有两个句柄结构，它们有单独的`mpsc`通道，但有一个`tokio：：select！`，如下所示：

```rs
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

The above loop will always exit if `chan1` is closed, even if `chan2` is still
open. If `chan2` is the channel that is part of the actor cycle, this breaks the
cycle and lets the actors shut down.

如果`chan1`关闭，则上述循环将始终退出，即使`chan2`仍处于打开状态。如果`chan2‘是演员循环的一部分，这就打破了循环，让演员们关门了。

An alternative is to simply call [`abort`](https://docs.rs/tokio/1/tokio/task/struct.JoinHandle.html#method.abort) on one of the actors in the cycle.

另一种方法是简单地对循环中的一个参与者调用`bort`。

### Multiple actors sharing a handle

### 多个参与者共享一个句柄

Just like you can have multiple handles per actor, you can also have multiple
actors per handle. The most common example of this is when handling a connection
such as a `TcpStream`, where you commonly spawn two tasks: one for reading and
one for writing. When using this pattern, you make the reading and writing tasks
as simple as you can — their only job is to do IO. The reader task will just
send any messages it receives to some other task, typically another actor, and
the writer task will just forward any messages it receives to the connection.

就像每个角色可以有多个句柄一样，每个句柄也可以有多个角色。最常见的例子是在处理像`TcpStream`这样的连接时，通常会产生两个任务：一个用于读取，一个用于写入。当使用这种模式时，您可以使读写任务尽可能简单--他们唯一的工作就是做IO。读取器任务只会将它接收到的任何消息发送到其他任务，通常是另一个参与者，而写入器任务只会将它接收到的任何消息转发到连接。

This pattern is very useful because it isolates the complexity associated with
performing IO, meaning that the rest of the program can pretend that writing
something to the connection happens instantly, although the actual writing
happens sometime later when the actor processes the message.

这种模式非常有用，因为它隔离了与执行IO相关的复杂性，这意味着程序的其余部分可以假装向连接写入某些内容是立即发生的，尽管实际写入发生在参与者处理消息的一段时间之后。

## Beware of cycles

## 当心自行车

I already talked a bit about cycles under the heading “Actors sending messages
to other actors”, where I discussed shutdown of actors that form a cycle.
However, shutdown is not the only problem that cycles can cause, because a cycle
can also result in a deadlock where each actor in the cycle is waiting for the
next actor to receive a message, but that next actor wont receive that message
until its next actor receives a message, and so on.

我已经在“演员向其他演员发送信息”的标题下谈到了一些周期，在那里我讨论了形成周期的演员的关闭。然而，关闭并不是循环可能导致的唯一问题，因为循环还可能导致死锁，其中循环中的每个参与者都在等待下一个参与者接收消息，但该下一个参与者直到其下一个参与者接收到消息才会收到该消息，依此类推。

To avoid such a deadlock, you must make sure that there are no cycles of
channels with bounded capacity. The reason for this is that the `send` method on
a bounded channel does not return immediately. Channels whose `send` method
always returns immediately do not count in this kind of cycle, as you cannot
deadlock on such a `send`.

为了避免这种死锁，您必须确保不存在容量受限的通道循环。原因是有界通道上的`send`方法不会立即返回。`send`方法总是立即返回的通道不会计入这种循环，因为这样的`send`不会死锁。

Note that this means that a oneshot channel cannot be part of a deadlocked
cycle, since their `send` method always returns immediately. Note also that if
you are using `try_send` rather than `send` to send the message, that also
cannot be part of the deadlocked cycle.

请注意，这意味着OneShot通道不能是死锁循环的一部分，因为它们的`send`方法总是立即返回。还要注意，如果您使用`try_send`而不是`send`来发送消息，这也不能是死锁周期的一部分。

Thanks to [matklad](https://matklad.github.io/) for pointing out the issues
with cycles and deadlocks.

感谢Matklad指出了循环和死锁的问题。