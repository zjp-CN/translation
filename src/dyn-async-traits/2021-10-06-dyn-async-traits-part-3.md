# Dyn Async Traits

[原文](https://smallcultfollowing.com/babysteps/blog/2021/10/06/dyn-async-traits-part-3/) |
日期：2021-10-06 11:06 -0400

在前一篇文章中，我谈到了编译器是如何执行动态分发的 impl。

在这篇文章中，我想开始探索一种理论上的 future，其 impl 是由 Rust 程序员手动编写的。

这在一定程度上是一种思考练习，但它也是设计 future 的一个可能因素：如果我们能让程序员更好地控制 `impl Trait for dyn Trait`，那么就可以启用许多用例。

## 示例

在这篇文章中， `async fn` 会有点让人分心，所以我们只使用一个简化的 `Iterator` trait：

```rust,ignore
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

正如在上一篇文章中所讨论的，编译器今天生成类似于以下内容的 impl：

```rust,ignore
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

这段代码使用了来自 [RFC 2580] 的 API，以及少量的伪代码。看看它有什么作用：

[RFC 2580]: https://rust-lang.github.io/rfcs/2580-ptr-meta.html

### 提取数据指针

```rust,ignore
type RuntimeType = ();
let data_pointer: *mut RuntimeType = self as *mut ();
```

Here, `self` is a wide pointer of type `&mut dyn Iterator<Item = I>`. The rules for `as` state that casting a wide pointer to a thin pointer drops the metadata[^ugh], so we can (ab)use that to get the data pointer. Here I just gave the pointer the type `*mut RuntimeType`, which is an alias for `*mut ()` — i.e., raw pointer to something. The type alias `RuntimeType` is meant to signify "whatever type of data we have at runtime". Using `()` for this is a hack; the "proper" way to model it would be with an existential type. But since Rust doesn` t have those, and I` m not keen to add them if we don` t have to, we` ll just use this type alias for now.

这里，`self` 是 `&mut dyn Iterator<Item = I>` 类型的宽指针。`as` 的规则规定，将宽指针转换为细指针会丢弃 metadata[^ugh]，因此我们可以使用/滥用它来获取数据指针。

这里，我给指针的类型是 `*mut RuntimeType`，这是 `*mut()` 的别名 —— 即指向某个对象的原始指针。

类型别名 `RuntimeType` 是为了表示“在运行时拥有的任何类型的数据”。使用 `()` 来实现这一点是一种技巧；对它进行建模的"正确"方法应该是使用 existential type。

但由于 Rust 没有这些，没有必要的话，我不想添加它们，所以现在只使用这种类型别名。

[^ugh]: 我其实并不喜欢这些规则，它们已经折腾了我几次。我认为应该引入一个存取器函数 (accessor function)，但我在
        [RFC 2580] 中没有看到 —— 可能是我错过了它，或者它已经存在。

### 提取 vtable（或 `DynMetadata`）

```rust,ignore
let vtable: DynMetadata = ptr::metadata(self);
```

[`ptr::metadata`]: https://doc.rust-lang.org/std/ptr/fn.metadata.html
[`Pointee`]: https://doc.rust-lang.org/std/ptr/trait.Pointee.html
[`DynMetadata`]: https://doc.rust-lang.org/std/ptr/struct.DynMetadata.html

[RFC 2580] 中增加了 [`ptr::metadata`] 函数。其目的是从宽指针中提取“元数据”。

该元数据的类型取决于所拥有的宽指针的类型：这由 [`Pointee`] trait 决定[^noreferent]。对于 `dyn` 类型，元数据是 [`DynMetadata`]，意思是“指向 vtable 的指针”。

在现在的 API 中，[`DynMetadata`] 非常有限：它允许你提取底层 `RuntimeType` 的大小/对齐，但不提供对内部实际函数指针的任何访问。

[^noreferent]: 我希望这被称为“referent”。自从我几年前学会这个词以来，我就发现它是如此优雅。但我承认 `Pointee`
               更显而易见，也更有趣，此外，“referent”似乎只是引用（我认为这是安全的东西，如 `&` 或 `&mut`，区别于单纯的指针 pointers）。

### 从 vtable 中提取函数指针

```rust,ignore
let fn_pointer: fn(*mut RuntimeType) -> Option<I> = 
    __get_next_fn_pointer__(vtable);
```

[Charles Lew]: https://github.com/crlf0710
[vtable 布局]: https://rust-lang.github.io/dyn-upcasting-coercion-initiative/design-discussions/vtable-layout.html

现在来看看伪代码。无论如何，我们需要一种从 vtable 中取出函数指针的方法。

这种方法在运行时的工作方式是每个方法在 vtable 中都有一个被分配的偏移量，基本上是进行数组查找；有点像 `vtable.methods()[0]`，其中 `methods()`
返回函数指针的数组 `&[fn()]`。

问题是有很多“动态类型”：每个方法的签名将是不同的。此外，我们希望有一些自由来改变 vtable 的布局方式。例如，[Charles Lew] 正在进行的（而且很棒的）的
dyn upcasting 需要修改 [vtable 布局]，预计还会有进一步的修改，因为我们试图支持具有多个 trait 的 `dyn` 类型，比如 `dyn Debug + Display`。

因此，现在，让我们将其保留为伪代码。一旦完成了示例的演练，我将返回到如何以向前兼容的方式对 `__get_next_fn_pointer__` 建模的问题。

需要指出的是，`fn_pointer` 的类型是一个 `fn(*mut RuntimeType) -> Option<I>`。这里有两件有趣的事情：

* 参数的类型为 `*mut RuntimeType`：使用类型别名表示该函数采用单指针（实际上，它是一个引用，但它们具有相同的布局）。这个指针应该指向的运行时数据与
  `self` 所指向的一样 —— 我们不知道它是什么，但只知道它们是相同的。这是因为 `self` 把一个指向类型为 `RuntimeType` 的数据的指针与一个期望引用 `RuntimeType`
  的一些函数的 vtable 放在一起。[^ub]
* 返回类型为 `Option<I>`，其中 `I` 是条目类型 (item type)：这很有趣，因为虽然我们不是静态地知道 `Self` 类型是什么，但知道 `Item` 类型。事实上，
  我们将为每一种条目生成一个不同副本，这使可以轻松地传递返回值。

[^ub]: 如果你使用不安全代码将一个随机指针与一个不相关的 vtable 放在一起，那么这里将会非常有趣，因为没有运行时检查这些类型是否对齐。

### 调用该函数

```rust,ignore
fn_pointer(data)
```

代码中的最后一行非常简单：调用这个函数！它返回一个 `Option<I>`，我们可以将其返回给调用者。

## 返回到伪代码

在那个想象中的 impl 中，我们依赖于一段伪代码：

```rust,ignore
let fn_pointer: fn(*mut RuntimeType) -> Option<I> = 
    __get_next_fn_pointer__(vtable);
```

那么，我们如何才能将 `__get_next_fn_pointer__` 从伪代码变成真正的代码呢？有两件事值得注意：

* 该函数的名称已经编码了我们想要的方法 `next`，我们或许不想生成一组无限的 `getter` 函数。
* 该函数的签名特定于我们想要的方法，因为它返回一个 `fn` 类型 `fn(*mut RuntimeType -> Option<I>`，该类型对 `next` 的签名进行了编码（当然，带有修改过的
  self 类型）。这似乎比仅仅返回必须由用户手动强制转换的泛型签名（如 `fn()`）要好得多；出错的可能性更小。

## 使用零大小的 `fn` 类型作为 API 的基础

解决这些问题的一种方法建立在 trait 体系之上。假设每个方法都有一个类型 `A`，并且该类型实现了类似于 `AssociatedFn` 的 trait：

```rust,ignore
trait AssociatedFn {
    // The type of the associated function, but as a `fn` pointer
    // with the self type erased. This is the type that would be
    // encoded in the vtable.
    type FnPointer;

    … // maybe other things
}
```

然后，定义一个通用的获取函数指针的函数，如下所示：

```rust,ignore
fn associated_fn<A>(vtable: DynMetadata) -> A::FnPtr
where
    A: AssociatedFn
```

现在还不是 `__get_next_fn_pointer__`，而是

```rust,ignore
type NextMethodType =  /* type corresponding to the next method */;
let fn_pointer: fn(*mut RuntimeType) -> Option<I> = 
   associated_fn::<NextMethodType>(vtable);
```

这个 `NextMethodType` 是什么呢？我们如何获取 `next` 方法的类型？想必必须引入一些语法，比如 `Iterator::Item`。

## 相关概念：零大小 `fn` 类型

这种关联函数类型的思想与 Rust 中已经存在的一个概念非常接近（但不完全相同）：零大小函数类型。

正如你可能知道的，Rust 的函数实际上是唯一标识该函数的特殊的零大小类型。（目前）没有这种类型的语法，但你可以通过[打印值的大小来观察它][playground]：

[playground]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e30569b1f9e4a36e436b7335627dd1ba

```rust
fn foo() { }

// The type of `f` is not `fn()`. It is a special, zero-sized type that uniquely
// identifies `foo`
let f = foo;
println!("{}", sizeof_value(&f)); // prints 0

// This type can be coerced to `fn()`, which is a function pointer
let g: fn() = f;
println!("{}", sizeof_value(&g)); // prints 8
```

也有出现在 impl 中的函数类型。例如，你可以在 `vec::IntoIter<u32>` 上获取表示 `next` 方法的类型实例，如下所示：

```rust,ignore
let x = <vec::IntoIter<u32> as Iterator>::next;
println!("{}", sizeof_value(&f)); // prints 0
```

## 零大小类型并不合适

现有的零大小类型不能用于我们的关联函数类型，原因有两个：

* 你不能说出他们的名字！可以通过添加语法来解决这个问题。
* trait 函数不存在独立于 impl 的零大小类型。

后一点很微妙[^blog]。在前面例子中，当我谈到从 impl 获取函数的类型时，你会注意到我给出了一个完全限定的函数名，它精确地指定了 `Self` 类型：

[^blog]: 事实上，直到我写这篇博客文章我才看到它！

```rust,ignore
let x = <vec::IntoIter<u32> as Iterator>::next;
//       ^^^^^^^^^^^^^^^^^^ the Self type
```

但我们在 impl 中想要的是编写不知道 Self 类型是什么的代码！因此，今天 Rust 类型系统中存在的这种类型并不完全是我们所需要的。但已经很接近了。

## 结论

显然，我还没有展示任何形式的最终设计，但我们已经看到了很多诱人的内容：

* 如今，编译器会生成一个 `impl Iterator for dyn Iterator` ，它从 vtable 中提取函数并神奇地调用它们。
* 但是，使用 [RFC 2580] 中的API，你几乎可以手工编写。我们缺少的是一种从 vtable 中提取函数指针的办法，而让这一点变得困难的是，需要一种办法来识别正在提取的函数。
* 我们有零大小的类型来表示今天的函数，但没有办法来命名它们，我们也没有只在 impl 中的 trait 中的函数的零大小类型。

当然，我在这里写的所有东西都是关于普通的函数。我们仍然需要返回到异步函数，这会增加一些额外的问题。下次见！
