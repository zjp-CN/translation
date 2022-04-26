* Feature Name: nll
* Start Date: 2017-08-02
* RFC PR: [rust-lang/rfcs#2094](https://github.com/rust-lang/rfcs/pull/2094)
* Rust Issue: [rust-lang/rust#43234](https://github.com/rust-lang/rust/issues/43234)

> 原文：[RFC 2094: NLL](https://rust-lang.github.io/rfcs/2094-nll.html)

# 摘要

扩展 Rust 的借用系统以支持非词法生命周期 (non-lexical lifetimes)
—— 这些生命周期基于<abbr title="control-flow graph">控制流图</abbr>，而不是<abbr title="lexical scopes">词法范围</abbr>。

此 RFC 详细描述了如何推断这些新的、更灵活的区域，并描述了如何调整错误消息。
此外，本 RFC 还描述了对<abbr title="borrow checker">借用检查程序</abbr>的一些其他扩展，其总体效果是消除了许多常见情况，在这些情况下，为了通过借用检查，需要对局部代码进行小的功能修改。（附录描述了本 RFC 一些剩下未解决的借用检查限制。）

# 动机

## 什么是生命周期？

借用检查器的基本思想是：值 (values) 在借用时可能不会发生变化或移动。

但我们如何知道值是否被借用？

其实很简单：无论何时创建借用，编译器都会为生成的引用分配一个生命周期 (lifetime)。

这个生命周期相当于引用可被使用的代码<abbr title="span">范围</abbr>。编译器将推断这个生命周期是它
**可以拥有的最小范围的生命周期，且包含所有使用的引用**。

请注意，Rust 以一种非常特殊的方式使用“生命周期”一词。

在日常用语中，Rust 的生命周期一词可以两种不同但相似的方式使用：

1. 引用的生命周期：使用该引用的<abbr title="the span of time">时间范围</abbr>。
2. 值的生命周期：该值被释放之前（或者换句话说，在值的析构函数运行之前）的时间范围。

第二个时间范围非常重要，它描述了一个值在多长时间范围内有效。为了区分两者，我们将第二个时间范围称为<abbr title="value's scope">值的作用域</abbr>[^span-scope]。

当然，生命周期和作用域是相互关联的。

具体来说，如果引用某个值，该引用的生命周期不能超过该值的作用域。否则，引用将指向已释放的内存。

[^span-scope]: 本文将 `span` 翻译成“范围”，将 `scope` 翻译成“作用域”。

为了更好地了解生命周期和作用域之间的区别，让我们考虑一个简单的例子。

在下面的代码案例中，Vec `data` 被（可变地）借用，从而引用被传递给函数 `capitalize`。由于 `capitalize`
不会返回引用，因此该借用的生命周期将仅限于该函数调用。相比之下，`data`
的作用域要大得多，其范围相当于 fn 主体的后部，从 `let` 一直延伸到封闭作用域结束。

```rust
fn foo() {
    let mut data = vec!['a', 'b', 'c']; // --+ 'scope
    capitalize(&mut data[..]);          //   |
//  ^~~~~~~~~~~~~~~~~~~~~~~~~ 'lifetime //   |
    data.push('d');                     //   |
    data.push('e');                     //   |
    data.push('f');                     //   |
} // <---------------------------------------+

fn capitalize(data: &mut [char]) {
    // do something
}
```

这个例子还展示了其他一些东西。如今 Rust 的生命周期比作用域灵活得多（如果没有我们希望得那样灵活，那么这就是本 RFC 要做的）：

* 作用域通常相当于某个<abbr title="block">块</abbr>（或者更具体地说，是从 `let` 延伸到封闭作用域结束的块的后部）。[scope-block]
* 相反，生命周期也可能跨越单个表达式，如本例所示。在本例中，借用的生命周期仅限于对 `capitalize` 
  的调用，而不延展到块的其余部分。这就是为什么下面的 `data.push` 调用是合法的。

只要引用只在一个语句中使用，现在的生命周期通常就足够了。然而，当引用跨越多个语句时，就会出现问题。

在这种情况下，编译器要求生命周期是包含这两条语句的最里面的表达式（通常是一个块），并且通常比实际需要或期望的要大得多。

[scope-block]: 作用域总是与块相对应，但有一个例外：临时值的作用域有时是封闭语句。

让我们来看一些问题案例。稍后，我们将看到非词法生命周期如何修复这些情况。

## 问题案例 #1：分配给变量的引用

一个常见的问题是引用被分配到变量中。考虑前面例子中的这个微小的变化：切片 `&mut data[..]`
不会直接传递给 `capitalize`，而是存储在一个局部变量中

```rust
fn bar() {
    let mut data = vec!['a', 'b', 'c'];
    let slice = &mut data[..]; // <-+ 'lifetime
    capitalize(slice);         //   |
    data.push('d'); // ERROR!  //   |
    data.push('e'); // ERROR!  //   |
    data.push('f'); // ERROR!  //   |
} // <------------------------------+
#fn capitalize(data: &mut [char]) {
#    // do something
#}
```

按照编译器当前（此 RFC 提出时）的工作方式，把引用分配给变量意味着它的生命周期必须与该变量的整个范围一样大。

在这种情况下，这意味着现在生命周期一直延长到块的末尾。这反过来意味着对 `data.push`
的调用现在会出错[^#1-error]，因为它们发生在 `slice` 的生命周期内。这是合乎逻辑的，但很烦人。

[^#1-error]: 现在实现了此 RFC，因此不会出错。后面案例 2 也是一样。

在这种情况下，你可以通过将 `slice` 放入自己的块中来解决问题：

```rust
fn bar() {
    let mut data = vec!['a', 'b', 'c'];
    {
        let slice = &mut data[..]; // <-+ 'lifetime
        capitalize(slice);         //   |
    } // <------------------------------+
    data.push('d'); // OK
    data.push('e'); // OK
    data.push('f'); // OK
}
#fn capitalize(data: &mut [char]) {
#    // do something
#}
```

由于我们引入了一个新的块，`slice` 
的作用域变小了，因此产生的生命周期也变小了。引入这样一个块是一种人为的、不是一个完全显而易见的解决方案。

## 问题案例 #2：条件控制流

另一个常见的问题发生在引用仅用于一个给定的匹配分支（或者更一般地说，一个控制流路径）时。

这种情况最常见于 map 数据结构。考虑以下函数，给定某个 `key`，如果它存在，则在 `map[key]` 中找到值，否则插入默认值：

```rust,ignore
fn process_or_default() {
    let mut map = ...;
    let key = ...;
    match map.get_mut(&key) { // -------------+ 'lifetime
        Some(value) => process(value),     // |
        None => {                          // |
            map.insert(key, V::default()); // |
            //  ^~~~~~ ERROR.              // |
        }                                  // |
    } // <------------------------------------+
}
```

这段代码今天不会编译。原因是 `map` 作为 `get_mut`
调用的一部分被借用，而且借用不仅必须包括对 `get_mut` 的调用，还必须包括匹配的 `Some`
分支。包含这两个表达式的最里面的表达式是匹配本身（如上所述），因此借用被认为一直延伸到匹配结束。

不幸的是，匹配不仅包含 `Some` 分支，还包含 `None` 分支，因此当我们在 `None`
分支中对 map 插入元素时，我们会得到一个错误，即 `map` 仍然是借用的。

这个特殊的例子相对容易解决。在许多情况下，可以将 `None` 的代码从 `match` 中移出，如下所示：

```rust
fn process_or_default1() {
    let mut map = ...;
    let key = ...;
    match map.get_mut(&key) { // -------------+ 'lifetime
        Some(value) => {                   // |
            process(value);                // |
            return;                        // |
        }                                  // |
        None => {                          // |
        }                                  // |
    } // <------------------------------------+
    map.insert(key, V::default());
}
```

这样调整代码之后，调用的 `map.insert`
不是匹配的一部分，因此它不是借用的一部分。虽然这是可行的，但不幸的是，需要像在上一个示例中人工引入块那样操作。

## 问题案例 #3：跨函数的条件控制流

虽然我们能够以一种相对简单（尽管令人恼火）的方式解决问题案例 
\#2，但条件控制流的其他形式却无法如此轻松地解决。当从函数中返回引用时尤其如此。

考虑下面的函数，如果 key 存在，它返回一个 key 的值，否则插入一个新的值（出于本节的目的，假设不存在 map 的 `entry` API）：

```rust
#use std::hash::Hash;
#use std::collections::HashMap;
fn get_default<'r,K:Hash+Eq+Copy,V:Default>(map: &'r mut HashMap<K,V>,
                                            key: K)
                                            -> &'r mut V {
    match map.get_mut(&key) { // -------------+ 'r
        Some(value) => value,              // |
        None => {                          // |
            map.insert(key, V::default()); // |
            //  ^~~~~~ ERROR               // |
            map.get_mut(&key).unwrap()     // |
        }                                  // |
    }                                      // |
}                                          // v
```

乍一看，这段代码似乎与我们之前看到的代码非常相似，如前一样，它不会编译通过。

事实上，这里的生命周期是完全不同的。原因是，在 `Some` 分支中，值被返回给调用者。由于 `value` 是 map
中的一个引用，这意味着 `map` 将一直被借用，直到调用那方的某个时刻（确切地说是 `'r`
这个时刻）。为了对这个生命周期参数 `'r` 所代表的含义有更好的直觉，请考虑一个假设的 `get_default`
调用方，生命周期 `'r` 表示调用方将使用这个引用的代码范围：

```rust,ignore
fn caller() {
    let mut map = HashMap::new();
    ...
    {
        let v = get_default(&mut map, key); // -+ 'r
          // +-- get_default() -----------+ //  |
          // | match map.get_mut(&key) {  | //  |
          // |   Some(value) => value,    | //  |
          // |   None => {                | //  |
          // |     ..                     | //  |
          // |   }                        | //  |
          // +----------------------------+ //  |
        process(v);                         //  |
    } // <--------------------------------------+
    ...
}
```

如果我们尝试与上一个示例中相同的解决方案，我们会发现它不起作用：

```rust
#use std::hash::Hash;
#use std::collections::HashMap;
fn get_default1<'r,K:Hash+Eq+Copy,V:Default>(map: &'r mut HashMap<K,V>,
                                             key: K)
                                             -> &'r mut V {
    match map.get_mut(&key) { // -------------+ 'r
        Some(value) => return value,       // |
        None => { }                        // |
    }                                      // |
    map.insert(key, V::default());         // |
    //  ^~~~~~ ERROR (still)                  |
    map.get_mut(&key).unwrap()             // |
}                                          // v
```

之前的 `value` 的生命周期限制在匹配语句，但这个新的生命周期扩展到调用方，因此借用不会因为我们退出匹配而结束。

因此，当我们试图在匹配后调用 `insert` 时，它仍然在范围内。

这个问题的解决方法有点复杂。它依赖于借用检查器使用函数的精确控制流来确定哪些借用在范围内这一事实。

```rust
#use std::hash::Hash;
#use std::collections::HashMap;
fn get_default2<'r,K:Hash+Eq+Copy,V:Default>(map: &'r mut HashMap<K,V>,
                                             key: K)
                                             -> &'r mut V {
    if map.contains_key(&key) {
    // ^~~~~~~~~~~~~~~~~~ 'n
        return match map.get_mut(&key) { // + 'r
            Some(value) => value,        // |
            None => unreachable!()       // |
        };                               // v
    }

    // 此刻，`map.get_mut` 从未被调用！
    // （恰恰相反，它的结果作为函数返回给调用方，不再此函数内使用了）
    map.insert(key, V::default()); // OK now.
    map.get_mut(&key).unwrap()
}
```

这里改动的是，把 `map.get_mut` 调用移到了一个 `if` 表达式中，然后一切就绪，这样进入 `if`
后就可以无条件地返回。这意味着借用从 `get_mut` 那一刻开始，一直持续到调用方的 
`'r`，但是借用检查器可以看到，该借用甚至不会在 `if` 之外开始。它不会考虑在调用 `map.insert` 那处的借用。

这种解决方法比其他方法更麻烦，因为生成的代码在运行时的效率实际上更低，因为它必须进行多次查找。

值得注意的是，Rust 的 hashmaps 包含了一个 `entry`
API，现在可以用来实现这个函数。生成的代码阅读起来更好，甚至比原始版本更高效，因为它也避免了在 key 不存在时进行额外的查找：

```rust
#use std::hash::Hash;
#use std::collections::HashMap;
fn get_default3<'r,K:Hash+Eq,V:Default>(map: &'r mut HashMap<K,V>,
                                        key: K)
                                        -> &'r mut V {
    map.entry(key)
       .or_insert_with(|| V::default())
}
```

无论如何，除了 `HashMap` 之外，其他数据结构也存在这个问题。即使在实践中最好使用 `entry`
API，但如果原始代码通过了借用检查器其实就挺好的。（有趣的是，借用检查器的局限性是开发 `entry` API 的最初动机之一！）

## 问题案例 #4：改变 `&mut` 引用

当前借用检查器禁止在引用对象（`*x`）被借用时重新分配变量 `x` 的一个 `&mut`。

这种情况最常见于编写一个逐步“遍历”数据结构的循环时。考虑以下函数，它将链表 `&mut List<T>` 转换成 `Vec<&mut T>`：

```rust
struct List<T> {
    value: T,
    next: Option<Box<List<T>>>,
}

fn to_refs<T>(mut list: &mut List<T>) -> Vec<&mut T> {
    let mut result = vec![];
    loop {
        result.push(&mut list.value);
        if let Some(n) = list.next.as_mut() {
            list = n;
        } else {
            return result;
        }
    }
}
```

如果试图编译它，我们会得到一个错误（实际上，我们会得到多个错误）：

```
error[E0506]: cannot assign to `list` because it is borrowed
  --> /Users/nmatsakis/tmp/x.rs:11:13
   |
9  |         result.push(&mut list.value);
   |                          ---------- borrow of `list` occurs here
10 |         if let Some(n) = list.next.as_mut() {
11 |             list = n;
   |             ^^^^^^^^ assignment to borrowed `list` occurs here
```

具体地说，问题在于我们借用了这个 `list.value`（或者更明确地说，`(*list).value`）。

当前借用检查器强制执行一条规则：借用一条路径时，不能将其指定给该路径或该路径的任何前缀。

此时这意味着你不能分配给以下任何一项：

* `(*list).value`
* `*list`
* `list`

因此，`list = n` 赋值行为被禁止。这些规则在某些情况下是有意义的（例如，如果 `list` 的类型是 `List<T>`，而不是
`&mut list<T>`，那么覆盖 `list` 也会覆盖 `list.value`），但在越过可变引用的情况下就没有意义了。

如 [Issue #10520](https://github.com/rust-lang/rust/issues/10520) 所述，该问题存在各种解决方法。一个技巧是将 `&mut` 
引用移动到一个不需要修改的临时变量中：

```rust
#struct List<T> {
#    value: T,
#    next: Option<Box<List<T>>>,
#}
fn to_refs<T>(mut list: &mut List<T>) -> Vec<&mut T> {
    let mut result = vec![];
    loop {
        let list1 = list;
        result.push(&mut list1.value);
        if let Some(n) = list1.next.as_mut() {
            list = n;
        } else {
            return result;
        }
    }
}
```

以这种方式构建程序时，借用检查器会看到 `(*list1).value` （而不是 `list`）被借用。这并不妨碍我们以后分配到 `list`。

显然，这种变通方法很烦人。事实证明，这里的问题并不特定于非词法生命周期本身。

相反，借用检查器在借用路径时执行的规则过于严格，并且没有考虑借用引用中固有的间接性。本 RFC 提出了一个解决方案。

## 解决方案的大致轮廓

本 RFC 提出了一种更灵活的生命周期模型。以前的生命周期是基于抽象语法树的，现在我们提出了通过控制流图定义的生命周期。

更具体地说，生命周期将基于编译器内部使用的 [MIR](https://blog.rust-lang.org/2016/04/19/MIR.html) 来推导。

在新的方案中，引用的生命周期仅适用于以后可能使用引用的函数部分（从编译器的角度来说，那里的引用是活动的）。

这可以解决连续几条语句（如问题案例 \#1）到更复杂的 —— 比如在问题案例 #2 中只涉及一个匹配分支而不涉及其他分支 —— 的情况。

然而，为了成功深入我们想要解决的全部示例，我们必须更进一步，而不仅仅是将生命周期更改为控制流图的一部分。

在进行<abbr title="subtyping">子类型</abbr>检查时，我们还必须考虑位置。

这与当今编译器的工作方式完全不同，现在在编译器中，子类型关系是“绝对的”。也就是说，在当前的编译器中，只要 `'a` 比 `'b`
活得更久（`'a: 'b`），那么类型 `&'a ()` 是类型 `&'b ()` 的一个子类型。这意味着
`'a` 相当于占据函数的更多部分。

根据这个建议，子类型可以在特定的点 P 处建立。此时，`'a` 的生命周期必须只比从 P 处可以到达的 `'b` 的生命周期长。

本 RFC 中的想法已以[原型形式](https://github.com/nikomatsakis/nll)实现。

该原型包括一个简化的控制流图，它允许创建可能出现的各种区域约束，并实现区域推理算法，然后解决这些约束。

# 详细设计

## 分层设计

我们用以下几个“层面”来描述设计：

1. 首先，我们将描述一个基本设计，重点是一个函数内的控制流。
2. 接下来，我们扩展控制流图以更好地处理无限循环。
3. 然后，我们将设计扩展到处理 dropck，特别是 RFC 1327 引入的 `#[may_dangle]` 属性。
4. 然后，我们将扩展设计以考虑命名的生命周期参数，如问题案例 3 中的那些。
5. 最后，我们对借用检查器进行简要描述。

## 第 0 层：定义

在描述设计之前，我们必须定义将要使用的术语。

本 RFC 是根据 MIR 的简化版本定义的，为了不引入复杂性，省略了各种细节。

> **Lvalues**.
> MIR 的 lvalue 是指向内存位置的路径。完整的 MIR lvalues 
> 是[通过 Rust enum 定义](https://github.com/rust-lang/rust/blob/bf0a9e0b4d3a4dd09717960840798e2933ec7568/src/librustc/mir/mod.rs#L839-L851)
> ，并包含许多<abbr title="knobs">节</abbr>，其中大多数与本 RFC 无关。

lvalue 的简化形式：

```
LV = x       // 局部变量
   | LV.f    // 访问字段
   | *LV     // 解引用
```

`*` 的优先级较低，因此 `*a.b.c` 将解引用 `a.b.c`；如果只写 `a`，你可以写 `(*a).b.c`。

> **Prefixes.** 
> lvalue 的前缀是去掉字段和解引用得到的所有 lvalue。`*a.b` 的前缀应该是 `*a.b`、`a.b` 和 `a`。

> **Control-flow graph.**
> MIR 被组织成一个
> [控制流图](https://en.wikipedia.org/wiki/Control_flow_graph)，而不是一个抽象的语法树。它是在编译器中通过转换
> “HIR”（高级 IR）来创建的。MIR CFG 由一组[基本块][basic block]组成。
> 每个基本块都有一系列[语句][statement]和一个[终止符][terminator]。

[basic block]: https://github.com/rust-lang/rust/blob/bf0a9e0b4d3a4dd09717960840798e2933ec7568/src/librustc/mir/mod.rs#L443-L463
[statement]: https://github.com/rust-lang/rust/blob/bf0a9e0b4d3a4dd09717960840798e2933ec7568/src/librustc/mir/mod.rs#L774-L814
[terminator]: https://github.com/rust-lang/rust/blob/bf0a9e0b4d3a4dd09717960840798e2933ec7568/src/librustc/mir/mod.rs#L465-L552

本 RFC 中涉及的语句分为三类：

* 像 `x = y` 的赋值语句。赋值的右侧被称作[rvalue]。不存在复合形式的 rvalue，因此每个语句都是一个会立即执行的离散的行为。\
  例如，Rust表达式 `a = b + c + d` 将被编译成两条 MIR 指令，比如 `tmp0 = b + c; a = tmp0 + d`。 
* `drop(lvalue)` 如果其中有值，则释放一个 lvalue；在极端情况下，这需要运行时检查（在 mir 中，一个称为 elaborate drops
  的过程执行此转换）
* `StorageDead(x)` 为` x` 取消栈存储分配。 LLVM 
  使用它们来允许栈分配的值使用相同的栈<abbr title="slot">插槽</abbr>（如果它们的活动存储范围不相交的话）。
  [Ralf Jung 最近的博客有更多细节](https://www.ralfj.de/blog/2017/06/06/MIR-semantics.html)。

[rvalue]: https://github.com/rust-lang/rust/blob/bf0a9e0b4d3a4dd09717960840798e2933ec7568/src/librustc/mir/mod.rs#L1037-L1071

## 第 1 层：函数内的控制流

### 运行示例

我们将参考下面这个示例 4 来解释设计。在介绍设计之后，我们将把它应用于三个问题案例，以及其他一些有趣的例子。

```rust,ignore
let mut foo: T = ...;
let mut bar: T = ...;
let mut p: &T;

p = &foo;
// (0)
if condition {
    print(*p);
    // (1)
    p = &bar;
    // (2)
}
// (3)
print(*p);
// (4)
```

本例的关键点是，变量 `foo` 只应在第 0 点和第 3 点被认为被借用，而不是在第
1 点借用。相比之下， `bar` 应在第 2 点和第 3 点被视为借用。它们在第 4 点，都不需要被认为是借用的，因为这里没有使用 `p`。

我们可以将这个示例转换为下面的控制流图。回想一下，MIR 中的控制流图由一列离散语句和尾随终止符的基本块组成：

```text
// let mut foo: i32;
// let mut bar: i32;
// let mut p: &i32;

A
[ p = &foo     ]
[ if condition ] ----\ (true)
       |             |
       |     B       v
       |     [ print(*p)     ]
       |     [ ...           ]
       |     [ p = &bar      ]
       |     [ ...           ]
       |     [ goto C        ]
       |             |
       +-------------/
       |
C      v
[ print(*p)    ]
[ return       ]
```

我们将使用诸如 `Block/Index` 这样的符号来表示控制流图中的特定语句或终止符。
`A/0` 和 `B/4` 分别指的是 `p=&foo` 和 `goto C`。

### 什么是生命周期？它如何与借用检查器交互？

首先，我们将生命周期视为**控制流图中的一个点集**；稍后在此 RFC 中，我们将扩展这些集合的域，以包括  "skolemized"
生命周期 —— 它对应于函数上声明的命名生命周期参数。如果生命周期包含点 P，这意味着该生命周期的引用在进入 P
时有效。生命周期出现在 MIR 表示中的不同位置：

* 变量（和临时变量等）的类型可能包含生命周期。
* 每个借用表达式都有一个指定的生命周期。

我们可以把示例 4 拓展成显式的生命周期名称。结果有三个生命周期：`'p`、`'foo`、`'bar`。

```rust,ignore
let mut foo: T = ...;
let mut bar: T = ...;
let mut p: &'p T;
//      --
p = &'foo foo;
//   ----
if condition {
    print(*p);
    p = &'bar bar;
    //   ----
}
print(*p);
```

`'p` 是变量 `p` 类型的一部分。它表示了控制流图中可以安全地解引用 `p` 
的部分。`'foo` 和 `'bar` 不一样：它们分别表示借用了 `foo` 和 `bar` 的生命周期。

像 `'foo` 和 `'bar` 这样与借用表达式关联的生命周期对借用检查器很重要。它们对应于控制流图中借用检查器将执行其限制的部分。

此时，由于两个借用都是共享借用（ `&` ），借用检查器将防止在 `'foo` 期间修改 `foo`，并防止在 `'bar`
期间修改 `bar`。如果这些是可变借用（ `&mut` ），借用检查器将在这些生命周期中阻止对 `foo` 和 `bar` 的所有访问。

人们可以对 `'foo` 和 `'bar` 做出许多有效的选择。但该 RFC
描述了一种推理算法，其目的是为可能工作的每个借用选择最小的使用寿命。这相当于施加我们所能施加的最少限制。

我们希望这个算法计算出 `'foo` 是 `{A/1, B/0, C/0}`，这明显排除了 B/1 到 B/4 的这些点。
`'bar` 应该被推断为集合 `{B/3, B/4, C/0}`。`'p` 会是 `'foo` 和 `'bar` 的并集，因为它包含变量 `p` 所有有效的点。

### 生命周期推断约束

推理算法通过分析 MIR 并创建一系列约束来工作。这些约束遵循以下语法：

```text
// 约束集合 C：
C = true
  | C, (L1: L2) @ P    // 生命周期 L1 在点 P 比生命周期 L2 更长

// 生命周期 L
L = 'a
  | {P}
```

这里最后的 `P` 代表控制流图中的一个点，记号 `'a` 指一些命名的生命周期推断变量（例如，`'p`、`'foo` 或 `'bar`）。

一旦创建了约束，推理算法就会处理这些约束，这通过定点迭代实现：
每个生命周期变量都以一个空集开始，然后在约束上迭代，不断增加生命周期，直到它们足够大，能够满足所有约束。

（把它与原型代码比对的话： `regionck.rs` 文件负责创建约束，而 `infer.rs` 负责处理约束。）

### 存活

理解 NLL 工作方式的一个关键因素是理解存活 (liveness)。术语“存活”来源于编译器分析，但它相当直观。

如果一个变量的当前值可以在以后使用，我们就说这个变量是存活的。这对于示例 4 非常重要：

```rust,ignore
let mut foo: T = ...;
let mut bar: T = ...;
let mut p: &'p T = &foo;
// `p` 在此存活：它的值可以在下一行使用
if condition {
    // `p` 在此存活：它的值可以在下一行使用
    print(*p);
    // `p` 在此死亡：它的值不会被使用了
    p = &bar;
    // `p` 在此存活：它的值可以在之后被使用
}
// `p` 在此存活：它的值可以在下一行使用
print(*p);
// `p` 在此死亡：它的值不会被使用了
```

这里的变量 `p`，它在程序的开头被赋值，然后在 `if` 期间被重新赋值。关键的一点是，
`p` 在重新分配之前会在范围内<abbr title="dead">死亡</abbr>（而不是存活）。即使变量 
`p` 会再次被使用，这也是正确的，因为 `p` 中的值将不会被使用。

传统的编译器基于变量计算是否存活，但我们希望计算生命周期的存活。

我们可以把基于变量的分析扩展到生命周期，如果有一个变量 `p` 存在于 P 点，那么生命周期 L 以 `p` 的形式出现，且存在于 P 点。（稍后，当我们讨论 dropck 
时，将对生命周期使用一个更严格的概念，在那个概念中，变量类型中的一些生命周期可能是活的，而另一些则不是活的。）

因此，上述示例 4 中，生命周期 `'p` 将与 `p`
在完全相同的时间点上运行。生命周期 `'foo` 和 `'bar` 没有（直接）存活的时间点，因为它们不会出现在任何变量的类型中。

然而，这并不意味着这些生命周期是无关紧要的；如下所示，后续分析引入的子类型约束最终将要求 `'foo` 和 `'bar`
比 `'p` 更长。

#### 基于存活的生命周期约束

我们生成的第一组约束来自与存活。具体来说，如果生命周期 L 在点 P 处有效，那么我们将引入如下约束：

```text
(L: {P}) @ P
```

（我们稍后讨论处理约束会看到，这个约束实际上只是将 `P` 插入 `L` 
的集合中。事实上，原型并不必具体化这些约束，而是直接将 `P` 插入 `L`。）

对于示例 4，这意味着我们将引入以下活动性约束：

```text
('p: {A/1}) @ A/1
('p: {B/0}) @ B/0
('p: {B/3}) @ B/3
('p: {B/4}) @ B/4
('p: {C/0}) @ C/0
```

### 子类型

每当引用从一个位置复制到另一个位置时，Rust 的子类型规则要求源引用的生命周期超过目标位置的生命周期。如前所述，在本 
RFC 中，我们将子类型的概念扩展为能够感知位置，这意味着我们要考虑复制值的位置。

例如，在点 A/0，示例 4 包含一个借用表达式 `p = &'foo foo`，此时，借用表达式将生成一个类型为 `&'foo T` 的引用，其中 `T` 是 
`foo` 类型。然后将该值分配给 `p`，其类型为 `&'p T`。因此，我们希望要求 `&'foo T` 是 `&'p T` 的一个子类型。

此外，这种关系需要保持在点 A/1 这个赋值发生的点 A/0 的后继点（这是因为 `p` 的新值首先在 
A/1 中可见）。我们将该子类型约束编写如下：

```text
(&'foo T <: &'p T) @ A/1
```

然后，Rust 标准的子类型规则（下面给出了两个示例）可以将该子类型规则“分解”为我们进行推理所需的生命周期约束：

```text
(T_a <: T_b) @ P
('a: 'b) @ P      // <-- 针对我们提出的推断算法的约束
------------------------
(&'a T_a <: &'b T_b) @ P

(T_a <: T_b) @ P
(T_b <: T_a) @ P  // (&mut T 是不变的)
('a: 'b) @ P      // <-- 另一个约束another constraint
------------------------
(&'a mut T_a <: &'b mut T_b) @ P
```

在示例 4 中，生成以下子类型约束：

```text
(&'foo T <: &'p T) @ A/1
(&'bar T <: &'p T) @ B/3
```

这些可以转换为以下生命周期约束：

```text
('foo: 'p) @ A/1
('bar: 'p) @ B/3
```

### 重新借用的约束

最后还有一个情况需要约束。我们经常会有一个借用表达式，它“重新借用”现有引用的引用对象：

```text
let x: &'x i32 = ...;
let y: &'y i32 = &*x;
```

此时，借用的生命周期 `'y` 与原引用的生命周期 `'x` 之间存在联系。特别是，`'x` 必须比 `'y` 更长（ `'x: 'y`
）。在这样的简单情况下，无论原始引用 `x` 是共享（ `&` ）还是可变（ `&mut`
）引用，关系都是相同的。然而，在涉及多个解引用的更复杂的情况下，处理方法是不同的。

**<abbr title="supporting prefix">支撑前缀</abbr>**。为了定义重新借用的约束，我们先介绍支撑前缀的思想，这个定义在一些地方很有用。

支撑 lvalue 的前缀是通过剥离字段和解引用做到的，除非我们在到达共享引用的解引用时停止。

直觉上，共享引用是不同的，因为它们实现了  `Copy`，所以人们总是可以将共享引用复制到一个临时变量，并获得一个等价的路径。

以下是一些支撑前缀的示例：

```rust,ignore
let r: (&(i32, i64), (f32, f64));

// The path (*r.0).1 has type `i64` and supporting prefixes:
// - (*r.0).1
// - *r.0

// The path r.1.0 has type `f32` and supporting prefixes:
// - r.1.0
// - r.1
// - r

let m: (&mut (i32, i64), (f32, f64));

// The path (*m.0).1 has type `i64` and supporting prefixes:
// - (*m.0).1
// - *m.0
// - m.0
// - m
```

**重新借用的约束**。考虑这样一个情况，我们有一个生命周期为 `'b` 的 lvalue `lv_b` 的借用（共享或可变）：

```rust,ignore
lv_l = &'b lv_b      // or:
lv_l = &'b mut lv_b
```

这时，计算 `lv_b` 的支撑前缀，并找到集合中的每个解引用 lvalue `*lv`，其中 `lv`
是具有生命周期 `'a` 的引用。然后添加一个约束 `('a: 'b) @ P`，其中 `P` 是借用之后的点（借用生效的点）。

来看一些例子。每个例子都会从原型实现链接到相应的测试文件。

[**例 1**](https://github.com/nikomatsakis/nll/blob/master/test/borrowck-read-variable-while-borrowed-indirect.nll)

为了理解为什么需要这个规则，先考虑这个单个引用简单的例子：

```rust,ignore
let mut foo: i32     = 22;
let r_a: &'a mut i32 = &'a mut foo;
let r_b: &'b mut i32 = &'b mut *r_a;
...
use(r_b);
```

`*r_a` 的支撑前缀是 `*r_a` 和 `r_a`（因为 `r_a` 是可变引用，所以我们递归）。

其中只有 `*r_a` 是一个解引用 lvalue，被解引用的引用 `r_a` 具有生命周期 `'a`。添加 `'a: 'b`
约束，从而确保只要使用 `r_b`，就认为 `foo` 是被借用的。如果没有这个约束，生命周期 `'a`
将在第二次借用之后结束，此时即使 `*r_b` 仍然可以用于访问 `foo`， `foo` 会被认为未借用。

[**例 2**](https://github.com/nikomatsakis/nll/blob/master/test/borrowck-write-variable-after-ref-extracted.nll)

现在考虑一个双重间接的情况：

```rust,ignore
let mut foo: i32     = 22;
let mut r_a: &'a i32 = &'a foo;
let r_b: &'b &'a i32 = &'b r_a;
let r_c: &'c i32     = &'c **r_b;
// 这里哪个被认为是借用？
use(r_c);
```

和前一个例子一样，重要的是，只要使用 `r_c`，就认为 `foo` 被借用。然而，变量 `r_a` 呢：它应该被借用吗？答案是否定的：一旦 
`r_c` 被初始化， `r_a` 的值就不再重要，（例如）即使 `foo` 仍然被借用，我们可以用一个新值覆盖 
`r_a`。这个结果与重新借用的规则不符：`**r_b` 的支撑路径只是 `**r_b`。我们不再添加任何路径，因为此路径已经是 `*r_b`
的解引用，而且 `*r_b` 具有（共享引用）类型 `&'a i32`。因此，我们将添加一个重新引用的约束：`'a: 'c`。此约束确保只要使用了
`r_c`，对 `foo` 的借用就仍然有效，但对 `r_a`（其生命周期为 `'b`）的借用可能会过期。

[**例 3**](https://github.com/nikomatsakis/nll/blob/master/test/borrowck-read-ref-while-referent-mutably-borrowed.nll)

上一个示例展示了共享引用的借用一旦被解引用后如何过期。然而，对于可变引用，这并不安全。考虑下面的例子：

```rust,editable
fn main() {
    let mut foo = 0;
    let mut p: &mut i32 = &mut foo;
    let q: &mut &mut i32 = &mut p;
    let r: &mut i32 = &mut **q;
    // use_(*p);  // <-- 这一行导致错误
    // use_(**q); // <-- 这一行导致错误
    use_(*r);
}
fn use_(_: i32) {}
```

这里的关键是，通过重新借用 `**q` 来创建一个引用 `r`，然后在程序的最后一行使用 `r`。使用 `r` 必须延长用于创建
`p` 和 `q` 的借用的生命周期。否则，你可以通过 `*r` 和 `*p`
访问（和改变）相同的内存。（事实上，真正的 rustc 在早期确实有一个与此类似的安全性缺陷。）

因为对可变引用解引用不会阻止枚举支撑前缀，所以 `**q` 的支撑前缀是 `**q`、`*q` 和 
`q`。因此，添加两个重新借用的约束：`'q: 'r` 和 `'p: 'r`，从而这两个借用确实在所讨论的行的范围内被考虑。

看待这个例子的另一种方法：为了创建可变引用 `p`，我们在 `foo`
上加一个“锁”（只要 `p` 在使用中，这个锁就会一直存在）。然后我们锁定可变引用 `p`
来创建 `q`；只要 `q` 还在使用，这把锁就必须一直有效。当我们借用 `**q` 来创建 `r` 时，这是最后一次直接使用 `q` —— 所以你可能认为因为 `q` 不再（直接）使用而可以解除对 `p` 的锁定。但是，这是不合理的，因为从那时起，`r` 和 `p`
都可以访问同一个内存。关键是要认识到， `r` 代表间接使用 `q` （而 `q` 又是间接使用
`p`），因此只要 `r` 在使用中， `p` 和 `q` 也必须被认为“在使用中”（因此它们的“锁”仍然有效）。

### 处理约束

一旦创建了约束，推理算法就会解决这些约束。这是通过定点迭代实现的：每个生命周期变量都从一个空集开始，然后迭代，不断增加生命周期，直到它们足够大，能够满足所有约束。

像 `('a: 'b) @P` 这样的约束的含义是，从 P 点开始，生命周期 `'a` 必须包括 `'b` 内的从 P 点可以到达的所有点。从
P 点开始进行[深度优先搜索][depth-first-search]来实现算法；如果退出生命周期
`'b`，搜索就会停止；否则，对于找到的每一点，将其添加到 `'a`。

[depth-first-search]: https://github.com/nikomatsakis/nll/blob/1cff361c9aeb6f553b528078866f5717f1872dad/nll/src/infer.rs#L71-L113

示例 4 中，全套约束是：

```text
('foo: 'p) @ A/1
('bar: 'p) @ B/3
('p: {A/1}) @ A/1
('p: {B/0}) @ B/0
('p: {B/3}) @ B/3
('p: {B/4}) @ B/4
('p: {C/0}) @ C/0
```

处理这些约束将的结果是以下生命周期，这正是我们所期望的答案：

```
'p   = {A/1, B/0, B/3, B/4, C/0}
'foo = {A/1, B/0, C/0}
'bar = {B/3, B/4, C/0}
```

### 为什么这个算法正确的直觉

为了使算法正确，我们必须保持一个关键的不变量。考虑在点 P 上用生命周期 L 借用某路径 H 来创建引用 
R；然后，在某个点 Q 处对该引用 R （或者 R 的复制品、或者 move 之后）解引用。

我们必须确保引用不会失效：这意味着在到达 Q 时，借用的内存一定没有被释放。如果引用 R
是一个共享引用（ `&T` ），那么内存也一定没有被写入数据（以 `UnsafeCell` 为模）。如果引用
R 是一个可变引用（ `&mut T` ），那么只能通过引用 R 访问内存。为了保证这些性质，必须防止可能影响
P（借用）和 Q（使用）之间所有点的借用内存的操作。

这意味着 L 必须至少包括 P 和 Q 之间的所有点。首先，考虑这种情况：通过借用创建的同一引用 R 在 Q 点进行访问：

```rust,ignore
R = &H; // point P
...
use(R); // point Q
```

变量 R 将存在于 P 和 Q 之间的所有点上。基于存活的规则足以满足这种情况：具体来说，因为
R 的类型包括生命周期 L，我们知道 L 必须包含 P 和 Q 之间的所有点，因为 R 存在于那里。

第二种情况是通过别名（或移动）访问 R 所引用的内存：

```rust,ignore
R = &H;  // point P
R2 = R;  // last use of R, point A
...
use(R2); // point Q
```

这时，仅靠存活规则是不够的。问题是赋值语句 `R2 = R` 很可能是 R 的最后一次使用，因此变量 R 在这一点上是死亡的。然而，
R 中的值稍后仍将（通过 R2 ）被解引用取消引用，因此我们希望生命周期 L 包含这些点。这就是子类型约束发挥作用的地方： R2
的类型包括一个生命周期 L2，赋值语句 `R2 = R` 将在 L 和 L2 之间建立一个 outlives 约束 `(L: L2) @ A`。此外，这个新变量
R2 必须介于赋值和最终使用之间（即沿着从 A 到 Q 的路径 ）。把这两个事实放在一起，我们看到 L 最终将包括从
P 到 A 的点（因为 R 存活）和从 A 到 Q 的点（因为子类型要求传播 R2 的存活）。

请注意，这些生命周期可能存在间隙。当同一变量被多次使用和覆盖时，可能会发生这种情况：

```rust,ignore
let R: &L i32;
let mut R2: &L2 i32;

R = &H1; // point P1
R2 = R;  // point A1
use(R2); // point Q1
...
R2 = &H2; // point P2
use(R2);  // point Q2
```

在本例中，R2 上的存活约束将确保 L2 （其类型中的生命周期）包括 Q1 和 Q2 （因为 R2
在这两个点上是存活的），而不包括 “…” 或点 P1 或 P2。注意， A1 处的子类型关系（ `(L: L2) @ A1)` ）确保了 L 包括
Q1，但不要求 L 包括 Q2（即使 L2 有点  Q2）。这是因为 R2 中 Q2 处的值不可能来自 A1 处的赋值；如果可以的话，那么要么
R2 必须在 A1 和 Q2 之间存活，要么它们受子类型约束。

### 其他例子

让我们从问题案例 #1 和 #2 开始（案例 #3 将在后面一节介绍命名的生命周期谈到）。

#### 问题案例 #1

翻译成 MIR，这个例子大致如下：

```rust,ignore
let mut data: Vec<i32>;
let slice: &'slice mut i32;
START {
    data = ...;
    slice = &'borrow mut data;
    capitalize(slice);
    data.push('d');
    data.push('e');
    data.push('f');
}
```

生成的约束条件如下：

```
('slice: {START/2}) @ START/2
('borrow: 'slice) @ START/2
```

因此， `'slice` 和 `'borrow` 都将被推断为 START/2，由此允许访问 START/3 中的 `data` 和以下语句。

#### 问题案例 #2

翻译成 MIR，这个例子大致如下（一些不相关的细节被省略）。请注意， `match` 语句为了测试 variant 被转换为一个 SWITCH
结构，并且为了从 `Some` variant 中提取内容被转化成一个 “downcast”（该操作是特定于 MIR
的，除了作为匹配的一部分之外，没有 Rust 的等价物）。

```
let map: HashMap<K,V>;
let key: K;
let tmp0: &'tmp0 mut HashMap<K,V>;
let tmp1: &K;
let tmp2: Option<&'tmp2 mut V>;
let value: &'value mut V;

START {
/*0*/ map = ...;
/*1*/ key = ...;
/*2*/ tmp0 = &'map mut map;
/*3*/ tmp1 = &key;
/*4*/ tmp2 = HashMap::get_mut(tmp0, tmp1);
/*5*/ SWITCH tmp2 { None => NONE, Some => SOME }
}

NONE {
/*0*/ ...
/*1*/ goto EXIT;
}

SOME {
/*0*/ value = tmp2.downcast<Some>.0;
/*1*/ process(value);
/*2*/ goto EXIT;
}

EXIT {
}
```

将生成以下存活约束：

```
('tmp0: {START/3}) @ START/3
('tmp0: {START/4}) @ START/4
('tmp2: {SOME/0}) @ SOME/0
('value: {SOME/1}) @ SOME/1
```

将生成以下基于子类型的约束：

```
('map: 'tmp0) @ START/3
('tmp0: 'tmp2) @ START/5
('tmp2: 'value) @ SOME/1
```

归根结底，我们最感兴趣的生命周期是 `'map`，它表示持续借用 `map` 的时间。处理上述约束，得到：

```
'map == {START/3, START/4, SOME/0, SOME/1}
'tmp0 == {START/3, START/4, SOME/0, SOME/1}
'tmp2 == {SOME/0, SOME/1}
'value == {SOME/1}
```

这表明， 在 `None` 分支中可以改变 `map`，也可以在 `Some` 分支中改变 `map`，但只能在调用 `process()` 之后（即从 Some/2 开始）。这是我们想要的结果。

#### 问题案例 #4：不变量

值得一看示例 4 的一个变体。这与之前的模式相同，但没有使用引用 `&'a T` ，而是使用引用 
`Foo<'a>`，它们相对于 `'a` 是不变的。这意味着， `Foo<'a>` 值中的生命周期 `'a` 
不能近似（也就是说，不能像正常引用那样缩短它）。通常不变性 (invariance) 是由于可变性而产生的（例如， `Foo<'a>` 可能有一个 
`Cell<&'a ()>` 类型的字段）。这里的关键是，不变性实际上对结果没有任何影响。这的确如此，因为基于位置的子类型。

```rust,ignore
let mut foo: T = ...;
let mut bar: T = ...;
let p: Foo<'a>;

p = Foo::new(&foo);
if condition {
    print(*p);
    p = Foo::new(&bar);
}
print(*p);
```

实际上，我们最终得到了与以前相同的约束，但以前只有
`'foo: 'p`/`'bar: 'p` 约束（由于子类型），现在我们还有 `'p: 'foo` 和 `'p: 'bar` 约束：

```
('foo: 'p) @ A/1
('p: 'foo) @ A/1
('bar: 'p) @ B/3
('p: 'bar) @ B/3
('p: {A/1}) @ A/1
('p: {B/0}) @ B/0
('p: {B/3}) @ B/3
('p: {B/4}) @ B/4
('p: {C/0}) @ C/0
```

关键的一点是，新的约束不会影响最终的答案：新的约束已经满足旧的答案。

#### vec-push-ref

在前几次迭代方案中，能够感知位置子类型规则被 SSA 形式等变换所取代。下面的示例展示了与这些方法相比，位置感知子类型的价值。

```rust,ignore
let foo: i32;
let vec: Vec<&'vec i32>;
let p: &'p i32;

foo = ...;
vec = Vec::new();
p = &'foo foo;
if true {
    vec.push(p);
} else {
    // 重点：`foo` 在此没有被借用
    use(vec);
}
```

可以将上面的内容转换为控制流图形式：

```
block START {
    v = Vec::new();
    p = &'foo foo;
    goto B C;
}

block B {
    vec.push(p);
    goto EXIT;
}

block C {
    // Key point: `foo` not borrowed here
    use(vec);
    goto EXIT;
}

block EXIT {
}
```

这里的存活关系是：

```
('vec: {START/1}) @ START/1
('vec: {START/2}) @ START/2
('vec: {B/0}) @ B/0
('vec: {C/0}) @ C/0
('p: {START/2}) @ START/2
('p: {B/0}) @ B/0
```

与此同时，`vec.push(p)` 建立了这种子类型关系：

```
('p: 'vec) @ B/1
('foo: 'p) @ START/2
```

处理结果是：

```
'vec = {START/1, START/2, B/0, C/0}
'p = {START/2, B/0}
'foo = {START/2, B/0}
```

这个例子的有趣之处在于，生命周期 `'vec` 必须包含 `if`
的两分支 —— 因为它在两个分支中都被使用 —— 但 `'vec` 
只在一条路径上与生命周期 `'p` 相关。因此，尽管 `'p` 必须比 `'vec` 长， 但是
`'p` 由于位置感知子类型的存在，永远不会包含 “else” 分支。

## 第 2 层：避免无限循环

之前的设计是根据“纯” MIR 控制流图来描述的。然而，使用原始图在无限循环方面会带来一些不受欢迎的特性。

在这种情况下，图没有出口 (exit) 
，这破坏了传统的反向分析定义，如存活。为了解决这个问题，当我们为函数构建控制流图时，将用额外的边来扩充它
—— 特别是，对于每个无限循环（ `loop {}` ），将添加虚假的 “unwind”
边。这确保了控制流图有一个最终退出节点（来成功返回和恢复节点），该退出节点控制图中的所有其他节点。

如果我们不添加这样的边，就会导致许多令人惊讶的类型检查程序。例如，只要函数从未返回，就可能借用具有 `'static` 
生命周期的局部变量：

```rust
fn main() {
    let x: usize;
    let y: &'static x = &x;
    loop { }
}
```

这会是可行的，因为（正如借用检查部分详细介绍的那样），永远无法访问 `StorageDead(x)` 
指令，因此可以接受任意长的借用期限。这进一步导致了其他仍然进行类型检查的程序，例如下面的示例使用（不正确但声明为 
unsafe 的） API 来生成线程：

```rust,ignore
let scope = Scope::new();
let mut foo = 22;

unsafe {
    // dtor joins the thread
    let _guard = scope.spawn(&mut foo);
    loop {
        foo += 1;
    }
    // drop of `_guard` joins the thread
}
```

如果没有 unwind 的边，这段代码将通过借用检查，因为无法访问 `_guard` 的 drop 函数 （和 `StorageDead` 指令），因此
`_guard` 被认为是不存活的（毕竟，它的析构函数确实永远不会运行）。然而，这将允许在无限循环期间，由 `scope.spawn()`
开启的线程修改 `foo` 变量，这个线程被赋予访问 `&mut foo` 引用的权限（尽管理论上它的生命周期很短）。

对于假的 unwind 边，编译器其实总是假设析构函数可以运行，因为理论上每个作用域都可以执行。这扩展了给 `scope.spawn()`
线程的 `&mut foo` 的借用，让其覆盖 loop 循环体，从而导致借用借用检查错误。

## 第 3 层：适应 drop 检查

MIR 包括一个相当于“删除” (drop) 变量的操作：

```
DROP(variable)
```

注意，尽管 MIR 一般支持 drop 任何 lvalue ，但在运行此分析时，我们总是一次 drop 整个变量。此操作执行 `variable` 
的析构函数，有效地“反初始化”值所在的内存（如果变量或部分变量已被 drop，则 drop 无效，但这与当前分析无关）。

有趣的是，在许多情况下，drop 一个值并不需要这个 drop 的值中的生命周期是有效的。毕竟， drop `&'a T` 或 `&'a mut T`
被定义为无操作 (no-op) ，因此引用是否指向有效内存并不重要。在这种情况下，我们说生命周期 `'a` 
可能会悬空 (dangle)。这是受到 C
语言的术语“<abbr title="dangling pointer">悬垂指针</abbr>”的启发，它意味着指向已释放或无效内存的指针。

然而，如果同一个引用存储在实现 `Drop` trait 的结构体的字段中，那么该结构体可以在其析构函数期间访问引用的值，因此在这种情况下，该引用的有效性非常重要。

换句话说，如果你有一个类型为 `Foo<'a>` 的值 `v`，它实现了 `Drop`，那么当 `v` 被 drop 
时，`'a` 通常不能悬空（就像任何其他操作都不允许 `'a` 悬空一样）。

更一般地说，RFC 1327 定义了具体的规则，一个类型中的哪些生命周期在 drop 过程中可能会悬空，而在哪些可能不会悬空。

我们将这些规则整合到存活分析中，如下所示：MIR 指令 `DROP(variable)` 在存活方面与其他 MIR 
指令不同。在某种意义上，我们在概念上运行两种不同的存活分析（在原型实践中，每个变量使用两位 (two bits) ）：

1. 我们已经见过第一种存活分析了，它表示变量的当前的值在未来可能被使用的时间。这相当于在 MIR
    中对变量使用 “non-drop”。每当一个变量按照这个定义处于存活状态时，其类型中的所有生命周期都是存活的。
2. 现在讲第二种存活分析，它表示变量表示变量的当前的值在未来可能会被 drop 的时间。这相当于在 MIR
    中对变量使用 “drop”。每当一个变量在这种意义上是存活的，其类型中的所有生命期（标记为 may-dangle 
    的生命期除外）都是存活的。

在 drop 过程中允许生命悬空是非常重要的！事实上，即使是最基本的非词法生命周期示例，如问题案例 
\#1，这也很重要。毕竟，如果我们将问题案例 \#1 翻译成 MIR，我们会看到引用 `slice` 最终会被放在块的末尾：

```rust,ignore
let mut data: Vec<i32>;
let slice: &'slice mut i32;
START {
    ...
    slice = &'borrow mut data;
    capitalize(slice);
    data.push('d');
    data.push('e');
    data.push('f');
    DROP(slice);
    DROP(data);
}
```

然而，这不影响我们的分析，因为 `'slice` 可能会在 drop 过程中“悬空”，而被视为不存活。

## 第 4 层：命名的生命周期

到目前为止，我们只考虑了限制在函数范围内的生命周期。通常，我们想要在当前函数结束开始或终止时推理生命周期。

更微妙的是，我们有时希望生命周期有时在当前函数中开始和结束，但可能（沿着某些路径）延伸到调用方。

考虑问题案例 #3（原型中的相应测试用例是 
[get-default](https://github.com/nikomatsakis/nll/blob/master/test/get-default.nll) ）：

```rust,ignore
fn get_default<'r,K,V:Default>(map: &'r mut HashMap<K,V>,
                               key: K)
                               -> &'r mut V {
    match map.get_mut(&key) { // -------------+ 'r
        Some(value) => value,              // |
        None => {                          // |
            map.insert(key, V::default()); // |
            //  ^~~~~~ ERROR               // |
            map.get_mut(&key).unwrap()     // |
        }                                  // |
    }                                      // |
}                                          // v
```

将上面转化为 MIR 后得到如下结果（这是“伪 MIR”）：

```
block START {
  m1 = &'m1 mut *map;  // temporary created for `map.get_mut()` call
  v = Map::get_mut(m1, &key);
  switch v { SOME NONE };
}

block SOME {
  return = v.as<Some>.0; // assign to return value slot
  goto END;
}

block NONE {
  Map::insert(&*map, key, ...);
  m2 = &'m2 mut *map;  // temporary created for `map.get_mut()` call
  v = Map::get_mut(m2, &key);
  return = ... // "unwrap" of `v`
  goto END;
}

block END {
  return;
}
```

这个例子的关键是，`map` 的第一次借用具有生命周期 `'m1`，它必须延伸到 `'r`
的末尾，但前提是我们分支到 SOME。否则，一旦我们进入 NONE 块，`'m1` 就会结束。

对此，我们将扩展区域的概念，使其不仅包括控制流图中的点，还包括（可能是空的）一组用于各种命名生命周期的“结束区域”。

定义某些命名区域 `'r` 为 `end('r)`。区域 `end('r)`
在语义上可以理解为调用方的控制流图的某个部分。

（实际上，它们可以延伸到调用方的末尾之外，进入调用方的调用方，等等，但这与我们无关）

然后，这个新区域可以表示为以下（伪代码形式）：

```rust,ignore
struct Region {
  points: Set<Point>,
  end_regions: Set<NamedLifetime>,
}
```

在本例中，当一个类型涉及一个命名的生命周期时，例如 `'r`，则这个类型可以由一个区域表示，该区域包括：

* 整个 CFG
* 以及该命名生命周期的结束区域（`end('r)`）

此外，我们可以对集合进行细化，让每个命名的生命周期 `'x` 包含 `'end('x)`，从而满足 `'r: 'x`。这是因为，如果
`'r: 'x`，那么直到 `'x` 已结束， `'r` 才会结束。

最后，我们必须调整对子类型的定义，以适应修改区域这一定义，做法如下，当有如下谁比谁更长的关系时

```
'b: 'a @ P
```

如果可以不离开 `'a` 从 P 到达终点 CFG ，那么现有的推理算法只需将终点添加到 `'b`
然后停止。新算法还将在 `'a` 到 `'b` 之间添加任何包含在 `'a` 的末端区域。

（如果还包括 `'a` 所包括的末端区域，则 `'b` 只比 `'a` 更长，假定可以从 P 到达 CFG 的终点）。

我们要求 CFG 的终点是可访问的，否则数据永远不会脱离当前函数，因此 `end('r)`
是不可访问的（因为 `end('r)` 只覆盖返回后执行的调用方中的代码）。

注：原型的这一部分已经部分实现。[Issue #12](https://github.com/nikomatsakis/nll/issues/12)
描述了当前的状态以及包含正在进行的 PRs 的链接。

## 第 5 层：借用检查的工作原理

在很大程度上， RFC 的重点是生命周期的结构，但值得一提的是如何将这些非词法生命周期集成到借用检查器中。

特别是，在此过程中，我们希望修复借用检查器的两个缺点：

1. **支持像 `vec.push(vec.len())` 这样的嵌套方法调用**。计划继续实施 [RFC 2025: nested_method_call][RFC 2025] 中提出的 `mut2`
    借用解决方案。本 RFC （尚未）提出 RFC 2025 中描述的基于类型的解决方案，如“未来借用”或“ 
    `Ref2`。原因将在备选方案部分讨论。为简单起见，对借用检查器的描述忽略了 RFC 2025。此处所述的扩展与 RFC 2025
    中提出的变更完全无关，这其实导致借用开始延迟。
2. 允许变量在包含可变引用的情况下修改此变量，即使它们的引用对象被借用。这是指问题案例 #4；我们希望原来代码可以通过。

[RFC 2025]: https://github.com/rust-lang/rfcs/pull/2025

### 借用检查第 1 阶段：计算作用域内的 loan

借用检查的第一阶段应计算 CFG 中的每个点的一个作用域内的 loan set。

loan 为元组 `('a, shared | uniq | mut, lvalue)`，其中：

1. `'a` 表示值被借用的生命周期
2. 它是共享的，还是唯一的，还是可变的
    * "unique" loans 其实就像 mutable loans，但 "unique" loans
      不允许改变其引用，它们只用在闭包解语法糖上，而且不属于 Rust 表层语法。
3. lvalue 是被借用的东西（例如 `x` 或者 `(*x).foo`）

通过定点数据流计算，可以找到每个点的作用域内的 loan set。

从 MIR 中的每个借用 rvalue （也就是每个赋值语句，比如 `tmp = &'a b.c.d` ）创建一个 loan
元组，给每个元组一个唯一的索引 `i`。然后，使用 bit-set 表示表示特定点作用域内的 loan set，并执行标准的正向数据流传播。

对于图中 P 点的语句，我们定义“传递函数” (transfer function) ，它表示将哪些 loans 带入或带出作用域：

* kill 不包括区域 P 的任何 loan；
* 一个借用语句会生成对应的 loan；
* 一个赋值语句 `lv = <rvalue>`，kill 它的任何带 `lv` 前缀的路径 P 的 loan。

最后一点值得详细阐述。这条规则允许我们支持问题案例 #4 那样的案例：

```rust,ignore
let list: &mut List<T> = ...;
let v = &mut (*list).value;
list = ...; // <-- 赋值
```

在标记的赋值处，`(*list).value` 的 loan 在作用域内，但以后不必在作用域内考虑它。这是因为变量 `list` 
现在拥有了一个新值，而这个新值还没有被借用（否则我们就无法生成它）。

具体地说，每当我们在 MIR 中看到赋值语句 `lv = <rvalue>`，就可以清除以 `lv` 作为前缀的借用路径 `lv_loan` 
的所有 loans。（在这个示例中，赋值给了 `list`，而且 loan 路径 `(*list).value` 以 `list` 作为前缀。）

注意。在这个阶段，当赋值出现，我们总是清除应用于覆盖路径的所有 loans；然而，在某些情况下，正是由于这些
loans，赋值本身可能是非法的。这个例子中，如果 `list` 的类型是 `List<T>` 而不是 
`&mut List<T>`，则会出现这种情况。此时，错误将由借用检查的下一环节报告，下一节会谈到。

### 借用检查第 2 阶段：报告错误

这个阶段中，应计算了每一点的 loans 在作用域内。然后，遍历 
MIR，并确定在给定作用域内的 loans 的非法行为。我们可以将事情分解为两种可以执行的操作，而不是遍历每一种 MIR 语句：

* 沿着两个方面（浅和深、读和写）来访问一个 lvalue
* drop 一个 lvalue

假设作用域内的一个 loan set 出现在操作的开始，无论哪种操作都指定如下规则，以确定它们何时是合法的。

* `StorageDead` 语句被视为浅写入  (shallow write)
* 赋值语句 `LV = RV` 是对 `LV` 的浅写入
* 而且在 rvalue `RV` 内：
    * 每个 lvalue operand 要么是深读入 (deep read) 要么是深写入 (deep write) 操作，这取决于 lvalue 是否实现了 `Copy`
        * move 被视为深写入
    * 共享借用 `&LV` 被视为深读入
    * 可变借用 `&mut LV` 被视为深写入

因此，借用检查的第二阶段包括迭代 MIR
中的每个语句，并检查给定作用域内的 loans 所执行的操作是否合法。将 MIR 语句转化为操作基本上是直截了当的：

有几个有趣的案例需要记住：

* MIR 模型判别更精确。在借用方面，它应该被视为一个单独的领域。
* 在今天的编译器中， `Box` 仍然是 MIR “内置”的。这个 RFC 忽略了这种可能性，而是认为借用的引用（ `&` 和
  `&mut` ）和原始指针（ `*const` 和 `*mut` ）是唯一的指针类型。虽然在处理 drop
  的过程中出现了一些问题（有关详细信息，请参阅 drop
  一节），但应该直接扩展这里所说的到 `Box` 的情况。

访问一个 lvalue `LV` 时，需要考虑两个方面：

* 访问可以是浅访问也可以是深访问：
    * 浅访问意味着访问在 `LV` 处到达的直接字段，但在 `LV` 中找到的引用或指针不会被解引用。现在，唯一的浅访问是像
      `x = ...` 这样对 `x` 浅写入。
    * 深访问意味着通过给定 lvalue 的可访问的所有数据都可能会通过此操作失效或访问成功。
* 访问可以是读或写：
    * 读意味着可以读取现有数据，但不会修改数据。
    * 写入意味着数据可能会被修改为新值或以其他方式失效（例如，它可能会在移动操作中被反初始化）。

`Deep` 访问通常很深，因为它们创建并释放了一个别名，此时， “Deep”
一词反映了通过该别名可能发生的情况。例如，`let x = &mut y` 被认为是 `y`
的一个深写入，即使实际的借用根本不做任何事情，我们也会创建一个可变别名 `x`，它可以用来改变从 `y`
访问的任何东西。移动语句 `let x = y` 也是类似：它写入 `y`
的浅层内容，但，我们可以通过新名称 `x`访问通过 `y` 访问的所有其他内容。

决定访问何时合法的伪代码就像下面这样：

```rust,ignore
fn access_legal(lvalue, is_shallow, is_read) {
    let relevant_borrows = select_relevant_borrows(lvalue, is_shallow);

    for borrow in relevant_borrows {
        // shared borrows like `&x` still permit reads from `x` (but not writes)
        if is_read && borrow.is_read { continue; }
        
        // otherwise, report an error, because we have an access
        // that conflicts with an in-scope borrow
        report_error();
    }
}
```

它分两步工作。

首先，枚举一组与 `lvalue` 相关的作用域内的借用，它们受到“深”“浅”操作的影响，稍后会描述。

然后，对于每一个这样的借用，检查它是否与操作冲突（即如果其中至少有一个可能正在写入），如果是，则报告错误。

如果相关的借用满足下列条件之一，则考虑这些借用**浅**访问路径 `lvalue`：

* 路径 `lvalue` 存在一个 loan
    * 因此：如果借用了 `a.b.c`，那么写入像 `a.b.c` 这样的路径是非法的
* 路径 `lvalue` 的前缀存在一个 loan
    * 因此：如果借用了 `a` 或 `a.b`，那么写入像 `a.b.c` 这样的路径是非法的
* `lvalue` 是 loan 路径的浅前缀 (shallow prefix)
    * 通过去掉字段可以找到浅前缀，但在任何解引用时要停止寻找
    * 因此：如果借用了 `a.b`，那么写入像 `a` 这样的路径是非法的
    * 但是：如果借用了 `*a`，那么写入 `a` 是合法的，无论 `a` 是否是共享引用还是可变引用

如果相关的借用满足下列条件之一，则考虑这些借用**深**访问路径 `lvalue`：

* 路径 `lvalue` 存在一个 loan
    * 因此：如果可变地借用了 `a.b.c`，那么读取像 `a.b.c` 这样的路径是非法的
* 路径 `lvalue` 的前缀存在一个 loan
    * 因此：如果可变地借用了 `a` 或 `a.b`，那么读取像 `a.b.c` 这样的路径是非法的
* `lvalue` 是 loan 路径的支撑前缀 (supporting prefix)
    * 稍后会定义支撑前缀
    * 因此：如果可变地借用了 `a.b`，那么读取像 `a` 这样的路径是非法的
    * 但是：与浅访问不同，如果可变地借用了 `*a`，那么读取 `a` 也是非合法的

可以认为 **drop lvalue** 是深写入 (deep write)，就像一次移动，但这过于保守。这方面的规则还在积极制定中，见
[\#40](https://github.com/nikomatsakis/nll-rfc/issues/40)。

# 如何教授

## 术语

在本 RFC 中，我选择继续使用“生命周期” (lifetime) 这个术语。它指程序中有效使用引用的那部分（或者，也可以指“借用的期限”）。

正如在开头所明确指出的那样，这个术语与另一种用法有些冲突。另一种用法下的生命周期指的是一个值的动态范围（我们称之为“作用域”
(scope) ）。

我认为，如果要重新开始，最好能找到一个更明确的另一个术语。然而，此时很难尝试更改“生命周期”术语，因此本
RFC 不尝试这样做。但为了避免混淆，错误消息最好指明其区域，并尽可能避免使用“生命周期”一词，或者使用限定词使其含义更清楚。

## 利用直觉：根据地点来指明错误

目前 Rust 使用词法作用域来确定生命周期的部分原因是，以前认为使用者对词法作用域进行推理会更简单。

时间和经验并没有证明这一说法：对许多使用者来说，“人为地”延长借用到块 (block)
结束会有些惊讶。此外，大多数使用者对控制流有相当直观的理解（这的确如此，因为为了理解程序要做什么，你必须理解控制流）。

因此，我们建议在解释借用错误和生命周期错误时利用这种直觉。尽可能地从以下三个方面解释所有错误：

* 借用发生的地点 (B)
* 使用引用的地点 (U)
* 可能使引用无效的中间点 (A)

我们应该这样选择地点的顺序：B 可以到达 A， A 可以到达 U。

一般来说，采用“叙述”的形式描述错误：

* 首先，产生了值的借用。
* 然后，产生了使引用无效的操作。
* 最后，在引用无效后，下一次该引用被使用了。

这种方法与我们今天所做的类似，但我们经常忽略第三点，即下一次使用的地方。

注意，“错误发生的地点”仍然因为第二步的操作。也就是说，从概念上讲，错误是由于两次使用引用之间，执行了无效操作导致的
（而不是在失效操作之后使用引用导致的）。这实际上更准确地反映了未定义行为的定义
（即，执行非法写入是导致未定义行为的原因，但由于后面的使用，写入是非法的）。

要看到有没有第三点带来的不同结果，考虑这个错误的程序：

```rust
fn main() {
    let mut i = 3;
    let x = &i;
    i += 1;
    println!("{}", x);
}
```

目前，Rust 报告以下错误：

```rust,ignore
error[E0506]: cannot assign to `i` because it is borrowed
 --> <anon>:4:5
   |
 3 |     let x = &i;
   |              - borrow of `i` occurs here
 4 |     i += 1;
   |     ^^^^^^ assignment to borrowed `i` occurs here
```

这里的点 B 和点 A 点突出显示，但使用点 U 没有显示出来。此外，赋值一行负主要责任。在本 RFC 下，应显示以下错误：

```rust,ignore
error[E0506]: cannot write to `i` while borrowed
 --> <anon>:4:5
   |
 3 |     let x = &i;
   |              - (shared) borrow of `i` occurs here
 4 |     i += 1;
   |     ^^^^^^ write to `i` occurs here, while borrow is still active
 5 |     println!("{}", x);
   |                    - borrow is later used here
```

另一个例子，这次使用 `match` 语句：

```rust
fn main() {
    let mut x = Some(3);
    match &mut x {
        Some(i) => {
            x = None;
            *i += 1;
        }
        None => {
            x = Some(0); // OK
        }
    }
}
```

错误应是：

```rust,ignore
error[E0506]: cannot write to `x` while borrowed
 --> <anon>:4:5
   |
 3 |     match &mut x {
   |           ------ (mutable) borrow of `x` occurs here
 4 |         Some(i) => {
 5 |              x = None;
   |              ^^^^^^^^ write to `x` occurs here, while borrow is still active
 6 |              *i += 1;
   |              -- borrow is later used here
   |
```

（注意，`None` 分支中的赋值不是错误，因为不再使用借用。）

## 一些特殊情况

在某些情况下，这三处地点在使用者的语法中并不都可以看到，因此可能需要仔细处理。

### 最后一次使用是 drop

有时，最后一次使用变量实际上发生在它的析构函数。比如：

```rust,editable
#![allow(unused)]

struct Foo<'a> { field: &'a u32 }

// 当你删除 Drop 实现，代码成功运行
impl<'a> Drop for Foo<'a> { 
    fn drop(&mut self) { }
}

fn main() {
    let mut x = 22;
    let y = Foo { field: &x };
    x += 1;
}
```

这段代码是合法的，但由于 `y` 的析构函数会在封闭作用域的的末尾隐式执行，错误消息可能显示如下：

```rust,ignore
error[E0506]: cannot assign to `x` because it is borrowed
  --> src/main.rs:13:5
   |
12 |     let y = Foo { field: &x };
   |                          -- borrow of `x` occurs here
13 |     x += 1;
   |     ^^^^^^ assignment to borrowed `x` occurs here
14 | }
   | - borrow might be used here, when `y` is dropped and runs the `Drop` code for type `Foo`
```

### 调用方法

一个调用方法的例子：

```rust,editable
fn main() {
    let mut x = vec![1];
    x.push(x.pop().unwrap());
    // Vec::push(&mut x, x.pop().unwrap());
}
```

对于这种情况，应报告下面的错误：

```rust,ignore
error[E0499]: cannot borrow `x` as mutable more than once at a time
 --> src/main.rs:3:12
  |
3 |     x.push(x.pop().unwrap());
  |     -------^^^^^^^----------
  |     | |    |
  |     | |    second mutable borrow occurs here
  |     | first borrow later used by call
  |     first mutable borrow occurs here
```

使用注释那行代码，则应报告的错误[^push-pop-error]：

```rust,ignore
error[E0499]: cannot borrow `x` as mutable more than once at a time
 --> src/main.rs:4:23
  |
4 |     Vec::push(&mut x, x.pop().unwrap());
  |     --------- ------  ^^^^^^^ second mutable borrow occurs here
  |     |         |
  |     |         first mutable borrow occurs here
  |     first borrow later used by call
```

[^push-pop-error]: 【译者注】我更新了这一部分，RFC 2094
原文针对这两种方式提出了两种错误，现在 Rust 更智能，把两种方式的错误统一了。

### 闭包

今天，当最初的借用是构造闭包的一部分时，我们希望不仅强调构造闭包的地点，还要强调**闭包中**使用相关变量的地点。

## 借用的变量超过其范围

考虑这个例子：

```rust
let p;
{
    let x = 3;
    p = &x;
}
println!("{}", p);
```

引用 `p` 指向 `x` 的生命周期超过了 `x` 的作用域。简而言之，栈的这一部分将 pop，而 `p`
仍在使用中。今天的编译器的借用检查器可通过一个特殊检查检测到这种情况，该检查计算被借用的路径的“最大作用域”（此处是 `x`）。

这对现有的系统是有意义的，因为生命周期和作用域是用相同的单位表示的（都是 AST 
的一部分）。在较新的非词法表述中，这种错误的检测方式会有所不同。

如前所述，我们可以看到，`StorageDead` 指令在 `p` 仍在使用时释放 `x` 的插槽 (slot) 
。因此，可以用同样的“三点式”来表达错误：

```rust,ignore
error[E0597]: `x` does not live long enough
 --> src/main.rs:7:9
  |
7 |     p = &x;
  |         ^^ borrowed value does not live long enough
8 | }
  | - `x` dropped here while still borrowed
9 | println!("{}", p);
  |                - borrow later used here
```

## 推理过程中的错误

剩下的一组与生命周期相关的错误主要发生在与函数签名的交互。例如：

```rust
impl Foo {
    fn foo(&self, y: &u8) -> &u8 {
        x
    }
}
```

为了更好地呈现这些类型的错误，我们已经开展了一些工作。
[issue 42516](https://github.com/rust-lang/rust/issues/42516)
中有大量示例和细节，那里的所有内容都应该适用于这里的内容。

简而言之，最主要的问题是识别模式并对改进函数签名提出修改建议，使其与函数体匹配（或至少更清楚地诊断出问题）。

尽可能地应该利用控制流中的点，并尝试以“叙述”的形式解释错误。

# 缺点

这项提议几乎没有缺点。主要的一点是，系统的**规则**变得更加复杂。

然而，这允许接受更多的程序，因此我们希望**使用 Rust** 时会感觉更简单。

此外，经验表明，对许多使用者来说，当前将引用的生命周期与词法作用域联系起来的方案令人困惑和惊讶。

# 备择方案

### NLL 的替代方法

在本 RFC 的筹备阶段，尝试并放弃了许多描述 NLL 的替代方案和方法。

1. [RFC 396](https://github.com/rust-lang/rfcs/pull/396) 将生命周期定义为支配树 (dominator tree) 
   的“前缀”，粗略地说，那是控制流图的单入口、多出口区域。与我们提出的系统不同，这个定义不允许在生命周期中出现洞 
   (holes)。确保连续的生命周期目的是保证安全；在这个 RFC
   中，我们使用生命周期约束来实现类似的效果。这种更灵活的设置允许我们处理问题案例 #3 这样的案例，
   RFC 396 不允许这种案例，而且也没有涉及 drop 检查和其他一些复杂的事情。
2. **SSA 或 SSI 变换**。
   我们没有将“当前位置”纳入子类型检查，而是考虑了先将 SSA 
   变换应用于输入程序，然后为每个变量指定不同类型。这确实允许某些示例进行一些否则无法进行的类型检查，但对于前面介绍的
   `vec-push-ref` 示例来说，它不够灵活。使用 SSA 也会带来其他复杂的问题。此外， Rust 
   允许间接地借用来改变（例如通过 `&mut` ）变量和临时变量。如果我们以一种天真的方式将 SSA
   应用于 MIR，那么在创建数字时，它将忽略这些赋值。例如：

    ```rust,ignore
    let mut x = 1;      // x0, has value 1
    let mut p = &mut x; // p0
    *p += 1;
    use(x);             // uses `x0`, but it now has value 2
    ```

    这里的 `x0` 的值由于从 `p` 写入而发生改变。因此，这不再是真正的 SSA 形式。通常， SSA
    变换通过把局部变量（如 `x` 和 `p` ）作为指向栈插槽 (stack slot)
    的指针来改变值，然后在安全的时候把这些栈插槽提升为局部变量。 MIR
    故意不使用 SSA 形式来避免这种别扭的方式（我们可以将其留给优化后端）。

3. **每个程序点使用一种类型**。按程序点键入。除了 SSA，我们还可以通过如下方案来应对 `vec-push-ref`：在 CFG
   中的每个点为每个变量提供一个不同的类型（类似于 Ericson2314 在
   [stateful MIR for Rust](https://github.com/Ericson2314/a-stateful-mir-for-rust)
   描述的那样），并将变换应用于每条边上的生命周期。在 rustc 的设计最后阶段，编译器团队也列举了这样的设计。作者认为，这种本
   RFC 是一种相当粗略的分析，但有一种更为常见的替代构想，即每个变量使用一种类型（而不是每个点的每个变量使用一种类型）。

    每个变量使用一种类型的这种设计有几个优点：
    * 它涉及的推理变量要少得多：如果每个变量有许多类型，那么每个类型在每个点上都需要不同的推理变量
    * 它涉及的约束要少得多：仅仅用于连接不同点之间的变量类型时无需约束
    * 它也更自然地适用于表层语言，因为表层语言中的变量只有一种类型

### Different "lifetime roles"

### 不同的“生命周期角色”

在关于嵌套方法调用的讨论中（见 [RFC 2025](https://github.com/rust-lang/rfcs/pull/2025)
，以及由此引发的讨论），有各种各样的建议，旨在接受像 `vec.push(vec.len())` 这样的调用的解语法糖：

```rust,ignore
let tmp0 = &mut vec;
let tmp1 = vec.len(); // does a shared borrow of vec
Vec::push(tmp0, tmp1);
```

[RFC 2025](https://rust-lang.github.io/rfcs/2025-nested-method-calls.html)
的替代方案侧重于增加引用的类型，使其具有不同的“角色”。最突出的建议是 
`Ref2<'r, 'w>`，它让可变引用变为具有两个不同的生命周期，“读取”的生命周期（ `r` ）和“写入”生命周期（ `w`
），其中读取包含引用的整个范围，但是写入只包含正在发生写操作的地点。此 RFC
不尝试更改为嵌套方法调用，而是继续使用 RFC 2025 的方法（这只影响借用检查）。然而，如果我们真的希望在将来采用 `Ref2`
风格的方法，可以向后兼容地进行，但需要修改（例如）存活要求。比如，当前如果变量 `x` 在某个点 P 处有效，那么 `x`
类型中的所有生命周期都必须包含 P ，但在 `Ref2` 
方法中，只有读取生命周期必须包含P。这意味着要根据它们的“角色”对生命周期进行不同的处理。将这样的变化隔离成一个单独的
RFC 似乎是个好主意。

# 悬而未决的问题

目前没有。

# 附录：本提案不会解决的问题

值得讨论一下当前 RFC 不会消除的几种借用检查错误。这些错误通常以某种形式跨越程序界限。

1. **闭包解语法糖**[^disjoint-capture]。

    以前，闭包总是捕获局部变量，即使闭包只在内部使用变量的某些子路径：

    ```rust
    ##![allow(dead_code)]
    #fn main() {}
    #struct A {
    #    vec: Vec<u8>,
    #    vec2: Vec<u8>,
    #}
    impl A {
        fn f(&mut self) {
            let _ = || self.vec.len(); // borrows `self`, not `self.vec`
            self.vec2.push(0); // 在 Rust 1.56 之前会报 error: self is borrowed
        }
    }
    ```

[^disjoint-capture]: 【译者注】这个问题在
[Rust-1.56: Disjoint capture in closures](https://blog.rust-lang.org/2021/10/21/Rust-1.56.0.html#disjoint-capture-in-closures) 中已经解决了。

2. **不相交的跨函数字段**。

    另一种错误发生在，一个方法只使用字段 `a`，而另一个方法只使用 
    `b`；现在，你无法做到这一点，因此这两种方法不能“并行”使用：

    ```rust
    ##![allow(dead_code)]
    #fn main() {}
    #fn use_a(_: &u8) {}
    #struct B {
    #    value: u8
    #}
    #struct Foo {
    #    a: u8,
    #    b: B,
    #}
    impl Foo {
        fn get_a(&self) -> &u8 {
            &self.a
        }
        fn inc_b(&mut self) {
            self.b.value += 1;
        }
        fn bar(&mut self) {
            let a = self.get_a();
            self.inc_b(); // Error: self is already borrowed
            use_a(a);
        }
    }
    ```

    解决这个问题的方法是重构代码，以明确表示这些方法是操作不相交的数据。例如，可以将方法分解为字段本身的方法：

    ```rust
    ##![allow(dead_code)]
    #fn main() {}
    #fn use_a(_: &u8) {}
    #struct B {
    #    value: u8
    #}
    #impl B {
    #    fn inc(&mut self) {
    #        self.value += 1;
    #    }
    #}
    #struct Foo {
    #    a: u8,
    #    b: B,
    #}
    impl Foo {
        fn bar(&mut self) {
            // 利用闭包的不相交捕获
            let get_a = || &self.a;
            let a = get_a();
            // 或者当 a 像 b 一样是带字段的结构体，则重构成
            // let a = self.a.get();
            self.b.inc();
            use_a(a);
        }
    }
    ```

    这样，当我们单独看 `bar()` 函数时，有 `self.a` 和 `self.b` 分别的两个借用，而不是两个对 `self` 的借用。

    另一种技巧是引入“自由函数” (free functions) （例如 `get(&self.a)` 和 `inc(&mut self.b)`
    ），它们可以更清楚地公开哪些字段被操作，或者内联方法体。这是一个非常重要的设计，超出了本 RFC
    的范围。请参阅内部线程上的[这条][comment]评论，以了解进一步的想法。

[comment]: https://internals.rust-lang.org/t/partially-borrowed-moved-struct-types/5392/2

3. **<abbr title="Self-referential">自引用</abbr>结构体**。

    我们还没有解决的最后一个限制是无法拥有“自引用结构体”。

    也就是说，你不能让一个结构本身存储一个 arena 和指向该 arena 的指针，然后移动该结构。

    这会出现在许多设置中。有各种解决方法：例如，有时你可以使用带有索引的向量，或者 [`owning_ref`] 
    crate。后面这种与[关联类型][GAT]结合使用时，实际上可能是某些用例的适当解决方案（它实际上是在 lib 
    代码中模拟“存在生命周期”的方法）。

    尤其对于 futures 来说，[`?Move` RFC][Move] 提出了另一种轻量级且有趣的方法。

[`owning_ref`]: https://crates.io/crates/owning_ref
[GAT]: https://github.com/rust-lang/rfcs/pull/1598
[Move]: https://github.com/rust-lang/rfcs/pull/1858
