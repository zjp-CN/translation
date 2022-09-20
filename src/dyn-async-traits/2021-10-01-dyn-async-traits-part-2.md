---

## layout: post
title: Dyn async traits, part 2
date: 2021-10-01 11:56 -0400

## 版面：帖子标题：Dyn Async特征第二部分日期：2021-10-01 11：56-0400

In the \[previous post\], we uncovered a key challenge for `dyn` and async traits: the fact that, in Rust today, `dyn` types have to specify the values for all associated types. This post is going to dive into more background about how dyn traits work today, and in particular it will talk about where that limitation comes from.

在上一篇文章中，我们揭示了`dyn`和异步特征的一个关键挑战：在今天的Rust中，`dyn`类型必须指定所有相关类型的值。这篇帖子将深入探讨Dyn特征如何在当今发挥作用的更多背景，特别是它将讨论这种限制来自哪里。

\[previous post\]: {{ site.baseurl }}/blog/2021/09/30/dyn-async-traits-part-1/

\[上一篇文章]：{{site.base url}}/BLOG/2021/09/30/dyn-async-特征-Part-1/

### Today: Dyn traits implement the trait

### 今天：Dyn特征实现了特征

In Rust today, assuming you have a “dyn-safe” trait `DoTheThing `, then the type `dyn DoTheThing ` implements `Trait`. Consider this trait:

在今天的Rust中，假设你有一个“dyn-safe”特征`DoTheThing`，那么类型`dyn DoTheThing`就实现了`Trait`。考虑一下这一特点：

```rust
trait DoTheThing {
	fn do_the_thing(&self);
}

impl DoTheThing for String {
    fn do_the_thing(&self) {
        println!(“{}”, self);
    }
}
```

And now imagine some generic function that uses the trait:

现在想象一下使用特征的一些通用函数：

```rust
fn some_generic_fn<T: ?Sized + DoTheThing>(t: &T) {
	t.do_the_thing();
}
```

Naturally, we can call `some_generic_fn` with a `&String`, but — because `dyn DoTheThing` implements `DoTheThing` — we can also call `some_generic_fn` with a `&dyn DoTheThing`:

当然，我们可以用`&String`来调用`ome_Generic_fn`，但是--因为`dyn DoTheThing`实现了`DoTheThing`--我们也可以用`&dyn DoTheThing`来调用`ome_Generic_fn`：

```rust
fn some_nongeneric_fn(x: &dyn DoTheThing) {
    some_generic_fn(x)
}
```

### Dyn safety, a mini retrospective

### Dyn Safe，迷你回顾展

Early on in Rust, we debated whether `dyn DoTheThing` ought to implement the trait `DoTheThing` or not. This was, indeed, the origin of the term “dyn safe” (then called “object safe”). At the time, I argued in favor of the current approach: that is, creating a binary property. Either the trait was dyn safe, in which case `dyn DoTheThing` implements `DoTheThing`, or it was not, in which case `dyn DoTheThing` is not a legal type. I am no longer sure that was the right call.

在Rust的早期，我们讨论了`dyn DoTheThing`是否应该实现`DoTheThing`这个特征。这确实是术语“动态安全”(当时称为“对象安全”)的由来。当时，我支持当前的方法：即创建一个二进制属性。要么特征是dyn安全的，在这种情况下，`dyn DoTheThing`实现了`DoTheThing`；要么不是，在这种情况下，`dyn DoTheThing`不是合法类型。我不再确定这是不是正确的决定。

What I liked at the time was the idea that, in this model, whenever you see a type like `dyn DoTheThing`, you know that you can use it like any other type that implements `DoTheThing`. 

我当时喜欢的想法是，在这个模型中，每当您看到像`dyn DoTheThing`这样的类型时，您就知道可以像实现`DoTheThing`的任何其他类型一样使用它。

Unfortunately, in practice, the type `dyn DoTheThing` is not comparable to a type like `String`. Notably, `dyn` types are not sized, so you can’t pass them around by value or work with them like strings. You must instead always pass around some kind of *pointer* to them, such as a `Box<dyn DoTheThing>` or a `&dyn DoTheThing`. This is “unusual” enough that we make you *opt-in* to it for generic functions, by writing `T: ?Sized`. 

遗憾的是，在实践中，`dyn DoTheThing`类型无法与`String`这样的类型相比较。值得注意的是，`dyn`类型没有大小，因此您不能通过值传递它们，也不能像字符串一样使用它们。相反，您必须始终传递指向它们的某种指针，例如`Box<dyn DoTheThing>`或`&dyn DoTheThing`。这是足够“不寻常”的，我们让你选择加入它的泛型功能，通过写`t：？sized`。

What this means is that, in practice, generic functions don’t accept `dyn` types “automatically”, you have to design *for* dyn explicitly. So a lot of the benefit I envisioned didn’t come to pass.

这意味着，在实践中，泛型函数并不“自动”接受“dyn`类型”，您必须显式地为dyn进行设计。因此，我预想的很多好处都没有实现。

### Static versus dynamic dispatch, vtables

### 静态分派与动态分派、vables

Let’s talk for a bit about dyn safety and where it comes from. To start, we need to explain the difference between *static dispatch* and *virtual (dyn) dispatch*. Simply put, static dispatch means that the compiler knows which function is being called, whereas dyn dispatch means that the compiler doesn’t know. In terms of the CPU itself, there isn’t much difference. With static dispatch, there is a “hard-coded” instruction that says “call the code at this address”[^link]; with dynamic dispatch, there is an instruction that says “call the code whose address is in this variable”. The latter can be a bit slower but it hardly matters in practice, particularly with a successful prediction.

让我们来谈谈Dyn的安全以及它的来源。首先，我们需要解释静态调度和虚拟(动态)调度之间的区别。简单地说，静态分派意味着编译器知道正在调用哪个函数，而动态分派意味着编译器不知道。就CPU本身而言，没有太大区别。在静态分派中，有一条“硬编码”指令说“在这个地址调用代码”；在动态分派中，有一条指令说“调用地址在这个变量中的代码”。后者可能会慢一点，但在实践中几乎无关紧要，特别是在预测成功的情况下。

[^link]: Modulo dynamic linking.

模数动态链接。

When you use a `dyn` trait, what you actually have is a *vtable*. You can think of a vtable as being a kind of struct that contains a collection of function pointers, one for each method in the trait. So the vtable type for the `DoTheThing` trait might look like (in practice, there is a bit of extra data, but this is close enough for our purposes):

当你使用一个‘dyn’特征时，你实际拥有的是一个vtable。您可以将vtable看作是一种结构，它包含一组函数指针，每个指针对应于特征中的每个方法。因此，`DoTheThing`特征的vtable类型可能如下所示(实际上，有一些额外的数据，但对于我们的目的来说已经足够接近了)：

```rust
struct DoTheThingVtable {
    do_the_thing: fn(*mut ())
}
```

Here the `do_the_thing` method has a corresponding field. Note that the type of the first argument *ought* to be `&self`, but we changed it to `*mut ()`. This is because the whole idea of the vtable is that you don’t know what the `self` type is, so we just changed it to “some pointer” (which is all we need to know).

这里的`do_the_thing`方法有一个对应的字段。注意，第一个参数的类型应该是`&self`，但我们将其改为`*mut()`。这是因为vtable的整个概念是您不知道‘self’类型是什么，所以我们只将其更改为“一些指针”(这是我们需要知道的全部内容)。

When you create a vtable, you are making an instance of this struct that is tailored to some particular type. In our example, the type `String` implements `DoTheThing`, so we might create the vtable for `String` like so:

当您创建vtable时，您是在创建此结构的一个实例，该实例是为某个特定类型量身定做的。在我们的示例中，类型`String`实现了`DoTheThing`，所以我们可以为`String`创建vtable，如下所示：

```rust
static Vtable_DoTheThing_String: &DoTheThingVtable = &DoTheThingVtable {
    do_the_thing: <String as DoTheThing>::do_the_thing as fn(*mut ())
    //            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //            Fully qualified reference to `do_the_thing` for strings
};
```

You may have heard that a `&dyn DoTheThing` type in Rust is a *wide pointer*. What that means is that, at runtime, it is actually a pair of *two* pointers: a data pointer and a vtable pointer for the `DoTheThing` trait. So `&dyn DoTheThing` is roughly equivalent to:

你可能听说过Rust中的`&dyn DoTheThing`类型是一个宽指针。这意味着，在运行时，它实际上是一对两个指针：一个数据指针和一个用于`DoTheThing`特性的vtable指针。所以`&dyn DoTheThing`大致相当于：

```
(*mut (), &’static DoTheThingVtable)
```

When you cast a `&String` to a `&dyn DoTheThing`, what actually happens at runtime is that the compiler takes the `&String` pointer, casts it to `*mut ()`, and pairs it with the appropriate vtable. So, if you have some code like this:

当您将`&String`转换为`&dyn DoTheThing`时，在运行时实际发生的是编译器获取`&String`指针，将其转换为`*mut()`，并将其与相应的vtable配对。因此，如果您有一些如下所示的代码：

```rust
let x: &String = &”Hello, Rustaceans”.to_string();
let y: &dyn DoTheThing = x;
```

It winds up “desugared” to something like this:

它最后“去糖”成了这样：

```rust
let x: &String = &”Hello, Rustaceans”.to_string();
let y: (*mut (), &’static DoTheThingVtable) = 
    (x as *mut (), Vtable_DoTheThing_String);
```

### The dyn impl

### 动态实施

We’ve seen how you create wide pointers and how the compiler represents vtables. We’ve also seen that, in Rust, `dyn DoTheThing` implements `DoTheThing`. You might wonder how that works. Conceptually, the compiler generates an impl where each method in the trait is implemented by extracting the function pointer from the vtable and calling it:

我们已经了解了如何创建宽指针以及编译器如何表示vtable。我们还看到，在Rust中，`dyn DoTheThing`实现了`DoTheThing`。你可能想知道这是怎么回事。从概念上讲，编译器生成一个impl，其中特征中的每个方法都是通过从vtable中提取函数指针并调用它来实现的：

```rust
impl DoTheThing for dyn DoTheThing {
    fn do_the_thing(self: &dyn DoTheThing) {
        // Remember that `&dyn DoTheThing` is equivalent to
        // a tuple like `(*mut (), &’static DoTheThingVtable)`:
        let (data_pointer, vtable_pointer) = self;

        let function_pointer = vtable_pointer.do_the_thing;
        function_pointer(data_pointer);
    }
}
```

In effect, when we call a generic function like `some_generic_fn` with `T = dyn DoTheThing`, we monomorphize that call exactly like any other type. The call to `do_the_thing` is dispatched against the impl above, and it is *that special impl* that actually does the dynamic dispatch. Neat.

实际上，当我们使用`t=dyn DoTheThing`调用一个泛型函数时，就像调用任何其他类型一样，将该调用单元化。对`do_the_thing`的调用针对上面的Iml进行分派，而实际执行动态分派的正是那个特殊的Iml。干净利落。

### Static dispatch permits monomorphization

### 静态调度允许单一化

Now that we’ve seen how and when vtables are constructed, we can talk about the rules for dyn safety and where they come from. One of the most basic rules is that a trait is only dyn-safe if it contains no generic methods (or, more precisely, if its methods are only generic over lifetimes, not types). The reason for this rule derives directly from how a vtable works: when you construct a vtable, you need to give a single function pointer for each method in the trait (or, perhaps, a finite set of function pointers). The problem with generic methods is that there is no single function pointer for them: you need a different pointer for each type that they’re applied to. Consider this example trait, `PrintPrefixed`:

现在我们已经了解了vtable是如何以及何时构建的，接下来我们可以讨论dyn安全的规则以及它们来自哪里。最基本的规则之一是，特征只有在不包含泛型方法的情况下才是动态安全的(或者，更准确地说，如果它的方法只在生存期内是泛型的，而不是类型)。此规则的原因直接源于vtable的工作方式：当您构造vtable时，需要为特征中的每个方法(或者，可能是有限的函数指针集)提供单个函数指针。泛型方法的问题是它们没有单一的函数指针：您需要为应用它们的每种类型使用不同的指针。考虑这个示例特征，`PrintPrefix`：

```rust
trait PrintPrefixed {
    fn prefix(&self) -> String;
    fn apply<T: Display>(&self, t: T);
}

impl PrintPrefixed for String {
    fn prefix(&self) -> String {
        self.clone()
    }
    fn apply<T: Display>(&self, t: T) {
        println!(“{}: {}”, self, t);
    }
}
```

What would a vtable for `String as PrintPrefixed` look like? Generating a function pointer for `prefix` is no problem, we can just use `<String as PrintPrefixed>::prefix`. But what about `apply`? We would have to include a function pointer for `<String as PrintPrefixed>::apply<T>`, but we don’t know yet what the `T` is!

\`字符串作为PrintPrefix`的vtable会是什么样子？为`prefix`生成函数指针是没有问题的，我们只需使用`<字符串作为PrintPrefix>：：prefix`即可。但“应用”又如何呢？我们必须为`<字符串as PrintPrefix>：：Apply<T>`包括一个函数指针，但我们还不知道`T`是什么！

In contrast, with static dispatch, we don’t have to know what `T` is until the point of call. In that case, we can generate just the copy we need.

相比之下，使用静态调度时，我们在调用点之前不必知道`T‘是什么。在这种情况下，我们可以生成所需的副本。

### Partial dyn impls

### 部分动态隐含

The previous point shows that a trait can have *some* methods that are dyn-safe and some methods that are not. In current Rust, this makes the entire trait be “not dyn safe”, and this is because there is no way for us to write a complete `impl PrintPrefixed for dyn PrintPrefixed`:

前面的一点说明了特征可以有一些动态安全的方法，也可以有一些不是动态安全的方法。在Current Rust中，这使得整个特征是不安全的，这是因为我们没有办法编写一个完整的`impl PrintPrefix for dyn PrintPrefix`：

```rust
impl PrintPrefixed for dyn PrintPrefixed {
    fn prefix(&self) -> String {
        // For `prefix`, no problem:
        let prefix_fn = /* get prefix function pointer from vtable */;
        prefix_fn(…);
    }
    fn apply<T: Display>(&self, t: T) {
        // For `apply`, we can’t handle all `T` types, what field to fetch?
        panic!(“No way to implement apply”)
    }
}
```

Under the alternative design that was considered long ago, we could say that a `dyn PrintPrefixed` value is always legal, but `dyn PrintPrefixed` only implements the `PrintPrefixed` trait if all of its methods (and other items) are dyn safe. Either way, if you had a `&dyn PrintPrefixed`, you could call `prefix`. You just wouldn’t be able to use a `dyn PrintPrefixed` with generic code like `fn foo<T: ?Sized + PrintPrefixed>`.

在很久以前考虑的替代设计中，我们可以说`dyn PrintPrefix‘值总是合法的，但是只有当它的所有方法(和其他项)都是dyn安全的时，`dyn PrintPrefix`才实现`PrintPrefix’特性。无论哪种方式，如果你有一个`&dyn PrintPrefix`，你可以调用`prefix`。您只是不能将`dyn PrintPrefix`与`fn foo<T：？SIZE+PrintPrefix>`这样的泛型代码一起使用。

(We’ll return to this theme in future blog posts.)

(我们将在未来的博客文章中回到这个主题。)

If you’re familiar with the “special case” around trait methods that require `where Self: Sized`, you might be able to see where it comes from now. If a method has a `where Self: Sized` requirement, and we have an impl for a type like `dyn PrintPrefixed`, then we can see that this impl could never be called, and so we can omit the method from the impl (and vtable) altogether. This is awfully similar to saying that `dyn PrintPrefixed` is always legal, because it means that there only a subset of methods that can be used via virtual dispatch. The difference is that `dyn PrintPrefixed: PrintPrefixed` still holds, because we know that generic code won’t be able to call those “non-dyn-safe” methods, since generic code would have to require that `T: ?Sized`.

如果您熟悉需要“Where Self：Sized”的特征方法的“特殊情况”，您可能会明白它是从哪里来的。如果一个方法有一个`where self：siized‘要求，而我们对`dyn PrintPrefix`这样的类型有一个Iml，那么我们可以看到这个Iml永远不能被调用，所以我们可以从Iml(和vtable)中完全省略该方法。这与`dyn PrintPrefix`总是合法的说法非常相似，因为这意味着只有一部分方法可以通过虚拟分派使用。不同之处在于`dyn PrintPrefix：PrintPrefix`仍然有效，因为我们知道泛型代码将无法调用那些“非dyn-Safe”方法，因为泛型代码必须要求`t：？siized`。

### Associated types and dyn types

### 关联类型和动态类型

We began this saga by talking about associated types and `dyn` types. In Rust today, a dyn type is required to specify a value for each associated type in the trait. For example, consider a simplified `Iterator` trait:

我们从讨论关联类型和‘dyn’类型开始这一传奇故事。在今天的Rust中，需要一个dyn类型来为特征中的每个关联类型指定值。例如，考虑简化的`迭代器‘特征：

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

This trait is dyn safe, but if you actually have a `dyn` in practice, you would have to write something like `dyn Iterator<Item = u32>`. The `impl Iterator for dyn Iterator` looks like:

这个特征是dyn安全的，但是如果您实际上有一个`dyn‘，那么您必须编写类似于`dyn Iterator<Item=u32>`的代码。‘Iml Iterator for Dyn Iterator’如下所示：

```
impl<T> Iterator for dyn Iterator<Item = T> {
    type Item = T;
    
    fn next(&mut self) -> Option<T> {
        let next_fn = /* get next function from vtable */;
        return next_fn(self);
    }
}
```

Now you can see why we require all the associated types to be part of the `dyn` type — it lets us write a complete impl (i.e., one that includes a value for each of the associated types).

现在您可以明白为什么我们要求所有关联类型都是`dyn`类型的一部分--它允许我们编写一个完整的IMPL(即，包含每个关联类型的一个值)。

### Conclusion

### 结论

We covered a lot of background in this post:

我们在这篇文章中涵盖了很多背景：

* Static vs dynamic dispatch, vtables
* The origin of dyn safety, and the possibility of “partial dyn safety”
* The idea of a synthesized `impl Trait for dyn Trait`