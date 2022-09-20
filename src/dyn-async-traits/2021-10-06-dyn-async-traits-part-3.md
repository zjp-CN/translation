---

## layout: post
title: Dyn async traits, part 3
date: 2021-10-06 11:06 -0400

## 版面：帖子标题：Dyn Async特征第三部分日期：2021-10-06 11：06-0400

In the previous "dyn async traits" posts, I talked about how we can think about the compiler as synthesizing an impl that performed the dynamic dispatch. In this post, I wanted to start explore a theoretical future in which this impl was written manually by the Rust programmer. This is in part a thought exercise, but it’s also a possible ingredient for a future design: if we could give programmers more control over the “impl Trait for dyn Trait” impl, then we could enable a lot of use cases.

在之前的“dyn async特征”帖子中，我谈到了如何将编译器看作是合成了执行动态分派的Impl。在这篇文章中，我想开始探索一种理论上的未来，在这种情况下，这个Impl是由Rust程序员手动编写的。这在一定程度上是一种思考练习，但它也是未来设计的一个可能因素：如果我们能让程序员更好地控制“为Dyn特征实施Iml特征”，那么我们就可以启用许多用例。

### Example

### 示例

For this post, `async fn` is kind of a distraction. Let’s just work with a simplified `Iterator` trait:

在这篇文章中，`async fn‘有点让人分心。让我们只使用一个简化的`迭代器‘特征：

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

As we discussed in the previous post, the compiler today generates an impl that is something like this:

正如我们在上一篇文章中所讨论的，编译器今天生成的Impl类似于以下内容：

```rust
impl<I> Iterator for dyn Iterator<Item = I> {
    type Item = I;
    fn next(&mut self) -> Option<I> {
        type RuntimeType = ();
        let data_pointer: *mut RuntimeType = self as *mut ();
        let vtable: DynMetadata = ptr::metadata(self);
        let fn_pointer: fn(*mut RuntimeType) -> Option<I> =
            __get_next_fn_pointer__(vtable);
        fn_pointer(data)
    }
}
```

This code draws on the APIs from [RFC 2580](https://rust-lang.github.io/rfcs/2580-ptr-meta.html), along with a healthy dash of “pseduo-code”. Let’s see what it does:

这段代码使用了来自RFC 2580的API，以及一小段健康的“pseduo-code”。让我们来看看它有什么作用：

#### Extracting the data pointer

#### 提取数据指针

```rust
type RuntimeType = ();
let data_pointer: *mut RuntimeType = self as *mut ();
```

Here, `self` is a wide pointer of type `&mut dyn Iterator<Item = I>`. The rules for `as` state that casting a wide pointer to a thin pointer drops the metadata[^ugh], so we can (ab)use that to get the data pointer. Here I just gave the pointer the type `*mut RuntimeType`, which is an alias for `*mut ()` — i.e., raw pointer to something. The type alias `RuntimeType` is meant to signify “whatever type of data we have at runtime”. Using `()` for this is a hack; the “proper” way to model it would be with an existential type. But since Rust doesn’t have those, and I’m not keen to add them if we don’t have to, we’ll just use this type alias for now.

这里，`self`是`&mut dyn Iterator<Item=i>`类型的宽指针。`as‘的规则规定，将宽指针转换为细指针会丢弃元数据，因此我们可以(Ab)使用它来获取数据指针。在这里，我只给了指针类型`*mut RounmeType`，这是`*mut()`的别名--即指向某个对象的原始指针。类型别名`RounmeType`是为了表示“我们在运行时拥有的任何类型的数据”。使用`()`来实现这一点是一种技巧；对它进行建模的“正确”方法应该是使用存在主义类型。但由于Rust没有这些，如果没有必要的话，我不想添加它们，所以我们现在只使用这种类型的别名。

[^ugh]: I don’t actually like these rules, which have bitten me a few times. I think we should introduce an accessor function, but I didn’t see one in [RFC 2580](https://rust-lang.github.io/rfcs/2580-ptr-meta.html) — maybe I missed it, or it already exists.

我其实并不喜欢这些规则，它们已经咬了我几次。我认为我们应该引入一个存取器函数，但我在RFC 2580中没有看到一个--可能是我错过了它，或者它已经存在。

#### Extracting the vtable (or `DynMetadata`)

#### 解压vtable(或`dyMetadata`)

```rust
let vtable: DynMetadata = ptr::metadata(self);
```

The [`ptr::metadata`](https://doc.rust-lang.org/std/ptr/fn.metadata.html) function was added in [RFC 2580](https://rust-lang.github.io/rfcs/2580-ptr-meta.html). Its purpose is to extract the “metadata” from a wide pointer. The type of this metadata depends on the type of wide pointer you have: this is determined by the [`Pointee`](https://doc.rust-lang.org/std/ptr/trait.Pointee.html) trait[^noreferent]. For `dyn` types, the metadata is a [`DynMetadata`](https://doc.rust-lang.org/std/ptr/struct.DynMetadata.html), which just means “pointer to the vtable”. In today’s APIs, the [`DynMetadata`](https://doc.rust-lang.org/std/ptr/struct.DynMetadata.html) is pretty limited: it lets you extract the size/alignment of the underlying `RuntimeType`, but it doesn’t give any access to the actual function pointers that are inside.

RFC 2580中增加了`ptr：：metadata`函数。其目的是从宽指针中提取“元数据”。该元数据的类型取决于您拥有的宽指针的类型：这由`Pointe`特性决定。对于`dyn`类型，元数据是`dyMetadata`，意思是“指向vtable的指针”。在当今的API中，`dyMetadata`非常有限：它允许您提取底层`RounmeType`的大小/对齐，但它不提供对内部实际函数指针的任何访问。

[^ynoreferent]: I wish that this was called “referent”. Ever since I learned that word a few years ago I have found it so elegant. But I admit that `Pointee` is both more obvious and kind of hilarious, and besides “referent” seems to simply *references* (which I consider to be safe things like `&` or `&mut`, as distinguished from mere *pointers*).

我希望这被称为“参照物”。自从我几年前学会这个词以来，我就发现它是如此优雅。但我承认`Pointe‘更明显，也更搞笑，此外，“Referent”似乎只是引用(我认为这是安全的东西，如`&`或`&mu`，区别于单纯的指针)。

#### Extracting the function pointer from the vtable

#### 从vtable中提取函数指针

```rust
let fn_pointer: fn(*mut RuntimeType) -> Option<I> = 
    __get_next_fn_pointer__(vtable);
```

Now we get to the pseudocode. *Somehow*, we need a way to get the fn pointer out from the vtable. At runtime, the way this works is that each method has an assigned offset within the vtable, and you basically do an array lookup; kind of like `vtable.methods()[0]`, where `methods()` returns a array `&[fn()]` of function pointers. The problem is that there’s a lot of “dynamic typing” going on here: the signature of each one of those methods is going to be different. Moreover, we’d like some freedom to change how vtables are laid out. For example, the ongoing (and awesome!) work on dyn upcasting by [Charles Lew](https://github.com/crlf0710) has required modifying our [vtable layout](https://rust-lang.github.io/dyn-upcasting-coercion-initiative/design-discussions/vtable-layout.html), and I expect further modification as we try to support `dyn` types with multiple traits, like `dyn Debug + Display`.

现在我们来看看伪代码。无论如何，我们需要一种从vtable中取出fn指针的方法。在运行时，这种方法的工作方式是每个方法在vtable中都有一个分配的偏移量，您基本上是进行数组查找；有点像`vable.Methods()[0]`，其中`方法()`返回函数指针的数组`&[fn()]`。问题是这里有很多“动态类型”：每个方法的签名将是不同的。此外，我们希望有一些自由来改变vtable的布局方式。例如，正在进行的(而且很棒！)Charles Lew的Dyn向上转换工作需要修改我们的vtable布局，我预计会有进一步的修改，因为我们试图支持具有多个特征的`dyn‘类型，比如`dyn Debug+Display`。

So, for now, let’s just leave this as pseudocode. Once we’ve finished walking through the example, I’ll return to this question of how we might model `__get_next_fn_pointer__` in a forwards compatible way.

因此，现在，让我们将其保留为伪代码。一旦我们完成了示例的演练，我将返回到如何以向前兼容的方式对`__GET_NEXT_FN_POINTER__`建模的问题。

One thing worth pointing out: the type of `fn_pointer` is a `fn(*mut RuntimeType) -> Option<I>`. There are two interesting things going on here:

需要指出的是，`fn_pointer`的类型是一个`fn(*mut RounmeType)->选项<i>`。这里有两件有趣的事情：

* The argument has type `*mut RuntimeType`: using the type alias indicates that this function is known to take a single pointer (in fact, it’s a reference, but those have the same layout). This pointer is expected to point to the same runtime data that `self` points at — we don’t know what it is, but we know that they’re the same. This works because `self` paired together a pointer to some data of type `RuntimeType` along with a vtable of functions that expect `RuntimeType` references.[^ub]
* The return type is `Option<I>`, where `I` is the item type: this is interesting because although we don’t know statically what the `Self` type is, we *do* know the `Item` type. In fact, we will generate a distinct copy of this impl for every kind of item. This allows us to easily pass the return value.

[^ub]: If you used unsafe code to pair up a random pointer with an unrelated vtable, then hilarity would ensue here, as there is no runtime checking that these types line up.

参数的类型为`*mut RounmeType`：使用类型别名表示该函数采用单个指针(实际上，它是一个引用，但它们具有相同的布局)。这个指针应该指向`self‘指向的相同的运行时数据--我们不知道它是什么，但我们知道它们是相同的。这是因为`self`将一个指向类型为`RounmeType`的数据的指针与一个期望引用`RunmeType`的函数的vtable配对在一起。返回类型为`Option<i>`，其中`I`是项类型：这很有趣，因为虽然我们不静态地知道`Self`类型是什么，但我们知道`Item`类型。事实上，我们将为每一种物品生成一个不同副本。这使我们可以轻松地传递返回值。如果您使用不安全代码将一个随机指针与一个不相关的vtable配对，那么这里将会非常有趣，因为没有运行时检查这些类型是否对齐。

#### Calling the function

#### 调用该函数

```rust
fn_pointer(data)
```

The final line in the code is very simple: we call the function! It returns an `Option<I>` and we can return that to our caller.

代码中的最后一行非常简单：我们调用函数！它返回一个`Option<i>`，我们可以将其返回给我们的调用者。

### Returning to the pseudocode

### 返回到伪代码

We relied on one piece of pseudocode in that imaginary impl:

在那个想象中的Impl中，我们依赖于一段伪代码：

```rust
let fn_pointer: fn(*mut RuntimeType) -> Option<I> = 
    __get_next_fn_pointer__(vtable);
```

So how could we possibly turn `__get_next_fn_pointer__` from pseudocode into real code? There are two things worth noting:

那么，我们如何才能将`__GET_NEXT_FN_POINTER__`从伪代码变成真正的代码呢？有两件事值得注意：

* First, the name of this function already encodes the method we want (`next`). We probably don’t want to generate an infinite family of these “getter” functions.
* Second, the signature of the function is specific to the method we want, since it returns a `fn` type(`fn *mut RuntimeType) -> Option<I>`) that encodes the signature for `next` (with the self type changed, of course). This seems better than just returning a generic signature like `fn()` that must be cast manually by the user; less opportunity for error.

### Using zero-sized fn types as the basis for an API

### 首先，这个函数的名称已经编码了我们想要的方法(`next`)。第二，该函数的签名特定于我们想要的方法，因为它返回一个`fn`类型(`fn*mut Rty meType)->选项<i>`)，该类型对`next`的签名进行编码(当然，带有更改的self类型)。这似乎比仅仅返回必须由用户手动强制转换的泛型签名(如`fn()`)要好得多；出错的可能性更小。使用零大小的fn类型作为API的基础

One way to solve these problems would be to build on the trait system. Imagine there were a type for every method, let’s call it `A`, and that this type implemented a trait like `AssociatedFn`:

解决这些问题的一种方法是建立在特质体系之上。假设每个方法都有一个类型，让我们将其称为`A`，并且该类型实现了一个类似`AssociatedFn`的特征：

```rust
trait AssociatedFn {
    // The type of the associated function, but as a `fn` pointer
    // with the self type erased. This is the type that would be
    // encoded in the vtable.
    type FnPointer;

    … // maybe other things
}
```

We could then define a generic “get function pointer” function like so:

然后，我们可以定义一个通用的“Get Function Points”函数，如下所示：

```rust
fn associated_fn<A>(vtable: DynMetadata) -> A::FnPtr
where
    A: AssociatedFn
```

Now instead of `__get_next_fn_pointer__`, we can write 

现在，不是`__GET_NEXT_FN_POINTER__`，我们可以写

```rust
type NextMethodType =  /* type corresponding to the next method */;
let fn_pointer: fn(*mut RuntimeType) -> Option<I> = 
   associated_fn::<NextMethodType>(vtable);
```

Ah, but what is this `NextMethodType`? How do we *get* the type for the next method? Presumably we’d have to introduce some syntax, like `Iterator::item`.

啊，但是这个`NextMethodType`是什么呢？我们如何获取下一个方法的类型？想必我们必须引入一些语法，比如`Iterator：：item`。

### Related concept: zero-sized fn types

### 相关概念：零尺寸FN类型

This idea of a type for associated functions is *very close* (but not identical) to an already existing concept in Rust: zero-sized function types. As you may know, the type of a Rust function is in fact a special zero-sized type that uniquely identifies the function. There is (presently, anyway) no syntax for this type, but you can observe it by printing out the size of values ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e30569b1f9e4a36e436b7335627dd1ba)):

这种关联函数的类型思想与Rust中已经存在的一个概念非常接近(但不完全相同)：零大小函数类型。正如您可能知道的，Rust函数的类型实际上是唯一标识该函数的特殊的零大小类型。(目前，无论如何)没有这种类型的语法，但您可以通过打印值的大小(操场)来观察它：

```rust
fn foo() { }

// The type of `f` is not `fn()`. It is a special, zero-sized type that uniquely
// identifies `foo`
let f = foo;
println!(“{}”, sizeof_value(&f)); // prints 0

// This type can be coerced to `fn()`, which is a function pointer
let g: fn() = f;
println!(“{}”, sizeof_value(&g)); // prints 8
```

There are also types for functions that appear in impls. For example, you could get an instance of the type that represents the `next` method on `vec::IntoIter<u32>` like so:

也有出现在Ims中的函数的类型。例如，您可以在`vec：：IntoIter<u32>`上获取表示`next`方法的类型的实例，如下所示：

```rust
let x = <vec::IntoIter<u32> as Iterator>::next;
println!(“{}”, sizeof_value(&f)); // prints 0
```

### Where the zero-sized types don’t fit

### 其中零大小的类型不适合

The existing zero-sized types can’t be used for our “associated function” type for two reasons:

现有的零大小类型不能用于我们的“关联函数”类型，原因有两个：

* You can’t name them! We can fix this by adding syntax.
* There is no zero-sized type for a *trait function independent of an impl*.

The latter point is subtle[^blog]. Before, when I talked about getting the type for a function from an impl, you’ll note that I gave a fully qualified function name, which specified the `Self` type precisely:

你不能说出他们的名字！我们可以通过添加语法来解决这个问题。特征函数不存在独立于隐式的零大小类型。后一点很微妙。在前面，当我谈到从IMPL获取函数的类型时，您会注意到我给出了一个完全限定的函数名，它精确地指定了`Self‘类型：

[^blog]: And, in fact, I didn’t see it until I was writing this blog post!

事实上，直到我写这篇博客文章我才看到它！

```rust
let x = <vec::IntoIter<u32> as Iterator>::next;
//       ^^^^^^^^^^^^^^^^^^ the Self type
```

But what we want in our impl is to write code that doesn’t know what the Self type is! So this type that exists in the Rust type system today isn’t quite what we need. But it’s very close.

但我们在实现中想要的是编写不知道self类型是什么的代码！因此，今天铁锈类型系统中存在的这种类型并不完全是我们所需要的。但已经很接近了。

### Conclusion

### 结论

I’m going to leave it here. Obviously, I haven’t presented any kind of final design, but we’ve seen a lot of tantalizing ingredients:

我要把它放在这里。显然，我还没有展示任何形式的最终设计，但我们已经看到了很多诱人的成分：

* Today, the compiler generates a `impl Iterator for dyn Iterator` that extract functions from a vtable and invokes them by magic.
* But, using the APIs from [RFC 2580](https://rust-lang.github.io/rfcs/2580-ptr-meta.html), you can *almost* write the by hand. What is missing is a way to extract a function pointer from a vtable, and what makes *that* hard is that we need a way to identify the function we are extracting
* We have zero-sized types that represent functions today, but we don’t have a way to name them, and we don’t have zero-sized types for functions in traits, only in impls.

Of course, all of the stuff I wrote here was just about normal functions. We still need to circle back to async functions, which add a few extra wrinkles. Until next time!

如今，编译器会生成一个‘impl Iterator for dyn Iterator’，它从vtable中提取函数并神奇地调用它们。但是，使用RFC 2580中的API，您几乎可以手工编写。我们缺少一种从vtable中提取函数指针的方法，而让这一点变得困难的是，我们需要一种方法来识别我们正在提取的函数。我们有零大小的类型来表示今天的函数，但我们没有方法来命名它们，我们也没有特征中的函数的零大小类型，只有在IMPLE中。当然，我在这里写的所有东西都是关于普通函数的。我们仍然需要返回到异步函数，这会增加一些额外的问题。下次见！

### Footnotes

### 脚注