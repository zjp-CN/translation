# Dyn async traits

[原文](https://smallcultfollowing.com/babysteps/blog/2022/01/07/dyn-async-traits-part-7/) |
日期：2022-01-07 19:37 -0500

大家好！欢迎来到 2022 年！去年年底，Tyler Mandry 和我围绕支持 dyn async trait 做了大量迭代工作
—— 也就是让使用 `async fn` 的 trait 变成 dyn 安全 —— 我们开始对我们的设计感觉很好。

本文是讨论我们现有成果的几篇博客文章的第一篇。

在这篇文章中，我将重申我们的目标，并给出设计的高级大纲。接下来的几篇帖子将更深入地探讨细节和下一步。

## 目标：让 traits 具有“正常”工作方式的 async fn

距离上一篇关于 dyn trait 的帖子已经有一段时间了，所以让我们从回顾总体目标开始：我们的愿景是允许 `async fn` 像 `fn` 一样在 traits 中使用。

例如，我们希望拥有一个异步版本的 `Iterator` trait[^stream]，大致如下所示：

[^stream]: 习惯上被称作 `Stream`。

```rust,ignore
trait AsyncIterator {
    type Item;
    
    async fn next(&mut self) -> Self::Item;
}
```

你应该能够像使用任何其他特征一样使用这个 `AsyncIterator` trait。当然，静态分发和 `impl Trait` 应该都可以工作：

```rust,ignore
async fn sum_static(mut v: impl AsyncIterator<Item = u32>) -> u32 {
    let mut result = 0;
    while let Some(i) = v.next().await {
        result += i;
    }
    result
}
```

但动态分发也应该起作用：

```rust,ignore
async fn sum_dyn(v: &mut dyn AsyncIterator<Item = u32>) -> u32 {
    //               ^^^
    let mut result = 0;
    while let Some(i) = v.next().await {
        result += i;
    }
    result
}
```

## 另一个目标：让 dyn 比现有的更干净

虽然我们一开始的目标是改进 `async fn`，但让 `dyn Trait` 在整体上更易用也是我们的兴趣。

原因有几个。首先，`async fn` 本身就是一个返回 `impl Trait` 的函数的语法糖，所以让 trait 中的 `async fn` 工作相当于让 [RPITIT][^RPITIT] 工作。

[RPITIT]: https://rust-lang.github.io/impl-trait-initiative/explainer/rpit.html
[^RPITIT]: 即 [return position impl trait in traits][RPITIT] —— traits 中函数的返回位置上支持 impl Trait。

但是，现有的 `dyn Trait` 设计包含了一些令人沮丧的限制，所以我们希望一个能尽可能多的改进这些限制的设计。

目前，我们的方案取消了以下限制，使使用这些功能的 trats 仍然与 `dyn` 兼容：

* 只要 `Trait` 是 dyn safe，返回位置就支持 `impl Trait`
    * 例如 `fn get_widget(&self) -> impl Iterator<Item = Widget>`
    * 如上所述，这意味着 `async fn` 能工作，因为它解糖成 `impl Future<Outut = ...>`
* 只要 `Trait` 是 dyn safe，参数位置就支持 `impl Trait`
    * 例如 `fn process_widgets(&mut self，items: impl Iterator<Item = Widget>)`
* 按值传入 self 的方法
    * 例如 `fn process(self)` 和 `d: Box<dyn Trait>` 都能够调用 `d.process()`
    * 最终这将扩展到其他像 Box 一样的智能指针

如果你把这三个放在一起，它代表 dyn safe 在 Rust 中的一个相当大的扩展。

下面的例子是 dyn safe，它以一种自然的方式将所有这些东西结合在一起：

```rust,ignore
trait Widget {
    async fn augment(&mut self, component: impl Into<WidgetComponent>);
    fn components(&self) -> impl Iterator<Item = WidgetComponent>;
    async fn transmit(self, factory: impl Factory);
}
```

## 最终目标：在没有分配器的情况下也能工作

支持 [RPITIT] 最直接的方式是分配一个 `Box` 来存储返回值。在大多数情况下，这是很好的。但在一些用例中，它不是一个好的选择：

* 在内核中，你希望使用自定义分配器
* 在紧循环 (tight loop) 中，分配的性能成本太高
* 在极端的嵌入式情况下，你根本没有分配器

因此，我们希望确保可以使用支持异步函数或 RPITIT 的特征，而不需要分配器，尽管这样做需要更多的工作。以下是想要支持这种情况的一些替代策略：

* 预先分配栈空间：当你创建 `dyn Trait` 时，你在栈上预留了一些空间来存储它可能返回的任何 Future 或 `impl Trait`
* 缓存：重用相同的 `Box` 以减少对性能的影响（一个好的分配器可以为你做这件事，但并不是所有的系统都带有高效的分配器）
* 封装的 trait：你只为你需要的类型派生一个包装器枚举体

不过，最终我们可以管理动态分发的方法的数量没有限制，因此我们目标不是有一套“内置”的策略，而是允许人们使用过程宏开发自己的策略集。

然后，我们可以在工具库甚至标准库中提供最常见的策略，同时如果他们有非常特殊的需求，还允许人们开发自己的策略。

## 设计的流程图

我画了一个流程图来说明我们的设计是如何在高层次上工作的（需使用明亮主题才能看清图下方的文字）：

<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1" width="431px" viewBox="-0.5 -0.5 431 731" content="&lt;mxfile host=&quot;app.diagrams.net&quot; modified=&quot;2022-01-08T00:40:35.768Z&quot; agent=&quot;5.0 (Macintosh)&quot; etag=&quot;ByO0M6C-FHR3Zr7-2PQV&quot; version=&quot;16.1.2&quot; type=&quot;google&quot;&gt;&lt;diagram id=&quot;DfosBROBM4uyRER-NLTF&quot; name=&quot;Page-1&quot;&gt;7VrZcpswFP0aP7bDYrD9GDtJO9O96bTNowwC1AhEhYjtfn0lEDsY0oESd/qSSEdCy7mr7nih7/zjKwpC7x2xIV5oin1c6NcLTVsvVf5XAKcUWOmrFHApslNILYA79AtKUJFojGwYVSYyQjBDYRW0SBBAi1UwQCk5VKc5BFd3DYELG8CdBXAT/YZs5slrGUqBv4bI9bKdVUWO+CCbLIHIAzY5lCD9ZqHvKCEsbfnHHcSCu4yX9LvbjtH8YBQGbMgHj5+Ce3h/89m4Xb35oG7CK2xvXmhmuswjwLG8sTwtO2UUUBIHNhSrKAt9e/AQg3chsMTogcucYx7zMe+pvOkgjHcEE5p8q0PVNuCK4xGj5AGWRjbmSgcmH5EHgJTBY+fV1JwwrmiQ+JDRE58iP9AlxadcCGn/UEhMXUvMK0nLyGQDpJa4+dIFkbwhuXwKr8a0vDprC1pWG6/7tbE0lHF41dbPjlhVb/AIbW6xskso84hLAoBvCnRbZbqY85aQUPL7AzJ2ku4HxIxU2ed80dN3+X3SuRedl0bWvT6WB69PspeeVRzwvAT4fUhMLXjm4kvp/AB1ITtHUIdIKcSAocfqQUYXz7JF7U3Mz7tFvOGKxlcG9lwqEua75CMNyXKXGYpmSIkFo6jfSvbAenATaX+IGUYBlPgYtqBUbUEzm7awaTEFcypL0Jse5S9YgkMCdgt8hAUNXyG1QQDGVXRjoKJrmzkVvenfdwBjSDt12CZW7Cdk9CpxKqO3ezye9tYdud5U3jY/vp7Mja/mUN4RtVTNktFef2zMqabZMVscso0eC8db89FX1E2VtdVL51jLEudWBTYIGb80CbKhPW1ZdeBO5WnAFzYU7CPxz6HEF1euxJm+nTpzM7XfYMfIYLX+6NJmoJNFl5Z89cIMdGgcUc1ZDbTtoVBT8PeE+kCs5MSBlZoPf8hyqsXiosM8KJp+iP9Q3XsUHERh+rZ20FGIdwqN1+cOSdq8L4u889dfFqo5NOOa9WmhtpUqapbyGbKYBvkkfopSyFE49xceFpba3GFBaRHC+BkvJVJo+vULUWCYIgc2BnI5ncNZXniI1YbmwKV65wyOQ+vOgZuOg53Cht+4xIRy/dwcx6pTCElurwCM3CAZMH/Goia+xdBhRa/O9j4Dsne38hAk1f6rkjz2LTLqerQ8+QQxriM4l/8XrkYRX5M4eWKmfKEAMbGPfFxFL0unwm2vnuYOY56/yaBNkkMHhEk6/4jNPl4yWtKs1RKbw/NUFMhHCi0UwcR3VCx2J3h2qjlyOiWqSeBJrNesmNsjq5pqtf4eEFFyrBTrJZQJJxGJvhW2jfjVryTsI9tO/HubZ6j6/DHSiqpzMJcDy/p5DXT8YuYsyff0xcy0SDkgRC5njZCb/ghZlIQuPaleP7NSi95dAhjDscPRHPv/MJmyWU80WvX9f7D8R4Jl3V9MGi15t/hFSDJW+lmNfvMb&lt;/diagram&gt;&lt;/mxfile&gt;" onclick="(function(svg){var src=window.event.target||window.event.srcElement;while (src!=null&amp;&amp;src.nodeName.toLowerCase()!='a'){src=src.parentNode;}if(src==null){if(svg.wnd!=null&amp;&amp;!svg.wnd.closed){svg.wnd.focus();}else{var r=function(evt){if(evt.data=='ready'&amp;&amp;evt.source==svg.wnd){svg.wnd.postMessage(decodeURIComponent(svg.getAttribute('content')),'*');window.removeEventListener('message',r);}};window.addEventListener('message',r);svg.wnd=window.open('https://viewer.diagrams.net/?client=1&amp;page=0&amp;edit=_blank');}}})(this);" style="cursor:pointer;max-width:100%;max-height:731px;"><defs/><g><rect x="0" y="0" width="180" height="520" fill="#e1d5e7" stroke="#9673a6" pointer-events="all"/><rect x="250" y="0" width="180" height="520" fill="#f8cecc" stroke="#b85450" pointer-events="all"/><path d="M 260 180 L 280 180 L 270 180 L 283.63 180" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 288.88 180 L 281.88 183.5 L 283.63 180 L 281.88 176.5 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="170" y="150" width="90" height="60" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><path d="M 179 150 L 179 210 M 251 150 L 251 210" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 70px; height: 1px; padding-top: 180px; margin-left: 180px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Vtable</i></div></div></div></foreignObject><text x="215" y="184" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Vtable</text></switch></g><path d="M 90 100 L 90 143.63" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 90 148.88 L 86.5 141.88 L 90 143.63 L 93.5 141.88 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><path d="M 50 20 L 130 20 L 130 88 Q 110 66.4 90 88 Q 70 109.6 50 88 L 50 32 Z" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 48px; margin-left: 51px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">Caller</div></div></div></foreignObject><text x="90" y="52" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Caller</text></switch></g><path d="M 330 210 L 330 230 L 330 200 L 330 213.63" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 330 218.88 L 326.5 211.88 L 330 213.63 L 333.5 211.88 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="290" y="150" width="80" height="60" rx="9" ry="9" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 180px; margin-left: 291px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><div><i>Argument</i></div><div><i>adaptation<br /></i></div><i> from vtable<br /></i></div></div></div></foreignObject><text x="330" y="184" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Argument...</text></switch></g><path d="M 330 300 L 330 320 L 330 290 L 330 303.63" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 330 308.88 L 326.5 301.88 L 330 303.63 L 333.5 301.88 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="290" y="220" width="80" height="80" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 260px; margin-left: 291px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Normal function found in the impl<br /></i></div></div></div></foreignObject><text x="330" y="264" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Normal functi...</text></switch></g><path d="M 290 340 L 136.37 340" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 131.12 340 L 138.12 336.5 L 136.37 340 L 138.12 343.5 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="290" y="310" width="80" height="60" rx="9" ry="9" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 340px; margin-left: 291px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Return value adaptation to vtable<br /></i></div></div></div></foreignObject><text x="330" y="344" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Return value...</text></switch></g><path d="M 50 410 L 130 410 L 130 478 Q 110 456.4 90 478 Q 70 499.6 50 478 L 50 422 Z" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" transform="rotate(-180,90,450)" pointer-events="all"/><path d="M 90 370 L 90 403.63" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 90 408.88 L 86.5 401.88 L 90 403.63 L 93.5 401.88 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="50" y="310" width="80" height="60" rx="9" ry="9" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 340px; margin-left: 51px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Return type adaptation from vtable<br /></i></div></div></div></foreignObject><text x="90" y="344" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Return type a...</text></switch></g><rect x="0" y="530" width="180" height="200" fill="none" stroke="none" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe flex-start; width: 178px; height: 1px; padding-top: 630px; margin-left: 2px;"><div style="box-sizing: border-box; font-size: 0px; text-align: left;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><div align="left"><b>Caller knows:</b></div><div align="left"><ul><li>Types of impl Trait arguments.</li></ul></div><div align="left"><b>Caller does not know:</b></div><ul><li>Type of the callee.</li><li>Precise return type, if function returns impl Trait.</li></ul></div></div></div></foreignObject><text x="2" y="634" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px">Caller knows:...</text></switch></g><path d="M 130 180 L 163.63 180" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="stroke"/><path d="M 168.88 180 L 161.88 183.5 L 163.63 180 L 161.88 176.5 Z" fill="rgb(0, 0, 0)" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"/><rect x="50" y="150" width="80" height="60" rx="9" ry="9" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 78px; height: 1px; padding-top: 180px; margin-left: 51px;"><div style="box-sizing: border-box; font-size: 0px; text-align: center;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><i>Argument adaptation to vtable<br /></i></div></div></div></foreignObject><text x="90" y="184" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">Argument adap...</text></switch></g><rect x="250" y="530" width="180" height="200" fill="none" stroke="none" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe flex-start; width: 178px; height: 1px; padding-top: 630px; margin-left: 252px;"><div style="box-sizing: border-box; font-size: 0px; text-align: left;" data-drawio-colors="color: rgb(0, 0, 0); "><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><div align="left"><b>Callee does not know:</b></div><div align="left"><ul><li>Types of impl Trait arguments.</li></ul></div><div align="left"><b>Callee knows:<br /></b></div><ul><li>Type of the callee.</li><li>Precise return type, if function returns impl Trait.</li></ul></div></div></div></foreignObject><text x="252" y="634" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px">Callee does not know:...</text></switch></g></g><switch><g requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"/><a transform="translate(0,-5)" xlink:href="https://www.diagrams.net/doc/faq/svg-export-text-problems" target="_blank"><text text-anchor="middle" font-size="10px" x="50%" y="100%">Viewer does not support full SVG 1.1</text></a></switch></svg>

1. 调用方 (caller) 有权访问某种 `dyn  Trait`，如 `w: &mut Widget`，并希望调用一个方法，如 `w.argument()`
2. 调用方在 vtable 中查找函数并调用它
     * 但 argument 接收一个 `impl Into<WidgetComponent>`，这意味着它是一个泛型函数。通常，每个 `Into`
       类型都有一个单独的函数副本！但我们必须让 vtable 只有一个副本！该怎么做呢？
     * 答案是 vtable 给一个副本进行编码，该副本期望“某种指针指向 `dyn Into<WidgetComponent>`”。这可以是一个 `Box`，但也可以是其他类型的指针：稍后我会详细介绍。
     * 因此，调用者的任务是创建一个“指向 `dyn Into<WidgetComponent>` 
       的指针”。它之所以能做到这一点，是因为它知道所提供的值的类型；在本例中，它将通过在栈上分配一些内存空间来做到这一点。
3. 同时，vtable 包含指向要调用的正确函数的指针。但它不是从 impl 直接指向函数的指针：它是包装该函数的轻量级填充程序。该填充程序负责将 vtable
   的 ABI 转换为用于静态分发的标准 ABI。
4. 同时，当函数返回时，它还带回了某种 Future。被调用方 (callee) 知道该类型，但调用方不知道。因此，被调用方的任务是将其转换为“某种指向
   `dyn Future` 的指针”，并将该指针返回给调用方。
     * 默认情况下，将其放入 Box，但被调用方可以自定义该类型以使用其他策略。
5. 调用方取回其“指向 `dyn Future` 的指针”，并且等待该类型，即使它不确切地知道等待什么样的 Future。

## 即将发布的帖子

在接下来的博客文章中，我将详述我在上述提到的几件事：

* 指向 `dyn Trait` 的指针：
    * 我们到底如何编码“某种类型的指针”？这是什么意思？
    * 这真的很关键，因为我们需要能够支持它
* `impl Trait` 参数的转化器 (adaptation)：
    * 对于泛型类型参数，如何转化成 vtable，或者从 vtable 中转化回来？
    * 提示：它涉及为参数创建一个 `dyn Trait`
* `impl Trait` 返回值参数的转化器：
    * 对于泛型类型参数，如何转化成 vtable，或者从 vtable 中转化回来？
    * 提示：它涉及返回一个 `dyn Trait`，可能但不一定以 `Box` 方式
* 按值 self 的转化器：
    * 对于按值的 self，如何转化成 vtable，或者从 vtable 中转化回来？什么时候可调用这些函数？
* Boxing 及其替代方法：
    * 当你通过动态分发调用异步函数或者返回 `impl Trait` 的函数时，默认将分配一个 `Box`，但我们已经看到这并不适用于所有人。
    * 如何方便地选择另一种策略，如栈预分配，以及人们如何创建自己的策略？

我们还将更新异步基础计划 ([async fundamentals initiative]) 页面，以提供更详细的设计文档。

[async fundamentals initiative]: https://rust-lang.github.io/async-fundamentals-initiative

## 附录：我仍希望看到的东西

我对我们在这一轮工作中的进展感到非常兴奋，但它并没有达到我最终想要的水平。

我的最终目标是，人们能够像使用 `impl Trait` 那样方便地使用动态分发，但我不完全确定如何实现这一目标。

这意味着能够编写不会涉及 `Box` 与 `&` 或其他细节的函数签名，当涉及 `impl Trait` 时，你不必处理这些细节。这也意味着不必太担心 `Send/Sync` 和生命周期。

如果我们能弄清楚如何做到的话，以下是我希望看到的一些改进：

* 支持 `Clone`：
    * 给定 `Widget: Clone` 和 `w: Box<dyn Widget>`，能够调用 `w.clone()`
    * 这几乎可以工作，但 `trait Clone: Sized` 这一事实让它变得困难
* 支持“部分 dyn safe” traits：
    * 现在，dyn safe 要么全是，要么不是。这有一个很好的含义，`dyn Foo: Foo` 适用于所有类型。然而，它也有局限性，许多人告诉我，他们觉得这很令人困惑。
    * 此外，`dyn Foo` 不是 `Sized` 的，因此，虽然在概念上 `dyn Foo` 实现了 `Foo` 很酷，但实际上不能像使用大多数其他类型一样使用 `dyn Foo`
* 改进 `Send` 与返回值的交互方式（例如，RPIT、traits 中的异步函数等）：
    * 如果你写 `dyn Foo + Send` 的话
* 避免过多地谈论指针
    * 当你使用 `impl Trait` 时，如今你获得了真正符合人体工程学的体验：
        * `fn apply_map(map_fn: impl FnMut(u32) -> u32)`
        * `fn items(&self) -> impl Iterator<Item = Item> + '_`
    * 相比之下，当你使用 `dyn Trait` 时，你最终不得不在很多细节上非常明确，你的调用者也必须改变：
        * `fn apply_map(map_fn: &mut dyn FnMut(u32) -> u32)`
        * `fn items(&self) -> Box<dyn Iterator<Item = Item> + '_>`
* 使 `dyn Trait` 感觉更参数化：
    * `struct Foo<T: Trait> { t: Box<T> }` 有一个很好的性质，它公开了 `T`。这意味着我们知道如果 `T: Send`，那么 `Foo<T>: Send`（假设 `Foo` 没有任何不是 `Send` 的字段）；如果 `T: 'static`，那么 `Foo<T>: 'static`，以此类推。这很酷。
    * 相反，`struct Foo { t: Box<dyn Trait> }` 隐藏了很多细节 —— 它不允许 `T` 包含任何引用，也不允许 `Foo` 为 `Send`。
* 让它健全 (sound)：
    * 关于 `dyn Trait` 有一些开放的健全性错误 (soundness bugs)，例如 [#57893]，我想关闭它们。这与列表中的其他内容相关。

[#57893]: https://github.com/rust-lang/rust/issues/57893
