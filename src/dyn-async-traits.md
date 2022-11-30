# dyn async traits 系列

从 [@Niko Matsakis](https://github.com/nikomatsakis) 的 [博客](https://smallcultfollowing.com/babysteps/) 上汇总并翻译《dyn async traits》系列。

Niko 是 Rust 语言诸多特性的设计者（比如 NLL）。

而这个系列主要探索在 trait 中支持 `async fn`，因此主要聚焦于思路梳理与原型设计。
