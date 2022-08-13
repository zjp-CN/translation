# 共享可变状态

> 原文：《[Shared mutable state in Rust]》 by tokio 维护者 [Alice Ryhl] 

本文解释了如何在 Rust 中共享可变值。例如，共享值可以是 hashmap 或计数器。

这在异步应用程序中通常是必需的，所以我们将探索如何在同步和异步应用程序中做到这一点。

本文的重点是共享数据。

如果你想共享 IO 资源，请参考我写的《[Actors with tokio]》。

如果 IO 资源是数据库连接，可以使用 [`r2d2`] （非异步）或 [`bb8`] （异步）等连接池。

[Shared mutable state in Rust]: https://github.com/Darksonn/ryhl.io/blob/master/content/blog/shared-mutable-state/index.md
[Alice Ryhl]: https://github.com/Darksonn 
[Actors with tokio]: https://github.com/Darksonn/ryhl.io/blob/master/content/blog/actors-with-tokio/index.md
[`r2d2`]: https://crates.io/crates/r2d2
[`bb8`]: https://crates.io/crates/bb8

## 在多个线程之间共享

要在多个线程之间共享值，你需要两件事：

1. 共享值：使用 [`Arc`]
2. 修改值：使用 [`Mutex`]

[`Arc`]: https://doc.rust-lang.org/stable/std/sync/struct.Arc.html.
[`Mutex`]: https://doc.rust-lang.org/stable/std/sync/struct.Mutex.html.

`Arc` 允许你共享值，因为无论何时克隆 (clone) `Arc`，你都会得到相同的共享值的新句柄。对内部值所做的任何更改都将在 `Arc`
的所有其他克隆中可见。这也使得克隆 `Arc` 变得非常便宜，因为你实际上不必复制其中的数据。

当最后一个 `Arc` 超出范围时，`Arc` 中的数据会被销毁。

如果数据被其他机制共享，如全局变量，则不需要 `Arc`。

然而，`Arc` 本身只能提供对其内部值的不可变访问，因为当另一个线程可能正在同时读取某个值时，修改该值是不安全的。

当这种情况发生时，这就称为数据竞争 (data race)。

如果你所需要的只是共享一个不变的值，那很好，但本文将介绍如何修改共享值。所以，还需要一个 `Mutex`。

`Mutex` 的目的是确保当时只有一个线程可以访问该值。它使用 [`lock`] 方法来实现这一点，该方法返回一个 [`MutexGuard`]。

如果你调用 `lock` 时，而另一个线程存在来自同一 `Mutex` 的 `MutexGuard`，则对 `lock` 的调用将休眠，直到那个 `MutexGuard` 
超出范围。这保证了一次最多只能存在一个 `MutexGuard`，并且不论何时访问内部值时都必须经过一个
`MutexGuard`。因此，可以保证没有两个线程可以同时访问共享值。这样就可以安全地修改该值。

[`lock`]: https://doc.rust-lang.org/stable/std/sync/struct.Mutex.html#method.lock
[`MutexGuard`]: https://doc.rust-lang.org/stable/std/sync/struct.MutexGuard.html

无论如何，既然我们了解了一般方法，那么来看一个例子。我推荐的基本写法如下：

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Clone)]
pub struct SharedMap {
    inner: Arc<Mutex<SharedMapInner>>,
}

struct SharedMapInner {
    data: HashMap<i32, String>,
}

impl SharedMap {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Mutex::new(SharedMapInner {
                data: HashMap::new(),
            }))
        }
    }

    pub fn insert(&self, key: i32, value: String) {
        let mut lock = self.inner.lock().unwrap();
        lock.data.insert(key, value);
    }

    pub fn get(&self, key: i32) -> Option<String> {
        let lock = self.inner.lock().unwrap();
        lock.data.get(&key).cloned()
    }
}
```

本示例介绍如何创建共享的 `HashMap`。 `SharedMap` 类型派生实现了 `Clone`，但对其调用 `clone` 
时，由于 `Arc`，实际上并不会克隆其中的所有数据。且正相反，克隆它是你可以在多个线程之间共享值的方式。

下面是一个使用它的示例：

```rust
fn main() {
    let map = SharedMap::new();

    map.insert(10, "hello world".to_string());

    let map1 = map.clone();
    let map2 = map.clone();

    let thread1 = std::thread::spawn(move || {
        map1.insert(10, "foo bar".to_string());
    });

    let thread2 = std::thread::spawn(move || {
        let value = map2.get(10).unwrap();
        if value == "foo bar" {
            println!("Thread 1 was faster");
        } else {
            println!("Thread 2 was faster");
        }
    });

    thread1.join().unwrap();
    thread2.join().unwrap();
}
```

这个例子有时会打印 `Thread 1 was faster`，有时会打印 `Thread 2 was faster`。这展示了如何在多个线程之间共享相同的 map。

### 用结构体包装起来

人们常常会定义自己的结构体，并在代码中传递 `Arc<Mutex<MyStruct>>`，然后到处都是 `lock` 调用。

但这是一个糟糕的主意，原因有几个：

1. 在读取代码时， `lock` 调用是不必要的杂乱。
2. 每次方法通过参数获取集合时，都必须在方法签名中编写一个不必要的 `Arc`/`Mutex`。这也是混乱的。
3. 你正在泄露实现细节。例如，如果你想用 `Mutex` 替换一个 `RwLock`，那么必须在整个代码库中改变每个共享集合。
4. 很容易意外地将 `Mutex` 锁定太长时间。这在异步代码中是一个特别糟糕的问题。

因此，我建议你定义一个包装结构，并将 `Arc`/`Mutex` 放在其中，对包装结构上的方法隔离所有 `lock` 
调用（见下面的做法）。这确保了锁是调用者不必关心的隐藏实现细节。

### 异步代码

这篇文章中的技术也适用于异步代码，但你需要注意一件事：

> 当互斥锁被锁定时，你不能 `.await` 任何东西。

在大多数情况下，如果你尝试这么做，编译器会给你一个错误。

但在某些情况下，编译器不会捕捉到它。此时，你的程序发生死锁。

要了解其中的原因，你必须对 Rust 中的 async/await 的工作原理有所了解。

简单地说，async/await 的工作原理是重复“交换”当前任务，以便它可以在单个线程上运行多个任务。这种“互换”只能在 `.await` 的情况下发生。

死锁是可能发生的，因为如果你在 `Mutex` 被锁定时调用 `.await`，那么该 `Mutex` 将保持锁定状态，直到该线程切换回该任务。

如果另一个任务试图锁定同一线程上的同一 `Mutex`，那么由于 `lock` 调用不是 `.await`，它将永远无法切换回另一个任务。这是形成死锁。

根据我的经验，避免上述问题的最好方法是：从一开始就不要锁定异步代码中的 `Mutex`。

相反，你可以在包装结构上定义非异步方法，并在那里锁定 `Mutex`。然后，从你的异步代码中调用这些非异步方法。

由于你不能在非异步方法中使用 `.await`，因此不可能在 `Mutex` 被锁定时意外地 `.await` 某个东西。

为了说明这一点，考虑一个例子。假设，你要实现一个 
debouncer，这让某个事件在一段时间内没有发生前一直休眠。经常在用户界面中使用到它，例如，一旦用户停止打字就自动搜索的搜索栏。

为了实现它，需要创建一个共享变量，其中包含希望休眠在何时停止的 [`Instant`]，每次事件发生时都会不断更新共享的 `Instant`。

[`Instant`]: https://doc.rust-lang.org/stable/std/time/struct.Instant.html

```rust
use std::sync::{Arc, Mutex};
use tokio::time::{Duration, Instant};

#[derive(Clone)]
pub struct Debouncer {
    inner: Arc<Mutex<DebouncerInner>>,
}

struct DebouncerInner {
    deadline: Instant,
    duration: Duration,
}

impl Debouncer {
    pub fn new(duration: Duration) -> Self {
        Self {
            inner: Arc::new(Mutex::new(DebouncerInner {
                deadline: Instant::now() + duration,
                duration,
            })),
        }
    }

    /// Reset the deadline, increasing the duration of any calls to `sleep`.
    pub fn reset_deadline(&self) {
        let mut lock = self.inner.lock().unwrap();
        lock.deadline = Instant::now() + lock.duration;
    }

    /// Sleep until the deadline elapses.
    pub async fn sleep(&self) {
        // This uses a loop in case the deadline has been reset since the
        // sleep started, in which case the code will sleep again.
        loop {
            let deadline = self.get_deadline();
            if deadline <= Instant::now() {
                // The deadline has already elapsed. Just return.
                return;
            }
            tokio::time::sleep_until(deadline).await;
        }
    }

    fn get_deadline(&self) -> Instant {
        let lock = self.inner.lock().unwrap();
        lock.deadline
    }
}
```

通过上面的实现，`Debouncer` 可以被克隆并移动到多个不同的地方，允许一个任务重置其他任务的睡眠时长，使它们睡眠的时间更长。

请注意，本例中的 `sleep` 方法从未真正锁定 `Mutex`，即使它需要读取其中的数据。相反，它使用 `get_deadline` 
这个辅助性的函数。这样，我们就不会意外地忘记在调用 `sleep_until` 之前 unlock 互斥锁。

### 编译器捕捉到的死锁

如果你在执行 `.await` 时试图锁定互斥锁，编译器通常会捕获我前面描述的死锁，并给你一个错误。错误如下所示：

```rust
error: future cannot be sent between threads safely
   --> src/lib.rs:13:5
    |
13  |     tokio::spawn(async move {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-0.2.21/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/lib.rs:7:5
    |
4   |     let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    |         -------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
...
7   |     do_something_async().await;
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ await occurs here, with `mut lock` maybe used later
8   | }
    | - `mut lock` is later dropped here
```

出现此错误是因为 `Mutex::lock` 返回的 `MutexGuard` 类型对于跨线程发送是不安全的，而每当你的异步任务在 `.await` 处被交换出去时，
tokio 可能会将你的异步任务移动到一个新的线程中。

仅当同时满足以下两个条件时才会出现此编译错误：

1. 你使用任务的方式是它可以跨线程移动。
2. 你使用的互斥锁的 `MutexGuard` 没有实现 `Send`。

如果你使用 [`tokio::task::spawn_local`] 运行异步代码，则第一个条件不满足，你不会收到错误。包含在 
`#[tokio::main]` 中的 [`block_on`] 也是如此。

此外，如果你使用来自外部库的锁，并且该锁的 guard 实现了 `Send`，那么你也不会收到该错误。这方面的一个例子是 
[`dashmap`]。因此，在使用 `dashmap` 时，请特别注意不要将其锁定在异步代码中。

[`tokio::task::spawn_local`]: https://docs.rs/tokio/latest/tokio/task/fn.spawn_local.html,
[`block_on`]: https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html#method.block_on
[`dashmap`]: https://docs.rs/dashmap

需要注意的是， `parking_lot` 库中的 `MutexGuard` 通常不是 `Send`，但是它有一个 `send_guard` feature 
可以开启，这会让它的所有 guards 都是 `Send`。 `dashmap` 启用该功能，因此如果你依赖 `dashmap`，则必须小心所有的 
`parking_lot` 互斥锁，因为该 feature 是全局启用的。

### tokio 中的 `Mutex`

[tokio-mutex]: https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html
[tokio-rwlock]: https://docs.rs/tokio/latest/tokio/sync/struct.RwLock.html

tokio 提供的 [`Mutex`][tokio-mutex] 和 [`RwLock`][tokio-rwlock] 是一个异步锁。这意味着调用 `lock` 使用了一个 
`.await`，它允许运行时在 `lock` 调用处于休眠状态时交换任务。这使得在锁被锁定时，能够不产生死锁来 `.await` 某些东西。

只有在锁被锁定时需要 `.await` 某些东西的情况下，你才应该使用异步锁。

通常，这并不是必需的，你应该尽可能避免使用异步锁。异步锁比阻塞锁慢得多。

无法在类型的析构函数中锁定异步锁，因为析构函数必须是同步的。

## `Mutex` 的替代方案

根据你要存储的值以及需要修改它的频率，有几个替代方案可能更合适。

对于这里提到的所有备选方案，我仍然建议你将它们包装在某种自定义结构中。

### 特定场景

其中最广为人知的是 [`RwLock`] 类型，它允许多个读取器并行读取值，但只有一个写入器。

另一种不太为人所知但非常有用的库是 [`arc-swap`]。当你有一个很少修改的值时，它很有用，因为你可以完全避免锁定。与 `RwLock`
相比，它最主要的区别在于，在任何写入过程中，`RwLock` 都会阻止所有读取器，而使用 `arc-swap`
时，你可以同时读写，现有的读取器会继续看到旧值。这种方法通常需要为每个修改克隆整个值，但可以使用引用计数来降低成本，你可能会使用
[`im`] 库。有时，将 `arcswap` 与 `Mutex` 组合在一起是有意义的，比如，`Mutex` 只被想要访问共享值的线程锁定。

另一个有用的工具是 [`evmap`] 库 (eventually consistent map)，它让你通过拥有两个副本来修改共享的
hashmap：一个用于读取，另一个用于写入。然后，这两张个 map 会每隔一段时间交换一次。此数据结构被称为最终一致 
(eventually consistent)，是因为你不能保证立即看到你的写入：它们可能尚未应用于两个 
map，但如果等待足够长的时间，你最终会看到你的写入。

还有 [`dashmap`] 库，它的工作原理是将 map 拆分成几个碎片 (shards)，并在每个碎片周围放置一个 
`RwLock`。对不同碎片的访问是完全独立的，但对同一碎片的访问类似于普通的 `RwLock<HashMap<K，V>>`。注意，
`dashmap` 使用的 guard 实现了 `Send`，所以如果你将其锁定在一个 `.await`
上，编译器将不会捕获它。如果你不遵循只在非异步函数中锁定它的建议，你最终会得到死锁。

如果你的共享类型是某种类型的缓存，则还可以将其存储在线程局部变量中 ([`thread_local`])。
这将为每个线程创建一个单独的版本，如果多个线程想要使用数据，则你需要承担复制数据的成本，但这意味着你可以访问该值，而无需担心线程安全。

最后，如果你共享的数据类型是整数，你可以使用 [`std::sync::atomic`] 类型，这让你可以共享可变数据，而不需要任何锁定。

[`RwLock`]: https://doc.rust-lang.org/stable/std/sync/struct.RwLock.html
[`arc-swap`]: https://docs.rs/arc-swap
[`im`]: https://docs.rs/im
[`evmap`]: https://docs.rs/evmap
[`thread_local`]: https://doc.rust-lang.org/std/macro.thread_local.html
[`std::sync::atomic`]: https://doc.rust-lang.org/stable/std/sync/atomic/index.html

### `Mutex` 还是 `RwLock`

我发现有一种倾向，那就是总是使用 `RwLock`，而从不使用 `Mutex`。

如果读取值的次数比修改值的次数多得多，那么这是一个很好的选择，但你必须注意饥饿问题 (starvation)：

> 当许多线程一直从一个 `RwLock` 读取数据时，可能导致在很长一段时间内阻止写入器获取写锁，因为读取器的数量永远不会降为零。这就是饥饿。

使用 `Mutex`，你不太可能遇到饥饿问题。

另一种避免饥饿的方法是使用 [`parking_lot`] 库的 `RwLock`，它使用了公平的策略，以防止写入器挨饿。

tokio 中的异步锁也运用了公平策略。

[`parking_lot`]: https://docs.rs/parking_lot
[fairness policy]: https://docs.rs/parking_lot/0.11/parking_lot/type.RwLock.html#fairness

## 如何返回引用

不需要。对 `Mutex` 内的值的引用只有在被锁定时才能存在，所以返回引用只能通过返回 `MutexGuard`
来完成。这可能会导致问题，例如，如果从异步代码调用方法，`MutexGuard` 突然不再隔离到非异步函数。

最简单的解决方法是简单地克隆值。这就是我们在第一个示例中所做的。

如果你希望能够在异步操作中访问该值，那么你几乎必须这样做。有一些方法可以让克隆变得更便宜。例如，使用 `Arc<str>` 来代替 `String`。

另一种选择是使用 `with_*` 模式，如下所示：

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Clone)]
pub struct SharedMap {
    inner: Arc<Mutex<SharedMapInner>>,
}

struct SharedMapInner {
    data: HashMap<i32, String>,
}

impl SharedMap {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Mutex::new(SharedMapInner {
                data: HashMap::new(),
            }))
        }
    }

    pub fn insert(&self, key: i32, value: String) {
        let mut lock = self.inner.lock().unwrap();
        lock.data.insert(key, value);
    }

    pub fn with_value<F, T>(&self, key: i32, func: F) -> T
    where
        F: FnOnce(Option<&str>) -> T,
    {
        let lock = self.inner.lock().unwrap();
        func(lock.data.get(&key).map(|string| string.as_str()))
    }
}
```

```rust
fn main() {
    let shared = SharedMap::new();
    shared.insert(10, "foo".to_string());

    shared.with_value(10, |value| {
        println!("The value is {:?}.", value);
    });
}
```

```text
The value is Some("foo").
```

这种模式很有用，因为它允许你在互斥锁被锁定的情况下运行代码，并在异步代码中调用 `with_value`
不会导致前面提到的任何问题。当你不想对共享值执行的每个操作定义新方法时，它也很有用。

## 在 `Mutex` 内放置 `TcpStream`

在编写聊天服务器之类的东西时，一个常见的错误是定义一个集合，如 `HashMap<UserID, TcpStream>`，然后将其放入某种锁中。

我认为这是一个很大的错误，我从来没有真正看到过它的结果是好的。要处理这种情况，我建议你改用
[actor 模式][Actors with tokio]，其中每个 `TcpStream` 由独自专门用于该 `TcpStream`
的派生任务。然后，你可以将 actor handles 放在一个 `HashMap` 中。
