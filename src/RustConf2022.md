# RustConf2022

## 开幕主题演讲

> 主题：*The experimentations/Strandardization Cycle*
>
> 视频：<https://www.youtube.com/watch?v=37yASSgrdGE&list=PL85XCvVPmGQhXeH3QiYct6eMLH1un1dcu>

### by *Josh Triplett*

博客：[joshtriplett.org](https://joshtriplett.org/)
推特：[@josh_triplett](https://twitter.com/josh_triplett)

Josh Triplett 是 Rust 语言设计、标准库和 Cargo 团队的开发人员。

他关注于建立友好、包容的社区来激发人们的活力，为系统性问题构建解决方案，以及用高级 Rust 编写低级系统代码。

Josh 之前曾在 Rust 峰会 (RustConf)、内核峰会 (Kernel Summit) 、linux.conf.au 等技术会议上发表演讲。

### 内容概要

常常面临以下情况：某个库创新的 API

- 具有比以往更好的编程体验
- 显然每个人都应该使用它
- 为什么像这样的东西在标准库里没有

回顾 Rust 错误处理的发展历史：

| crate         | 提供的功能                                      | 最早版本与发布时间    | 最新版本与发布时间      | 备注                                  |
|---------------|-------------------------------------------------|-----------------------|-------------------------|---------------------------------------|
| [quick-error] | 使用宏；每种错误为枚举体的成员；自动实现 traits | `0.1.0`<br>2015-09-09 | `2.0.1`<br>2021-05-07   | 伴随 Rust 1.0 出现                    |
| [error-chain] | 保留和展示导致错误的错误                        | `0.1.8`<br>2016-04-29 | `0.12.4`<br>2020-08-02  | [starting with error-chain]           |
| [failure]     | `#[derive(Fail)]`；更简单；捕获所有错误的类型   | `0.0.1`<br>2017-10-08 | `0.1.8`<br>2020-05-02   | 启发标准库的 `Error` trait            |
| [anyhow]      | 新的捕获所有错误的类型；基于标准库              | `1.0.0`<br>2019-10-07 | `1.0.64`<br>2022-09-04  | 启发语言设计：允许 main 返回 `Result`；很多标准库调整 |
| [thiserror]   | `#[derive(Error)]`；就像手写 Error 实现一样轻量 | `1.0.0`<br>2019-10-09 | `1.0.34`<br>2022-09-04  | 同上                                  |
| [eyre]        | 控制错误报告和上下文信息                        | `0.1.0`<br>2020-02-05 | `0.6.8`<br>2022-04-04|更多标准库调整： `provide` api、把 `Error` trait 移至 core|

从 `try!` 到 `?`，到将来标准库可能提供的捕获所有错误的类型 (catch-all type) 和 `#[derive(Error, Display)]`，整个演变过程清晰地展现了为什么
Rust 把一个东西标准化是如此谨慎和缓慢：
* 必须经过生态长期的自我演化把所有问题处理好，然后再把它们添加进 Rust
* 一步步添加，并且让已有的库依然能够工作

核心的一点：**Rust 与其生态共同演进，不断良性循环发展**。

这对于目前的 async/await 也不例外：在生态中实验，Rust 的使用者主要使用生态所提供的东西。成熟后，把宏之类的东西变成 Rust 原生语法。

虽然 async/await 稳定于 Rust 1.39 (2019-11-07)，但异步标准化仍在进展之中，比如 `AsyncWrite` trait 之类的东西。关于目前异步的梳理情况，见
本次大会的 Nick Cameron 的 [Async Rust] 演讲。

其他一些生态与标准库发展的例子：
* [boolinator] 启发 `bool::then` 和 `bool::then_some`
* [num_cpus] 启发 `std::thread::available_parallelism`
* 引入 safe 所有权和借用概念到 OS 资源上，从而引导生态

在应用程序和试验性库中进行试验，在 Rust 语言和标准库中实现标准化 —— 这两点持续循环。如果
* 太快稳定，就会导致无法修复的问题，并且受困于稳定性保证，从而扼杀 API 自身的创造性
* 太慢稳定，就会导致缺乏坚实的基础，充满不确定性，最终扼杀在 API 之上的创造性

所以，得到的教训是：必须在“基于 API 试验”和“在 API 上试验”两者之间权衡清楚，才开始标准化。

此外，谨防“第二系统效应” (second-system effect)[^second-system] —— 所以重新设计应该在标准库之外完成原型。

尽量保持标准库的生态中心地位不突出：使用更少的 nightly features，稳定只针对标准库的 features。

减少标准化造成的现有代码破坏：例如针对 [`Iterator::intersperse`] vs [`itertools::intersperse`] 重名问题，正在考虑
* [`cfg(accessible(...))`]
* 启用标准库版次 (editions)

> 我们认为，仅仅只有不犯错误的人来构建健壮的系统是不够的；最好提供工具和流程来捕捉和防止错误发生（来让每个人参与整个系统）。
>
> 我们的座右铭是“一种让每个人都能够构建可靠和高效软件的语言”。
>
> 我们希望人们感到被赋予能力在 Rust 项目中去改变、犯错、学习和成长，尽管他们对那些改变并非 100% 确信做得到。这就是我们所有人走到今天的方式！
>
> —— *[Inside Rust Blog: Imposter Syndrome](https://blog.rust-lang.org/inside-rust/2022/04/19/imposter-syndrome.html)*

一个思考是：“Rust 什么时候停止改变或者说完成”。这等价于问“Rust 何时不再与其生态一起演进”、“Rust 何时不再听取使用者的声音”、“Rust 何时开始消亡”。

提出这个问题的人其实在寻求一个稳定的基础，他们担心建造东西所依赖的基础不断变化。但 Rust 能够演进自身的同时提供稳定的基础：
* 不破坏 stable Rust 通过的代码
* 不破坏现有代码来支持新代码（但这意味着要学习更多的东西）

最后，谈论了“技术 ∙ 过程 ∙ 人”三者在 Rust 中如何妥善处理与改进。

[quick-error]: https://docs.rs/quick-error
[error-chain]: https://docs.rs/error-chain
[failure]: https://docs.rs/failure
[anyhow]: https://docs.rs/anyhow
[thiserror]: https://docs.rs/thiserror
[eyre]: https://docs.rs/eyre
[starting with error-chain]: https://brson.github.io/2016/11/30/starting-with-error-chain
[Async Rust]: https://www.youtube.com/watch?v=tHrvYtPNAHA&list=PL85XCvVPmGQhXeH3QiYct6eMLH1un1dcu
[boolinator]: https://docs.rs/boolinator
[num_cpus]: https://docs.rs/num_cpus
[`Iterator::intersperse`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.intersperse
[`itertools::intersperse`]: https://docs.rs/itertools/latest/itertools/trait.Itertools.html#method.intersperse
[`cfg(accessible(...))`]: https://github.com/rust-lang/rfcs/blob/master/text/2523-cfg-path-version.md

[^second-system]: 注：指在完成一个小型、优雅而成功的系统之后，人们倾向于对下一个计划有过度的期待，可能因此建造出一个巨大、有各种特色的怪兽系统。
                      第二系统效应可能造成软件专案计划过度设计，产生太多变数，过度复杂，无法达成期待，并因而失败。
