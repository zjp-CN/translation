---

## layout: post
title: "Dyn async traits, part 7: a design emerges?"
date: 2022-01-07 19:37 -0500

## 版面：帖子标题：“Dyn Async特征，第7部分：设计的出现？”日期：2022-01-0719：37-0500

Hi all! Welcome to 2022! Towards the end of last year, Tyler Mandry and I were doing a lot of iteration around supporting "dyn async trait" -- i.e., making traits that use `async fn` dyn safe -- and we're starting to feel pretty good about our design. This is the start of several blog posts talking about where we're at. In this first post, I'm going to reiterate our goals and give a high-level outline of the design. The next few posts will dive more into the details and the next steps.

大家好！欢迎来到2022年！去年年底，泰勒·曼德里和我围绕着支持“动态异步特性”做了大量迭代--也就是让使用“非同步特性”的特性变得安全--我们开始对我们的设计感觉很好。这是几篇博客文章的开头，讨论我们所处的位置。在第一篇文章中，我将重申我们的目标，并给出设计的高级大纲。接下来的几篇帖子将更深入地探讨细节和下一步。

## The goal: traits with async fn that work "just like normal"

## 目标：具有“正常”工作方式的非同步FN特征

It's been a while since my last post about dyn trait, so let's start by reviewing the overall goal: **our mission is to allow `async fn` to be used in traits just like `fn`**. For example, we would like to have an async version of the `Iterator` trait that looks roughly like this[^stream]:

距离我上一篇关于dyn特征的帖子已经有一段时间了，所以让我们从回顾总体目标开始：我们的使命是允许`async fn`像`fn`一样用于特征。例如，我们希望拥有一个异步版本的`Iterator‘特征，大致如下所示：

[^stream]: This has traditionally been called `Stream`.

这条河传统上被称为‘溪流’。

```rust
trait AsyncIterator {
    type Item;
    
    async fn next(&mut self) -> Self::Item;
}
```

You should be able to use this `AsyncIterator` trait in all the ways you would use any other trait. Naturally, static dispatch and `impl Trait` should work:

您应该能够像使用任何其他特征一样使用这个`AsyncIterator‘特征。当然，静态调度和`Impll Trait`应该可以工作：

```rust
async fn sum_static(mut v: impl AsyncIterator<Item = u32>) -> u32 {
    let mut result = 0;
    while let Some(i) = v.next().await {
        result += i;
    }
    result
}
```

But dynamic dispatch should work too:

但动态调度也应该起作用：

```rust
async fn sum_dyn(v: &mut dyn AsyncIterator<Item = u32>) -> u32 {
    //               ^^^
    let mut result = 0;
    while let Some(i) = v.next().await {
        result += i;
    }
    result
}
```

## Another goal: leave dyn cleaner than we found it

## 另一个目标：让戴恩比我们发现的更干净

While we started out with the goal of improving `async fn`, we've also had a general interest in making `dyn Trait` more usable overall. There are a few reasons for this. To start, `async fn` is itself just sugar for a function that returns `impl Trait`, so making `async fn` in traits work is equivalent to making [RPITIT](https://rust-lang.github.io/impl-trait-initiative/explainer/rpit.html) ("return position impl trait in traits") work. But also, the existing `dyn Trait` design contains a number of limitations that can be pretty frustrating, and so we would like a design that improves as many of those as possible. Currently, our plan lifts the following limitations, so that traits which make use of these features would still be compatible with `dyn`:

虽然我们一开始的目标是改进`Async fn`，但让`Dyn Trait`在整体上更易用也是我们的普遍兴趣。造成这种情况的原因有几个。首先，`async fn`本身就是一个返回`impl Trait`的函数的糖，所以让特征中的`async fn`起作用相当于让RPITIT(“返回位置在特征中实施特征”)起作用。但是，现有的‘Dyn Trait’设计包含了一些令人沮丧的限制，所以我们希望设计能改进尽可能多的这些限制。目前，我们的方案取消了以下限制，使使用这些功能的特征仍然与`dyn`兼容：

* Return position `impl Trait`, so long as `Trait` is dyn safe.
  * e.g., `fn get_widgets(&self) -> impl Iterator<Item = Widget>`
  * As discussed above, this means that `async fn` works, since it desugars 
* Argument position `impl Trait`, so long as `Trait` is dyn safe.
  * e.g., `fn process_widgets(&mut self, items: impl Iterator<Item = Widget>)`.
* By-value self methods.
  * e.g., given `fn process(self)` and `d: Box<dyn Trait>`, able to call `d.process()`
  * eventually this would be extended to other "box-like" smart pointers

If you put all three of those together, it represents a pretty large expansion to what dyn safety feels like in Rust. Here is an example trait that would now be dyn safe that uses all of these things together in a natural way:

返回位置`impl Trait`，只要`Trait`是dyn安全的，例如，`fn get_widget(&self)->impIterator<Item=Widget>`如上所述，这意味着`async fn`起作用，因为它将参数位置`impl Trait`去掉，只要`Trait`是动态安全的，例如，`fn Process_Widget(&mut self，Items：Iml Iterator<Item=Widget>)`.通过自身的方法，例如，给定`fn process(Self)`和`d：box<dyn特征>`，能够调用`d.process()`最终这将扩展到其他“盒状”智能指针如果你把这三个放在一起，它代表着Dyn安全在铁锈中的感觉的一个相当大的扩展。下面是一个例子，它现在是Dyn Safe，它以一种自然的方式将所有这些东西结合在一起：

```rust
trait Widget {
    async fn augment(&mut self, component: impl Into<WidgetComponent>);
    fn components(&self) -> impl Iterator<Item = WidgetComponent>;
    async fn transmit(self, factory: impl Factory);
}
```

## Final goal: works without an allocator, too, though you have to work a bit harder

## 最终目标：也可以在没有分配器的情况下工作，尽管你必须更努力一点

The most straightforward way to support [RPITIT](https://rust-lang.github.io/impl-trait-initiative/explainer/rpit.html) is to allocate a `Box` to store the return value. Most of the time, this is just fine. But there are use-cases where it's not a good choice:

支持RPITIT最直接的方式是分配一个`Box`来存储返回值。在大多数情况下，这是很好的。但在一些用例中，它不是一个好的选择：

* In a kernel, where you would like to use a custom allocator.
* In a tight loop, where the performance cost of an allocation is too high.
* Extreme embedded cases, where you have no allocator at all.

Therefore, we would like to ensure that it is possible to use a trait that uses async fns or RPITIT without requiring an allocator, though we think it's ok for that to require a bit more work. Here are some alternative strategies one might want to support:

在内核中，您希望使用自定义分配器。在紧循环中，分配的性能成本太高。在极端的嵌入式情况下，您根本没有分配器。因此，我们希望确保可以使用使用异步FNS或RPITIT的特征，而不需要分配器，尽管我们认为这样做需要更多的工作。以下是人们可能想要支持的一些替代策略：

* Pre-allocating stack space: when you create the `dyn Trait`, you reserve some space on the stack to store any futures or `impl Trait` that it might return.
* Caching: reuse the same `Box` over and over to reduce the performance impact (a good allocator would do this for you, but not all systems ship with efficient allocators).
* Sealed trait: you derive a wrapper enum for just the types that you need.

Ultimately, though, there is no limit to the number of ways that one might manage dynamic dispatch, so the goal is not to have a "built-in" set of strategies but rather allow people to develop their own using procedural macros. We can then offer the most common strategies in utility crates or perhaps even in the stdlib, while also allowing people to develop their own if they have very particular needs.

预先分配堆栈空间：当您创建`dyn Trait`时，您在堆栈上预留了一些空间来存储它可能返回的任何未来或`Impll Trait`。缓存：反复重用相同的`Box`以减少对性能的影响(一个好的分配器可以为您做这件事，但并不是所有的系统都带有高效的分配器)。封闭的特点：您只为您需要的类型派生一个包装器枚举。不过，最终，我们可以管理动态调度的方法的数量没有限制，因此目标不是有一套“内置”的策略，而是允许人们使用过程性宏开发自己的策略集。然后，我们可以在公用事业箱中甚至在stdlib中提供最常见的策略，同时还允许人们开发自己的策略，如果他们有非常特殊的需求。

## The design from 22,222 feet

## 设计高度为22,222英尺

I've drawn a little diagram to illustrate how our design works at a high-level:

我画了一个小图来说明我们的设计是如何在高层次上工作的：

<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1" width="431px" viewBox="-0.5 -0.5 431 731" content="&lt;mxfile host=&quot;app.diagrams.net&quot; modified=&quot;2022-01-08T00:40:35.768Z&quot; agent=&quot;5.0 (Macintosh)&quot; etag=&quot;ByO0M6C-FHR3Zr7-2PQV&quot; version=&quot;16.1.2&quot; type=&quot;google&quot;&gt;&lt;diagram id=&quot;DfosBROBM4uyRER-NLTF&quot; name=&quot;Page-1&quot;&gt;7VrZcpswFP0aP7bDYrD9GDtJO9O96bTNowwC1AhEhYjtfn0lEDsY0oESd/qSSEdCy7mr7nih7/zjKwpC7x2xIV5oin1c6NcLTVsvVf5XAKcUWOmrFHApslNILYA79AtKUJFojGwYVSYyQjBDYRW0SBBAi1UwQCk5VKc5BFd3DYELG8CdBXAT/YZs5slrGUqBv4bI9bKdVUWO+CCbLIHIAzY5lCD9ZqHvKCEsbfnHHcSCu4yX9LvbjtH8YBQGbMgHj5+Ce3h/89m4Xb35oG7CK2xvXmhmuswjwLG8sTwtO2UUUBIHNhSrKAt9e/AQg3chsMTogcucYx7zMe+pvOkgjHcEE5p8q0PVNuCK4xGj5AGWRjbmSgcmH5EHgJTBY+fV1JwwrmiQ+JDRE58iP9AlxadcCGn/UEhMXUvMK0nLyGQDpJa4+dIFkbwhuXwKr8a0vDprC1pWG6/7tbE0lHF41dbPjlhVb/AIbW6xskso84hLAoBvCnRbZbqY85aQUPL7AzJ2ku4HxIxU2ed80dN3+X3SuRedl0bWvT6WB69PspeeVRzwvAT4fUhMLXjm4kvp/AB1ITtHUIdIKcSAocfqQUYXz7JF7U3Mz7tFvOGKxlcG9lwqEua75CMNyXKXGYpmSIkFo6jfSvbAenATaX+IGUYBlPgYtqBUbUEzm7awaTEFcypL0Jse5S9YgkMCdgt8hAUNXyG1QQDGVXRjoKJrmzkVvenfdwBjSDt12CZW7Cdk9CpxKqO3ezye9tYdud5U3jY/vp7Mja/mUN4RtVTNktFef2zMqabZMVscso0eC8db89FX1E2VtdVL51jLEudWBTYIGb80CbKhPW1ZdeBO5WnAFzYU7CPxz6HEF1euxJm+nTpzM7XfYMfIYLX+6NJmoJNFl5Z89cIMdGgcUc1ZDbTtoVBT8PeE+kCs5MSBlZoPf8hyqsXiosM8KJp+iP9Q3XsUHERh+rZ20FGIdwqN1+cOSdq8L4u889dfFqo5NOOa9WmhtpUqapbyGbKYBvkkfopSyFE49xceFpba3GFBaRHC+BkvJVJo+vULUWCYIgc2BnI5ncNZXniI1YbmwKV65wyOQ+vOgZuOg53Cht+4xIRy/dwcx6pTCElurwCM3CAZMH/Goia+xdBhRa/O9j4Dsne38hAk1f6rkjz2LTLqerQ8+QQxriM4l/8XrkYRX5M4eWKmfKEAMbGPfFxFL0unwm2vnuYOY56/yaBNkkMHhEk6/4jNPl4yWtKs1RKbw/NUFMhHCi0UwcR3VCx2J3h2qjlyOiWqSeBJrNesmNsjq5pqtf4eEFFyrBTrJZQJJxGJvhW2jfjVryTsI9tO/HubZ6j6/DHSiqpzMJcDy/p5DXT8YuYsyff0xcy0SDkgRC5njZCb/ghZlIQuPaleP7NSi95dAhjDscPRHPv/MJmyWU80WvX9f7D8R4Jl3V9MGi15t/hFSDJW+lmNfvMb&lt;/diagram&gt;&lt;/mxfile&gt;" onclick="(function(svg){var src=window.event.target||window.event.srcElement;while (src!=null&amp;&amp;src.nodeName.toLowerCase()!='a'){src=src.parentNode;}if(src==null){if(svg.wnd!=null&amp;&amp;!svg.wnd.closed){svg.wnd.focus();}else{var r=function(evt){if(evt.data=='ready'&amp;&amp;evt.source==svg.wnd){svg.wnd.postMessage(decodeURIComponent(svg.getAttribute('content')),'*');window.removeEventListener('message',r);}};window.addEventListener('message',r);svg.wnd=window.open('https://viewer.diagrams.net/?client=1&amp;page=0&amp;edit=_blank');}}})(this);" style="cursor:pointer;max-width:100%;max-height:731px;"><defs/><g><rect x="0" y="0" width="180" height="520" fill="#e1d5e7" stroke="#9673a6" pointer-events="all"/><rect x="250" y="0" width="180" height="520" fill="#f8cecc" stroke="#b85450" pointer-events="all"/><path d="M 260 180 L 280 180 L 270 180 L 283.63 180" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 288.88 180 L 281.88 183.5 L 283.63 180 L 281.88 176.5 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="170" y="150" width="90" height="60" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><path d="M 179 150 L 179 210 M 251 150 L 251 210" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 70px; height: 1px; padding-top: 180px; margin-left: 180px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Vtable</i></div></div></div></foreignObject><text x="215" y="184" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Vtable</text></switch></g><path d="M 90 100 L 90 143.63" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 90 148.88 L 86.5 141.88 L 90 143.63 L 93.5 141.88 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><path d="M 50 20 L 130 20 L 130 88 Q 110 66.4 90 88 Q 70 109.6 50 88 L 50 32 Z" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 48px; margin-left: 51px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">Caller</div></div></div></foreignObject><text x="90" y="52" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Caller</text></switch></g><path d="M 330 210 L 330 230 L 330 200 L 330 213.63" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 330 218.88 L 326.5 211.88 L 330 213.63 L 333.5 211.88 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="290" y="150" width="80" height="60" rx="9" ry="9" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 180px; margin-left: 291px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><div><i>Argument</i></div><div><i>adaptation<br /></i></div><i> from vtable<br /></i></div></div></div></foreignObject><text x="330" y="184" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Argument...</text></switch></g><path d="M 330 300 L 330 320 L 330 290 L 330 303.63" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 330 308.88 L 326.5 301.88 L 330 303.63 L 333.5 301.88 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="290" y="220" width="80" height="80" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 260px; margin-left: 291px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Normal function found in the impl<br /></i></div></div></div></foreignObject><text x="330" y="264" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Normal functi...</text></switch></g><path d="M 290 340 L 136.37 340" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 131.12 340 L 138.12 336.5 L 136.37 340 L 138.12 343.5 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="290" y="310" width="80" height="60" rx="9" ry="9" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 340px; margin-left: 291px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Return value adaptation to vtable<br /></i></div></div></div></foreignObject><text x="330" y="344" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Return value...</text></switch></g><path d="M 50 410 L 130 410 L 130 478 Q 110 456.4 90 478 Q 70 499.6 50 478 L 50 422 Z" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" transform="rotate(-180,90,450)" pointer-events="all"/><path d="M 90 370 L 90 403.63" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 90 408.88 L 86.5 401.88 L 90 403.63 L 93.5 401.88 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="50" y="310" width="80" height="60" rx="9" ry="9" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 340px; margin-left: 51px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Return type adaptation from vtable<br /></i></div></div></div></foreignObject><text x="90" y="344" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Return type a...</text></switch></g><rect x="0" y="530" width="180" height="200" fill="none" stroke="none" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe flex-start; width: 178px; height: 1px; padding-top: 630px; margin-left: 2px;"><div style="box-sizing: border-box; font-size: 0px; text-align: left;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><div align="left"><b>Caller knows:</b></div><div align="left"><ul><li>Types of impl Trait arguments.</li></ul></div><div align="left"><b>Caller does not know:</b></div><ul><li>Type of the callee.</li><li>Precise return type, if function returns impl Trait.</li></ul></div></div></div></foreignObject><text x="2" y="634" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px">Caller knows:...</text></switch></g><path d="M 130 180 L 163.63 180" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 168.88 180 L 161.88 183.5 L 163.63 180 L 161.88 176.5 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="50" y="150" width="80" height="60" rx="9" ry="9" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 180px; margin-left: 51px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Argument adaptation to vtable<br /></i></div></div></div></foreignObject><text x="90" y="184" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Argument adap...</text></switch></g><rect x="250" y="530" width="180" height="200" fill="none" stroke="none" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe flex-start; width: 178px; height: 1px; padding-top: 630px; margin-left: 252px;"><div style="box-sizing: border-box; font-size: 0px; text-align: left;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><div align="left"><b>Callee does not know:</b></div><div align="left"><ul><li>Types of impl Trait arguments.</li></ul></div><div align="left"><b>Callee knows:<br /></b></div><ul><li>Type of the callee.</li><li>Precise return type, if function returns impl Trait.</li></ul></div></div></div></foreignObject><text x="252" y="634" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px">Callee does not know:...</text></switch></g></g><switch><g requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"/><a transform="translate(0,-5)" xlink:href="https://www.diagrams.net/doc/faq/svg-export-text-problems" target="_blank"><text text-anchor="middle" font-size="10px" x="50%" y="100%">Viewer does not support full SVG 1.1</text></a></switch></svg>

VableVtable调用程序从vableArgumentArgument改编...在inimplmal Normal函数中找到的Normal函数...返回值适配到vableReturn Value...从vableReturn type a...返回类型适配...调用者知道：Iml特征参数的类型。调用者不知道：调用者的类型。如果函数返回Impl Trait，则精确返回类型。调用者知道：...参数对vtable的适配Argument adap...被调用者不知道：Iml特征参数的类型。Callee知道：调用的类型。如果函数返回Impl Trait.Callee不知道：...查看器不支持完整的SVG 1.1

Let's walk through it:

让我们一起来了解一下：

1. To start, we have the caller, which has access to some kind of `dyn` trait, such as `w: &mut Widget`, and wishes to call a method, like `w.augment()`
1. The caller looks up the function for `augment` in the vtable and calls it:
   * But wait, augment takes a `impl Into<WidgetComponent>`, which means that it is a generic function. Normally, we would have a separate copy of this function for every `Into` type! But we must have only a single copy for the vtable! What do we do?
   * The answer is that the vtable encodes a copy that expects "some kind of pointer to a `dyn Into<WidgetComponent>`". This could be a `Box` but it could also be other kinds of pointers: I'm being hand-wavy for now, I'll go into the details later.
   * The caller therefore has the job of creating a "pointer to a `dyn Into<WidgetComponent>`". It can do this because it knows the type of the value being provided; in this case, it would do it by allocating some memory space on the stack.
1. The vtable, meanwhile, includes a pointer to the right function to call. But it's not a direct pointer to the function from the impl: it's a lightweight shim that wraps that function. This shim has the job of converting *from* the vtable's ABI into the standard ABI used for static dispatch.
1. When the function returns, meanwhile, it is giving back some kind of future. The callee knows that type, but the caller doesn't. Therefore, the callee has the job of converting it to "some kind of pointer to a `dyn Future`" and returning that pointer to the caller.
   * The default is to box it, but the callee can customize this to use other strategies.
1. The caller gets back its "pointer to a `dyn Future`" and is able to await that, even though it doesn't know exactly what sort of future it is.

## Upcoming posts

## 首先，我们有调用方，它有权访问某种`dyn`特征，如`w：&mut Widget`，并希望调用一个方法，如`w.Augment()`调用方在vtable中查找函数并调用它：但等等，Augment会`Iml Into<WidgetComponent>`，这意味着它是一个泛型函数。通常，我们会为每个`Into‘类型都有一个单独的函数副本！但我们必须为vtable只有一个副本！我们该怎么做呢？答案是vtable对一个副本进行编码，该副本期望“某种指向`dyn的指针变成<WidgetComponent>`”。这可以是一个`Box`，但也可以是其他类型的指针：我现在只是挥手示意，稍后我会详细介绍。因此，调用者需要创建一个“指向`dyn into<WidgetComponent>`的指针”。它之所以能做到这一点，是因为它知道所提供的值的类型；在本例中，它将通过在堆栈上分配一些内存空间来做到这一点。同时，vtable包含指向要调用的正确函数的指针。但它不是从IMPL直接指向函数的指针：它是包装该函数的轻量级填充程序。该填充程序负责将vtable的ABI转换为用于静态调度的标准ABI。同时，当函数返回时，它还带回了某种未来。被调用方知道该类型，但调用方不知道。因此，被调用方的任务是将其转换为“某种指向‘dyn Future`的指针”，并将该指针返回给调用方。默认情况下，将其装箱，但被调用方可以自定义该类型以使用其他策略。调用方取回其“指向`dyn Future`的指针”，并且能够等待该类型，即使它不确切地知道它是什么样的未来。即将发布的帖子

In upcoming blog posts, I'm going to expand on several things that I alluded to in my walkthrough:

在接下来的博客文章中，我将详述我在演练中提到的几件事：

* "Pointer to a `dyn Trait`":
  * How exactly do we encode "some kind of pointer" and what does that mean?
  * This is really key, because we need to be able to support 
* Adaptation for `impl Trait` arguments:
  * How do we adapt to/from the vtable for arguments of generic type?
  * Hint: it involves create a `dyn Trait` for the argument
* Adaptation for impl trait return values:
  * How do we adapt to/from the vtable for arguments of generic type?
  * Hint: it involves returning a `dyn Trait`, potentially boxed but not necessarily
* Adaptation for by-value self:
  * How do we adapt to/from the vtable for by-value self, and when are such functions callable?
* Boxing and alternatives thereto:
  * When you call an async fn or fn that returns `impl Trait` via dynamic dispatch, the default behavior is going to allocate a `Box`, but we've seen that doesn't work for everyone. How convenient can we make it to select an alternative strategy like stack pre-allocation, and how can people create their own strategies?

We'll also be updating the [async fundamentals initiative](https://rust-lang.github.io/async-fundamentals-initiative/) page with more detailed design docs.

“指向`dyn Trait`的指针”：我们到底如何编码“某种类型的指针”？这是什么意思？这真的很关键，因为我们需要能够支持对`impl Trait`参数的适配：我们如何适配泛型类型的参数的vtable/从vtable中适配？提示：它涉及为impl特征返回值的参数Adaptation创建一个`dyn Trait`：对于泛型类型的参数，我们如何适配到vtable/从vtable中适配：它涉及返回一个‘dyn Trait`，它可能被装箱，但不是必须的。yAdaptation对于按值self，我们如何适配到v表中/从vtable中适配而这样的函数什么时候是可调用的？装箱及其替代方法：当您通过动态调度调用返回`impl Trait`的异步fn或fn时，默认行为将分配一个`Box`，但我们已经看到这并不适用于所有人。我们如何方便地选择另一种策略，如堆栈预分配，以及人们如何创建自己的策略？我们还将更新异步基础计划页面，提供更详细的设计文档。

## Appendix: Things I'd still like to see

## 附录：我仍然希望看到的东西

I'm pretty excited about where we're landing in this round of work, but it doesn't get `dyn` where I ultimately want it to be. My ultimate goal is that people are able to use dynamic dispatch as conveniently as you use `impl Trait`, but I'm not entirely sure how to get there. That means being able to write function signatures that don't talk about `Box` vs `&` or other details that you don't have to deal with when you talk about `impl Trait`. It also means not having to worry so much about `Send/Sync` and lifetimes.

我对我们在这一轮工作中的进展感到非常兴奋，但它并没有达到我最终想要的水平。我的最终目标是，人们能够像使用‘Impl Trait`一样方便地使用动态调度，但我不完全确定如何实现这一目标。这意味着能够编写函数签名，这些签名不会谈论`Box`与`&`或其他细节，当您谈论`Impll Trait`时，您不必处理这些细节。这也意味着不必太担心`Send/Sync`和生命周期。

Here are some of the improvements I would like to see, if we can figure out how:

以下是我希望看到的一些改进，如果我们能弄清楚如何做到的话：

* Support clone:
  * Given trait `Widget: Clone` and `w: Box<dyn Widget>`, able to invoke `w.clone()`
  * This *almost* works, but the fact that `trait Clone: Sized` makes it difficult.
* Support "partially dyn safe" traits: 
  * Right now, dyn safe is all or nothing. This has the nice implication that `dyn Foo: Foo` for all types. However, it is also limiting, and many people have told me they find it confusing. Moreover, `dyn Foo` is not `Sized`, and hence while it's cool conceptually that `dyn Foo` implements `Foo`, you can't actually *use* a `dyn Foo` in the same way that you would use most other types.
* Improve how `Send` interacts with returned values (e.g., RPIT, async fn in traits, etc):
  * If you write `dyn Foo + Send`, that 
* Avoid having to talk about pointers so much
  * When you use `impl Trait`, you get a really ergonomic experience today:
    * `fn apply_map(map_fn: impl FnMut(u32) -> u32)`
    * `fn items(&self) -> impl Iterator<Item = Item> + '_`
  * In contrast, when you use dyn trait, you wind up having to be very explicit around lots of details, and your callers have to change as well:
    * `fn apply_map(map_fn: &mut dyn FnMut(u32) -> u32)`
    * `fn items(&self) -> Box<dyn Iterator<Item = Item> + '_>`
* Make dyn trait feel more parametric:
  * If I have an `struct Foo<T: Trait> { t: Box<T> }`, it has the nice property that it exposes the `T`. This means we know that `Foo<T>: Send` if `T: Send` (assuming `Foo` doesn't have any fields that are not send), we know that `Foo<T>: 'static` if `T: 'static`, and so forth. This is very cool.
  * In contrast, `struct Foo { t: Box<dyn Trait> }` bakes a lot of details -- it doesn't permit `t` to contain any references, and it doesn't let `Foo` be `Send`.
* Make it sound:
  * There are a few open soundness bugs around dyn trait, such as [\#57893](https://github.com/rust-lang/rust/issues/57893), and I would like to close them. This interacts with other things in this list.