
Pinning in plain English
========================

![](data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMTYwMCIgaGVpZ2h0PSI4NDAiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgdmVyc2lvbj0iMS4xIi8+)
![](https://blog.schichler.dev/_next/image?url=https%3A%2F%2Fcdn.hashnode.com%2Fres%2Fhashnode%2Fimage%2Fupload%2Fv1637755591169%2FOMsBWv8Bc.png%3Fw%3D1600%26h%3D840%26fit%3Dcrop%26crop%3Dentropy%26auto%3Dcompress%2Cformat%26format%3Dwebp&w=3840&q=75)

[![](data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMjAwIiBoZWlnaHQ9IjIwMCIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB2ZXJzaW9uPSIxLjEiLz4=)
![](https://blog.schichler.dev/_next/image?url=https%3A%2F%2Fcdn.hashnode.com%2Fres%2Fhashnode%2Fimage%2Fupload%2Fv1600792675173%2FrY-APy9Fc.png%3Fauto%3Dcompress&w=640&q=75)
](https://hashnode.com/@Tamme)

[Tamme Schichler](https://hashnode.com/@Tamme)

[Published on Nov 25, 2021](https://blog.schichler.dev/pinning-in-plain-english-ckwdq3pd0065zwks10raohh85)

15 min read

> The header image shows an orange bell pepper sitting on a wooden cutting board.
> 
> The same bell pepper has been mirrored into the right half of the image. It is still clearly the same fruit, but the mirror image would largely only be compatible with a fundamentally different biology, as many of its more complex molecules are chiral.
> 
> It was fresh and refreshing.
> 
> * * *

Pinning in Rust is a powerful and very convenient pattern that is, in my eyes, not supported well enough in the wider ecosystem.

A common sentiment is that it's hard to understand and that [the pin module documentation](https://doc.rust-lang.org/stable/core/pin/index.html) is confusing. (Personally, I think it works well as reference to answer questions about edge cases, but it's a dense read and not necessarily a good intro text.)

This post is my attempt to make the feature more approachable, to hopefully inspire more developers to make their crates pinning-aware where that would be helpful.

* * *

**License and Translations**

See past the end of this post. In short, this post as a whole is licensed under [CC BY-NC-SA 2.0](https://creativecommons.org/licenses/by-nc-sa/2.0/) except for code citations explicitly marked as such. Code snippets and blocks that are _not_ marked as citations are also licensed under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/).

* * *

Alright, on to the actual content!

Note that I've added links to (mostly) the Rust documentation in various places.  
These are "further reading", so I recommend reading this post entirely before looking into them. They'll (hopefully) be much easier to understand that way around.

The Problem
===========

**In Rust, any instance is considered trivially movable** as long as its size is known at compile-time, which means that anyone who owns or has an `&mut` (a plain exclusive reference) to an instance can copy its unstructured data (i.e. its directly contained bytes) to a different memory address, and can then expect nothing to break when they reuse the old location otherwise or use the moved instance.

Unlike in C# or JavaScript, this matters:  
References, pointers and addresses can be fairly readily converted between each other, and **addresses (and to some extent pointers) are plain numbers that can be used in offset calculations**. This makes it possible to, for example, create very efficient generic collections, as they can always store instances "by-value" in the same allocation. If they've run out of space to add a new item, they implicitly reallocate their storage, possibly moving their contents to a new location in memory.

However, unlike in C++ ([1](https://en.cppreference.com/w/cpp/language/copy_constructor), [2](https://en.cppreference.com/w/cpp/language/move_constructor), [3](https://en.cppreference.com/w/cpp/language/copy_assignment), [4](https://en.cppreference.com/w/cpp/language/move_assignment)), **there is no built-in mechanism to update pointers and addresses not known directly** to the owner of an instance, **or to prevent moves completely for a specific type**: You can't overload or delete the plain assignment operator `=`, and there is no concept of a [move](https://en.cppreference.com/w/cpp/language/move_constructor) or [copy constructor](https://en.cppreference.com/w/cpp/language/copy_constructor) that could be called when an instance is moved implicitly.

Rust-style references (`&` or `&mut`) are lightweight pointers with some aliasing restrictions that are always valid¹ and can't be automatically updated from elsewhere, which in turn **rules out reallocation of their target _entirely_ during these borrows** (but code with access to the exclusive `&mut T` can still [`swap`](https://doc.rust-lang.org/stable/core/mem/fn.swap.html) instances freely).

(¹ more specifically: References are always guaranteed to be dereferenceable, which allows the compiler to load and cache their pointed-to values early in many cases. C++ makes the same dereferenceability guarantee, but allows mutable aliasing by default.)

All this together makes it tricky to write a memory-safe API that relies on the known location of an instance between calls into it, as **taking continuous possession of it through ownership or a borrow would be inflexible** and often inconvenient, and **using an indirect handle would be too inefficient** for many low-level components.

Pinning TL;DR (simplified)
==========================

Instead, **Rust opts to explicitly change the visible type of a reference (and with that the API) to prevent accidental moves outside of `unsafe` code**.

While there are some exceptions to this, the _default_ assertion of pinning is that "If a `Pin<&T>` is accessible, then that instance of `T` will be available at that address until dropped."

It's possible to weaken this somewhat but, aside from when `T: Unpin`, only by using `unsafe` in some way.

Whenever a type implements [`Unpin`](https://doc.rust-lang.org/stable/core/marker/trait.Unpin.html), pinning its instances [is meaningless](https://doc.rust-lang.org/stable/core/pin/struct.Pin.html#impl-DerefMut) however.

**For simplicity, types of pinned values (in this post: `T`) are implied to be `!Unpin` for the rest of this post, unless otherwise noted.** This means some sentences won't be true if you, for example, try to pin an `f64` or a `struct` where all of its members are `Unpin`. Please keep this in mind while reading on.

"pin" vs. "pinned"
==================

Whenever pinning happens in Rust, there are two components involved:

*   A "pin" type that asserts "the value can't be moved (by _safe_ Rust)".
    
    This is often called a "pinning pointer".
    
    Pins are very often `Unpin`, but this isn't relevant to their function.  
    Instead, this rule is enforced only through API constraints of the pin.
    
*   A _pinned value_, which can't be moved by the consumer of an API.
    
    This is always a specific value, not all members of a type in general, as "pinned" is _not_ an inherent property of any type.
    

Pins are often compounds
========================

For example, look at the signature of [`Box::pin`](https://doc.rust-lang.org/stable/alloc/boxed/struct.Box.html#method.pin):

Copy

```
// Citation.
pub fn pin(x: T) -> Pin<Box<T>> { … }

```

This function pins the value `x` of type `T` behind a pinning smart pointer of type `Pin<Box<_>>`.

Think of `Pin<Box<_>>` as "pinning `Box`", and not as "`Box` inside a `Pin`". `Pin` is not a container that the `Box` can be _safely_ taken out of ([unless `T: Unpin`](https://doc.rust-lang.org/stable/core/pin/struct.Pin.html#method.into_inner)).

Side note: A plain `Pin<Box<_>>` is pinn**ing** but not pinn**ed**.  
In fact, `Pin<Box<_>>` is always `Unpin` (because `Box<_>` is always `Unpin`) and as such can not be meaningfully pinned by itself.

This, including `Unpin` on the pin, is the same for all standard smart pointers and Rust references:

| Rust | English |
| --- | --- |
| `Pin<Box<_>>` | "pinning `Box`" |
| `Pin<Rc<_>>` | "pinning `Rc`" |
| `Pin<Arc<_>>` | "pinning `Arc`" |
| `Pin<&_>` | "pinning (shared) reference" |
| `Pin<&mut _>` | "pinning (exclusive) reference" |

I often shorten "pinning `Box`" to "`Pin`\-`Box`" for myself when reading silently, and you should be understood when saying it out loud like that too.

`Unpin` is an auto trait
========================

Very few types in Rust are actually `!Unpin`. As [`Unpin`](https://doc.rust-lang.org/stable/core/marker/trait.Unpin.html) is an [`auto trait`](https://doc.rust-lang.org/beta/unstable-book/language-features/auto-traits.html), it is implemented for all composed types (structs, enums and unions) whose members are `Unpin` already. It's auto-implemented for nearly all built-in types and **implemented explicitly also for [pointers](https://doc.rust-lang.org/stable/std/primitive.pointer.html)**! This means that pointer wrappers like [`NonNull`](https://doc.rust-lang.org/stable/core/ptr/struct.NonNull.html) are _also_ `Unpin`!

In fact, the only type that is explicitly `!Unpin` as of stable Rust 1.56, including in custom crates, is [`core::marker::PhantomPinned`](https://doc.rust-lang.org/stable/core/marker/struct.PhantomPinned.html), a marker you can use as member type to make your custom type `!Unpin`.

You can see the full list of (non-)implementors here: [`Unpin`#implementors](https://doc.rust-lang.org/stable/core/marker/trait.Unpin.html#implementors)

Values (mostly) don't start out pinned
======================================

Even for `T` where `T` is not `Unpin`, a plain instance of `T` on the stack or accessible through a plain `&mut T` is not yet pinned. This also means it could be discarded without running its destructor, by calling [`mem::forget`](https://doc.rust-lang.org/stable/core/mem/fn.forget.html) with it as parameter for example.

An instance of `T` only becomes pinned when passed to a function like `Box::pin` that makes [these guarantees](https://doc.rust-lang.org/stable/core/pin/index.html#drop-guarantee) (and ideally exposes `Pin<&T>` somehow, as necessary).

Function of `Pin<_>`
====================

The only differences between `Box<T>` and `Pin<Box<T>>` are that:

*   `Pin<Box<T>>` never exposes `&mut T` or a plain `T`,
    
    so moving the value elsewhere is impossible.
    
*   `Pin<Box<T>>` exposes `Pin<&T>` and `Pin<&mut T>`.
    
    This is called "pin projection" (towards the stored value).
    
    Getting access to these pinning references lets you call methods on the value that require `self: Pin<&Self>` or `self: Pin<&mut Self>`, and also associated functions with similar argument types.
    

The plain `&T` value reference is accessible like before, and can also be found as such by dereferencing `Pin<&T>`, regardless of wither `T` is `Unpin`.

**All** smart pointers and references that are [`Deref<Target = T>`](https://doc.rust-lang.org/stable/core/ops/trait.Deref.html) (and optionally, for mutable access, [`DerefMut`](https://doc.rust-lang.org/stable/core/ops/trait.DerefMut.html)) function _exactly_ like this when pinning.

In order to keep the rest of the post easy to read:

**Definitions** (valid in this post only):

| Shorthand | implied constraint | English |
| --- | --- | --- |
| `T` | **not** `T: Unpin` | "\[arbitrary\] type" / "\[arbitrary\] value" |
| a pinned `T` | **not** `T: Unpin` | "a pinned value" |
| `P` | `P: Deref<Target = T>`,  optionally `P: DerefMut` | "pointer" |
| `Pin<P>` | `P: Deref<Target = T>`,  optionally `P: DerefMut` | "pinning pointer" |

Pinning is a compile-time-only concept
======================================

`Pin<P>` is a [`#[repr(transparent)]`](https://doc.rust-lang.org/stable/reference/type-layout.html#the-transparent-representation) wrapper around its single member, a private field with type `P`.

In other words:

**`Pin<Box<T>>` (for example) has the exact same runtime representation as `Box<T>`.**

Since `Box<T>` in turn has the same runtime representation as its derived `&T`, dereferencing `Pin<Box<T>>` to arrive at `&T` is an identity function that returns the exact same value it started out from, which means (but only with optimisations!) no runtime operation is necessary at all.

Converting a pointer or container into its pinning version is an equally free action.

This is possible because moves in Rust are already pretty explicit once references are involved: The underlying assignment maybe be hidden inside another method, but there is no system that will move heap instances around without being told to do so (unlike for example in C#, where pinning is a runtime operation [integrated into the GC API](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.gchandletype?view=net-6.0).)

The only exception to this, where a `&`\-referenced instance's copy can appear with a new address implicitly, are types that are [`Copy`](https://doc.rust-lang.org/stable/core/marker/trait.Copy.html). This trait isn't [`auto`](https://doc.rust-lang.org/unstable-book/language-features/auto-traits.html), which means it must be derived explicitly for each type for which implicit trivial copies should be available.

(Side-note: Don't implement `Copy` on types where identity matters at all. Deriving [`Clone`](https://doc.rust-lang.org/stable/core/clone/trait.Clone.html) is usually enough.  
`Copy` is largely convenience for immutable instances that you want to pass by value a lot, so for example [`Cell`](https://doc.rust-lang.org/stable/core/cell/struct.Cell.html) does not implement it even if the underlying type does.)

Pinning is a matter of perspective
==================================

A value becomes pinned by making it impossible for safe Rust to move the instance or free its memory without dropping it first. (A pin giving safe Rust access to `Pin<&T>` or `Pin<&mut T>` asserts this formally, especially towards `T`'s unrelated implementation.)

However, as the type of the pinned instance itself does not change, it can remain visible "unpinned" inside the module that implements a pin in the first place.

_`Pin<_>` hides the normal mutable API only through encapsulation, but can't erase it entirely._

This is especially true for pins with [interior mutability](https://doc.rust-lang.org/core/cell/index.html) or composed of non-pinning fields, since they will normally share much of their implementation between the non-pinning and pinning API.

Safe code inside such modules will (currently) often handle `&mut T` while a derived `Pin<&T>` could have been presented outwards before, and extra care must be taken to avoid unsound moves in that case.  
This may change in the future as more pinning pointers and collections become available and if Rust makes it easier to add methods to custom types wrapped in `Pin<_>`.

Collections can pin
===================

This is most obvious with `Box<T>` or `Pin<Box<T>>` where the `Box` acts as 1-item collection of `T`. The same can be said about these types with "`Arc`" and "`Rc`" instead of "`Box`".

As such, **it makes sense to also pay attention to `C` where `C` is a collection of items of type `T`**.

It's likewise possible to use `Pin` to create a new type `Pin<C>` that behaves similarly to how a pinning smart pointer would, by giving out neither `&mut T` or `T`.

**Definitions** (valid in this post only):

| Shorthand | implied constraint | English |
| --- | --- | --- |
| `C` | owns instances of type `T` | "collection" |
| `Pin<C>` | owns instances of type `T` | "pinning collection" |

(The standard library doesn't have many utilities for this, as general collections are much more diverse than smart pointers. If you decide to write a pinning-aware collection, you will have to implement much of the API yourself and, as of Rust 1.56, may have to provide extension methods through traits to make calling it seamless.)

A collection `C` "can pin" if it allows some projection from its pinned form (`&Pin<C>` or `&mut Pin<C>`) to `Pin<&T>` and optionally to `Pin<&mut T>`.

A collection may also be _inherently_ pinning, in which case it will act like `Pin<C>` without `Pin` appearing in the type. We won't look at this kind of collection directly here.

`Pin<P>` vs. `Pin<C>` vs. `T`
=============================

How plain (non-pinning) pointers and collections behave should be clear enough, so I'll only compare how their and `T`'s API _tend to_ differ when they are pinning or pinned:

|  | `Pin<P>` | `Pin<C>` | `T` behind `Pin<&T>` or `Pin<&mut T>` |
| --- | --- | --- | --- |
| English | "pinning pointer" | "pinning collection" | "pinned value" |
| `: Unpin` | nearly always | very often | rarely in practice, as pinning with `T: Unpin` is meaningless. |
| APIs accessible  vs. without pinning | Access to `Pin<&T>`  and optionally `Pin<&mut T>` | varies,  
but often similar to `Pin<P>` | Functions that require `Pin<&T>` or `Pin<&mut T>` |
| APIs inaccessible  
after pinning | Access to `&mut T`,  
unwrapping `T` | varies, but usually:  
Access to `&mut T`, removing `T`,  
anything that would reallocate `T` | Functions that require `&mut T` |
| Unchanged APIs  
(examples) | Access to `&T` | Access to `&T`,  
dropping `T` in place | Functions that require `&T` |
| `: Clone` | usually | possibly  
`where T: Clone`¹ | varies |

¹ If implemented that way, then `pub fn clone_unpinning(this: &Pin<Self>) -> Self { … }` can also be implemented. However, if `T: Clone`, then it's likely that `T` is also `Unpin`, which makes pinning pretty much useless.  
See the end of this post for a more useful implementation that can clone meaningfully pinned instances also.

Which functions require `Pin<&T>`?
==================================

How `Pin<&T>` and `Pin<&mut T>` are used varies, but there are three broad categories most cases fall into:

Avoiding reference-counting
---------------------------

If smart pointers to an instance are copied often but accessed rarely, and references cannot be used because their lifetime can't be constrained statically, then it makes sense to shift the runtime cost from cloning the pointers into a validity check on access instead. The smart pointers are replaced by copiable handles, in this case.

How do the handles know when their target has disappeared? Pinning a `T` asserts that `<T as Drop>::drop` will run _for that particular instance_, so there will be an opportunity to notify a longer-lived registry.

This also enables use cases where the handles cannot be dropped explicitly, like if they are stored directly by an arena allocator like [`bumpalo`](https://crates.io/crates/bumpalo). You can see an example of this pattern in my crate [`lignin`](https://docs.rs/lignin/0.1/lignin/), which supports (V)DOM callbacks this way.

Embedding externally-managed data
---------------------------------

My crate [`tiptoe`](https://lib.rs/crates/tiptoe) stores its smart pointers' reference counts directly inside the hosted value instances. Pinning allows them to still expose an exclusive reference as `Pin<&mut T>`.

You can read more about intrusive reference-counting and the heap-only pattern it enables in this earlier post:

  Intrusive Smart Pointers + Heap Only Types = �� [

Intrusive Smart Pointers + Heap Only Types = �� Simplify your API and safely clone an \`Arc\` through \`&Value\`. Pinning-friendly intrusive smart pointer crate included! blog.schichler.dev

](https://blog.schichler.dev/intrusive-smart-pointers-heap-only-types-ckvzj2thw0caoz2s1gpmi1xm8)

Persisting self-references
--------------------------

Consider the following [`async`](https://rust-lang.github.io/async-book/03_async_await/01_chapter.html) block:

Copy

```
async {
    let a = "Hello, future!";
    let b = &a;
    yield_once().await;
    println!("{}", b);
}

```

This code creates an opaquely-typed [`Future`](https://doc.rust-lang.org/stable/core/future/trait.Future.html) instance that will, at least formally, contain a reference to another of its fields after it is [polled](https://doc.rust-lang.org/stable/core/future/trait.Future.html#the-poll-method) for the first time.

The instance won't be externally borrowed anymore at that time, but moving it would break the private reference `b` to `a`, so `Future::poll(…)` requires `self: Pin<&mut Self>` as receiver.

This ensures that instances of `impl Future` will only enter such a state when they are already guaranteed not to be moved again. If an executor does need to move `Future`s, it can require `Future + Unpin` instead, [which allows converting `&mut _` to `Pin<&mut _>` on the fly](https://doc.rust-lang.org/stable/core/pin/struct.Pin.html#method.new).

Side note: Pinning is a huge deal for safe `async` performance! _As even instances of eventually self-referential `impl Future`s start out unpinned, they are directly composable_ without workarounds like lifting their state onto the heap on demand. This results in less fragmentation, less unusual control flow and smaller code size (before inlining, at least) in the generated state machines, all of which makes it much easier to evaluate them quickly. (Or rather: It raises the ceiling for what an async runtime can reasonably achieve, as it won't be held back by generated code it has no control over. While a simple async runtime is fairly easy to write in Rust, great ones are schedulers that operate very "close to the metal" and as such are often strongly affected by hardware quirks. Their development is quite interesting to follow.)

Weakening non-moveability
=========================

For the final section of this post, let's take a step back and look at the initial pinning guarantee again.

While pinned instances can't be moved safely by other _unrelated_ code, it's still often possible to provide functions like the following:

Copy

```
/// Swaps two pinned instances, making adjustments as necessary.
pub fn swap(
    a: Pin<&mut Self>,
    b: Pin<&mut Self>,
) { … }

```

As `swap` has access to `Self`'s private fields (and can rely on internal implementation details regarding how `Self` makes use of pinning _exactly_), it's able to patch any self-referential or global instance registry pointers as necessary during the exchange.

It's also possible to similarly recreate the rest of C++'s address-aware value movement infrastructure, as pointed out by Miguel Young in April in [Move Constructors in Rust: Is it possible?](https://mcyoung.xyz/2021/04/26/move-ctors/#towards-move-constructors-copy-constructors) and implemented in the [`moveit`](https://docs.rs/moveit/0.3.0/moveit/) crate.

In addition to a Rust program more nicely interfacing with C++ this way, pinning collections can also use `moveit`'s [`MoveNew`](https://docs.rs/moveit/0.3.0/moveit/trait.MoveNew.html) and [`CopyNew`](https://docs.rs/moveit/0.3.0/moveit/trait.CopyNew.html) traits to port part of their non-pinning API to their pinning interface in a more Rust-like fashion:

Copy

```
impl<T> C<T> {
    pub fn push_pin(
        this: &mut Pin<Self>,
        value: T,
    ) -> Pin<&mut T>
    where T: MoveNew {
        todo!("Potentially reallocate existing items.")
    }

    pub fn push_pinned(
        this: &mut Pin<Self>,
        value: Pin<MoveRef<'_, T>>,
    ) -> Pin<&mut T>
    where T: MoveNew {
        todo!("Potentially reallocate existing items, move new item.")
    }
}

// Careful: This also enables `Clone` for `Pin<C<T>>`!
impl<T: CopyNew> Clone for C<T> {
    pub fn clone(self) -> Self {
        todo!("Clone items in an address-aware way.")
    }
}

```

Collections with stable backing storage can often accept new values regardless of whether they are currently pinning, but a pinning [`Vec`](https://doc.rust-lang.org/stable/alloc/vec/struct.Vec.html)\-like for example sometimes has to reallocate and as such must allow its values to patch pointers to expand arbitrarily.

The `CopyNew` trait can be implemented more broadly than the standard [`Clone`](https://doc.rust-lang.org/stable/core/clone/trait.Clone.html), which can't be used where internal pointers or certain types of back-reference may exist (e.g. non-owning "multicast"-like references to the instance in question).

* * *

Thanks
======

To [Robin Pederson](https://hashnode.com/@TheBerkin) and [telios](https://hashnode.com/@telios) for proof-reading and various suggestions on how to improve clarity, and to [Milou](https://github.com/jimkoen) for criticism and suggestions from a C++ perspective.

License and Translations
========================

This post as a whole with exception of citations is licensed under [CC BY-NC-SA 2.0](https://creativecommons.org/licenses/by-nc-sa/2.0/). All code samples (that is: code blocks and snippets formatted `like this`), except for citations, are additionally licensed under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/), so that you can freely use them in your projects under any license or no license.

Citations from official Rust projects retain their original `MIT OR Apache-2.0` license and are used as permitted at [https://www.rust-lang.org/policies/licenses](https://www.rust-lang.org/policies/licenses). Sorry about the complexity here, unfortunately my country barely recognises fair use.

If you translate this post, please let me know so that I can link it here. I should be able to post a German translation myself before long.

(I suggest using the same license structure for code snippets in translations as here, though this is not something I can enforce. If a translation uses a different license, you can likely still take the code you need from the original here under CC0.)

[#rust](https://hashnode.com/n/rust)[#patterns](https://hashnode.com/n/patterns)[#api](https://hashnode.com/n/api)

![](data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iNjQiIGhlaWdodD0iNjQiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgdmVyc2lvbj0iMS4xIi8+)
![](https://blog.schichler.dev/_next/image?url=https%3A%2F%2Fcdn.hashnode.com%2Fres%2Fhashnode%2Fimage%2Fupload%2Fv1594643783311%2FZ4fzAt9ln.png%3Fh%3D64%26w%3D64%26fit%3Dcrop%26crop%3Dentropy%26auto%3Dcompress%26auto%3Dcompress%2Cformat%26format%3Dwebp&w=128&q=75)
1

