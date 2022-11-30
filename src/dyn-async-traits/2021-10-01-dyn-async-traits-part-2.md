# Dyn Async Traits

[原文](https://smallcultfollowing.com/babysteps/blog/2021/10/01/dyn-async-traits-part-2/) |
日期：2021-10-01 11:56 -0400

在[上一篇][previous]文章中，我们揭示了 `dyn` 和异步 traits 的一个关键挑战：在今天的 Rust 中， `dyn` 类型必须指定所有关联类型的值。

而这篇文章将深入探讨 dyn traits 如何在当今发挥作用的更多背景，特别是讨论这种限制来自哪里。

[previous]: ./2021-09-30-dyn-async-traits-part-1.md

## Dyn trait 已实现了 trait 

在今天的 Rust 中，假设你有一个 "dyn-safe" trait `DoTheThing`，那么类型 `dyn DoTheThing` 就实现了 `Trait`。

```rust,ignore
trait DoTheThing {
	fn do_the_thing(&self);
}

impl DoTheThing for String {
    fn do_the_thing(&self) {
        println!("{}", self);
    }
}
```

现在想象一下使用 trait 的一些泛型函数：

```rust,ignore
fn some_generic_fn<T: ?Sized + DoTheThing>(t: &T) {
	t.do_the_thing();
}
```

当然，我们可以用 `&String` 来调用 `some_generic_fn`，但是因为 `dyn DoTheThing` 实现了 `DoTheThing`，所以也可以用 `&dyn DoTheThing` 来调用 `some_generic_fn`：

```rust,ignore
fn some_nongeneric_fn(x: &dyn DoTheThing) {
    some_generic_fn(x)
}
```

## 简单回顾 Dyn Safety

在 Rust 的早期，我们讨论了 `dyn DoTheThing` 是否应该实现 `DoTheThing` 这个 trait 。

这其实是"动态安全" (dyn safe) （当时称为"对象安全" (object safe) ）这一术语的由来。当时，我支持现在的做法：即创建一个二元属性。
* 要么 trait 是 dyn 安全的，此时 `dyn DoTheThing` 实现了 `DoTheThing`
* 要么 trait 不是 dyn 安全的，此时 `dyn DoTheThing` 不是合法类型

我现在不再确定这是不是正确的决定。

我当时喜欢这个想法，在这个模型中，每当你看到像 `dyn DoTheThing` 这样的类型时，就知道可以像实现 `DoTheThing` 的任何其他类型一样使用 `dyn DoTheThing`。

遗憾的是，在实践中，`dyn DoTheThing` 类型无法像 `String` 这样的类型。尤其是 `dyn` 类型没有大小，因此你不能通过值传递它们，也不能像 String 一样使用它们。

你必须始终传递指向它们的某种指针，例如 `Box<dyn DoTheThing>` 或 `&dyn DoTheThing`。这相当"不常见"，所以我们让你选择性地通过写 `T: ?Sized` 来在泛型函数中加入它。

这意味着，在实践中，泛型函数并不"自动"接受 `dyn` 类型，你必须显式地为 dyn 进行设计。因此，我预想的很多好处都没有实现。

## 静态分发 | 动态分发 | 虚表

让我们来谈谈 dyn safety 以及它的来源。

首先，我们需要解释静态分发 (static dispatch) 和虚拟（动态）分发 (virtual/dyn dispatch) 之间的区别。

简单地说，静态分发意味着编译器知道正在调用哪个函数，而动态分发意味着编译器不知道。

就 CPU 本身而言，没有太大区别:
* 在静态分发中，有一条"硬编码"指令说"在这个地址调用代码"[^link]。
* 在动态分发中，有一条指令说"调用地址在这个变量中的代码"。后者可能会慢一点，但在实践中几乎无关紧要，特别是在预测成功的情况下。

[^link]: 模组动态链接 (modulo dynamic linking)

当你使用一个 `dyn` trait 时，你实际拥有的是一个虚表 (vtable)。你可以将 vtable 看作是一种结构体，它包含一组函数指针，每个指针对应于 trait
中的每个方法。因此，`DoTheThing` trait 的 vtable 类型可能如下所示（实际上，有一些额外的数据，但对于我们的目的来说已经足够接近了）：

```rust,ignore
struct DoTheThingVtable {
    do_the_thing: fn(*mut ())
}
```

这里的 `do_the_thing` 方法有一个相应的字段。注意，第一个参数的类型应该是 `&self`，但我们将其改为 `*mut()`。

这是因为整个概念的 vtable 是不知道 `self` 类型是什么，所以只将其更改为"一些指针"（这就是我们需要知道的全部内容）。

当你创建 vtable 时，你是在创建此结构体的一个实例，该实例是为某个特定类型量身定做的。

在示例中，类型 `String` 实现了 `DoTheThing`，所以我们可以为 `String` 创建 vtable，如下所示：

```rust,ignore
static Vtable_DoTheThing_String: &DoTheThingVtable = &DoTheThingVtable {
    do_the_thing: <String as DoTheThing>::do_the_thing as fn(*mut ())
    //            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //            Fully qualified reference to `do_the_thing` for strings
};
```

你可能听说过 Rust 中的 `&dyn DoTheThing` 类型是一个宽指针 (wide pointer)。这意味着，在运行时，它实际上是两个指针：一个指向数据的指针和一个指向
`DoTheThing` trait 的 vtable 指针。所以 `&dyn DoTheThing` 大致相当于：

```rust,ignore
(*mut (), &’static DoTheThingVtable)
```

当你将 `&String` 转换为 `&dyn DoTheThing` 时，在运行时实际发生的是编译器获取 `&String` 指针，将其转换为 `*mut ()`，并将其与相应的 vtable 
配对。因此，如果你有一些如下所示的代码：

```rust,ignore
let x: &String = &"Hello, Rustaceans".to_string();
let y: &dyn DoTheThing = x;
```

它最后"解糖"成了这样：

```rust,ignore
let x: &String = &"Hello, Rustaceans".to_string();
let y: (*mut (), &'static DoTheThingVtable) = 
    (x as *mut (), Vtable_DoTheThing_String);
```

## impl dyn

我们已经了解了如何创建宽指针以及编译器如何表示 vtable，还看到，在 Rust 中， `dyn DoTheThing` 实现了 `DoTheThing`。

你可能想知道这是怎么回事。从概念上讲，编译器生成一个 impl，其中 trait 中的每个方法都是通过 vtable 提取函数指针并调用它来实现的：

```rust,ignore
impl DoTheThing for dyn DoTheThing {
    fn do_the_thing(self: &dyn DoTheThing) {
        // Remember that `&dyn DoTheThing` is equivalent to
        // a tuple like `(*mut (), &'static DoTheThingVtable)`:
        let (data_pointer, vtable_pointer) = self;

        let function_pointer = vtable_pointer.do_the_thing;
        function_pointer(data_pointer);
    }
}
```

实际上，当我们使用 `T = dyn DoTheThing` 调用一个泛型函数时，就像调用任何其他类型一样，将该调用单态化。

对 `do_the_thing` 的调用将对上面的 impl 进行分发，而实际执行动态分发的正是那个特殊的 impl。干净利落。

## 静态分发允许单态化

现在我们已经了解了 vtable 是如何以及何时构建的，接下来讨论 dyn safety 的规则以及它们来自哪里。

最基本的规则之一是， trait 只有在不包含泛型方法的情况下才是 dyn safe 的（或者，更准确地说，如果它的方法只对生命周期是泛型的，而不是对类型泛型）。

此规则的原因直接源于 vtable 的工作方式：当你构造 vtable 时，需要为 trait 中的每个方法（或者，可能是有限的函数指针集）提供单个函数指针。

泛型方法的问题是它们没有单一的函数指针：你需要为应用它们的每种类型使用不同的指针。考虑这个示例 `PrintPrefix` trait：

```rust,ignore
trait PrintPrefixed {
    fn prefix(&self) -> String;
    fn apply<T: Display>(&self, t: T);
}

impl PrintPrefixed for String {
    fn prefix(&self) -> String {
        self.clone()
    }
    fn apply<T: Display>(&self, t: T) {
        println!("{}: {}", self, t);
    }
}
```

`String as PrintPrefix` 的 vtable 会是什么样子？为 `prefix` 生成函数指针是没有问题的，只需使用 `<String as PrintPrefix>::prefix` 即可。

但 `apply` 呢？必须为 `<String as PrintPrefix>::Apply<T>` 生成一个函数指针，但我们还不知道 `T` 是什么！

相比之下，使用静态分发时，我们在调用点之前不必知道 `T` 是什么，此时，只需生成所需要的副本。

## 部分 dyn impls

前面的一点说明了 trait 可能有一些 dyn safe 的方法，也可能有一些不是 dyn safe 的方法。

在当前的 Rust中，这使得整个 trait 是不安全的，这是因为我们没有办法编写一个完整的 `impl PrintPrefix for dyn PrintPrefix`：

```rust,ignore
impl PrintPrefixed for dyn PrintPrefixed {
    fn prefix(&self) -> String {
        // For `prefix`, no problem:
        let prefix_fn = /* get prefix function pointer from vtable */;
        prefix_fn(…);
    }
    fn apply<T: Display>(&self, t: T) {
        // For `apply`, we can’t handle all `T` types, what field to fetch?
        panic!("No way to implement apply")
    }
}
```

在很久以前考虑的替代设计中，我们可以说 `dyn PrintPrefix` 的值总是合法的，但只当它的所有方法（和其他条目）都是 dyn safe 的时，
`dyn PrintPrefix` 才实现 `PrintPrefix` trait。

或者如果你有一个 `&dyn PrintPrefix`，你可以调用 `prefix`，只是不能将 `dyn PrintPrefix` 与 `fn foo<T: ?Size + PrintPrefix>` 这样的泛型代码一起使用。

我们将在以后的博客文章中回到这个主题。

如果你熟悉需要 `Where Self: Sized` 的 trait 方法的"特殊情况"，你可能会明白它是从哪里来的。

如果一个方法要求 `where Self: Sized`，而我们对 `dyn PrintPrefix` 这样的类型有一个 impl，那么这个 impl 永远不能被调用，所以我们可以从 impl （和 
vtable）中完全省略该方法。这与 `dyn PrintPrefix` 总是合法的说法非常相似，因为这意味着只有一部分方法可以通过动态分发使用。

不同之处在于 `dyn PrintPrefix: PrintPrefix` 仍然成立，因为我们知道泛型代码将无法调用那些非 dyn safe 方法，因为泛型代码必须要求 `T: ?Sized`。

## 关联类型和 dyn 类型

我们从讨论关联类型和 `dyn` 类型开始这一长篇故事。在今天的 Rust 中，需要一个 dyn 类型来为 trait 中的每个关联类型指定值。例如，考虑简化的 `Iterator` trait：

```rust,ignore
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

这个 trait 是 dyn 安全的，但是如果你实践中上有一个 `dyn`，那么你必须编写类似于 `dyn Iterator<Item = u32>` 的代码。`impl Iterator for dyn Iterator` 像这样：

```rust,ignore
impl<T> Iterator for dyn Iterator<Item = T> {
    type Item = T;
    
    fn next(&mut self) -> Option<T> {
        let next_fn = /* get next function from vtable */;
        return next_fn(self);
    }
}
```

现在你可以明白为什么我们要求所有关联类型都是 `dyn` 类型的一部分 —— 它允许我们编写一个完整的 impl（即包含每个关联类型的一个值）。

## 结论

在这篇文章涵盖了很多背景：

* 静态分发 vs 动态分发， vtables
* dyn safety 的起源和“部分” dyn safe 的可能性
* `impl Trait for dyn Trait`
