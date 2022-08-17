# 贡献诊断翻译

> 原文：[Contribute to the diagnostic translation effort!]

[Contribute to the diagnostic translation effort!]: https://blog.rust-lang.org/inside-rust/2022/08/16/diagnostic-effort.html

## 翻译的诊断

Rust 诊断工作小组正在领导一项工作，以增加对编译器中错误消息国际化的支持，允许编译器以英语以外的语言生成输出。

例如，考虑以下诊断，其中用户使用冒号而不是箭头来指定函数的返回类型：

```rust,ignore
error: return types are denoted using `->`
 --> src/main.rs:1:21
  |
1 | fn meaning_of_life(): u32 { 42 }
  |                     ^ help: use `->` instead
```

可以用中文输出该诊断：

```rust,ignore
 --> src/main.rs:1:21
  |
1 | fn meaning_of_life(): u32 { 42 }
  |                     ^ 帮助: 使用`->`来代替
```

或者甚至使用西班牙语：

```rust,ignore
error: el tipo de retorno se debe indicar mediante `->`
 --> src/main.rs:1:21
  |
1 | fn meaning_of_life(): u32 { 42 }
  |                     ^ ayuda: utilice `->` en su lugar
```

翻译错误消息让非英语母语人士用其首选语言来使用 Rust。

## 目前的情况

实施诊断翻译已经开始，但我们正在寻求帮助！

在 `rustc` 中实现了诊断翻译的核心基础设施；这使得 Rust 可以发出带有翻译消息的诊断。

然而， `rustc` 中的每个诊断都必须移植才能使用这个新的基础设施，否则它们无法被翻译。

这是一项大量的工作，因此诊断工作组选择将翻译工作与向“诊断结构”（稍后将详细介绍）的过渡结合起来，并同时完成这两项工作。

一旦大多数诊断消息被移植到新的基础设施，诊断工作组就开始为翻译团队创建工作流程，来把所有诊断消息翻译成不同的语言。

此外，[这份](https://hackmd.io/@davidtwco/rkXSbLg95)文档列出了与诊断翻译相关的每个 PR。

## 请参与进来

在诊断翻译方面有很多工作要做，但好消息是，许多工作可以并行完成，而且它不需要编译器开发背景或熟悉 `rustc` 就能做出贡献！

如果你感兴趣，尽管开始吧！你可以在 zulipchat 的 [`#t-compiler/wg-diagnostics`] 中寻求帮助，或者联系 [`@davidtwco`]。

注意：本文章不会随工作组对诊断翻译工作流程的迭代和改进而更新，因此请始终参考开发人员指南的
《[diagnostic structs]》或《[diagnostic translation]》。

[`#t-compiler/wg-diagnostics`]: https://rust-lang.zulipchat.com/#narrow/stream/147480-t-compiler.2Fwg-diagnostics
[`@davidtwco`]: https://github.com/davidtwco
[diagnostic structs]: https://rustc-dev-guide.rust-lang.org/diagnostics/diagnostic-structs.html
[diagnostic translation]: https://rustc-dev-guide.rust-lang.org/diagnostics/translation.html

### (一) 设置本地开发环境

在协助诊断翻译工作之前，你需要设置你的开发环境，因此请遵循 `rustc` [开发指南][dev-guide-setup] 上的说明。

[dev-guide-setup]: https://rustc-dev-guide.rust-lang.org/building/how-to-build-and-run.html

### (二) 准备移植诊断

`rustc` 中几乎所有的诊断都是使用传统的 `DiagnosticBuilder` 接口实现的，如下所示：

```rust,ignore
self.struct_span_err(self.prev_token.span, "return types are denoted using `->`")
    .span_suggestion_short(
        self.prev_token.span,
        "use `->` instead",
        "->".to_string(),
        Applicability::MachineApplicable,
    )
    .emit();
```

`struct_span_err` 在给定两件事的情况下创建一个新的诊断：一个 `Span` 和一条消息。

`struct_span_err` 不会是你在编译器的源代码中遇到的唯一诊断函数，但其他函数都非常相似。

你可以在 `rustc` [开发指南][rustc-dev-error] 中阅读有关诊断基础设施的更多信息。

`Span` 只是识别用户源代码中的某个位置，整个编译器中都使用 `Span` 来报告诊断：例如，前面示例中
`self.prev_token.span` 应该是位置 `main.rs:1:21`。

在这个例子中，消息只是一个字符串字面值 (`&'static str`)，需要用同一消息的标识符来替换，不管请求的是哪种语言。

有两种方式可以将诊断程序移植到新的基础设施：

1. 如果这是一个简单的诊断，没有任何逻辑来决定是否添加建议、注意、帮助或标签，则[使用诊断派生宏]
2. [手动实现] `SessionDiagnostic`

在这两种情况下，诊断都表示为类型。使用类型表示诊断是诊断工作组的一个目标，因为它有助于将诊断逻辑与主代码路径分开。

每种诊断类型都应该实现 `SessionDiagnostic` （无论手动或自动）。在
`SessionDiagnostic` trait 中，有一个成员函数，它把此 trait 转换为要发出的 `Diagnostic`。

[rustc-dev-error]: https://rustc-dev-guide.rust-lang.org/diagnostics.html#error-messages
[使用诊断派生宏]: #使用诊断派生宏
[手动实现]: #手动实现-sessiondiagnostic

#### 使用诊断派生宏

诊断派生宏有：

- 用于整体诊断的 `SessionDiagnostic`
- 用于部分诊断的 `SessionSubdiagnostic`
- 用于 lints 的 `DecorateLint`

使用它们来自动实现诊断 trait。

首先，在以你的诊断命名的当前 crate 的 `errors` 模块中创建一个新类型，如
`rustc_typeck::errors` 或 `rustc_borrowck::errors`。这可能类似于：

```rust,ignore
struct ReturnTypeArrow {

}
```

接下来，添加包含我们需要的所有信息的字段，比如一个简单 `Span`：

```rust,ignore
struct ReturnTypeArrow {
    span: Span,
}
```

大多数情况下，字段是用来发出原始诊断逻辑的 `Span` 和插入到诊断消息中的值。

然后，添加派生宏和错误属性，并注释主要的 `Span` （它被传给 `struct_span_err`）。

```rust,ignore
#[derive(SessionDiagnostic)]
#[error(parser::return_type_arrow)]
struct ReturnTypeArrow {
    #[primary_span]
    span: Span,
}
```

每个诊断都应该有一个唯一的路径 （slug，本例中为 `parser::return_type_arrow`）。

习惯上，总是以与错误相关的 crate 开头（本例中为 `parser`）。

这个路径将用于在翻译资源中查找实际的诊断消息，很快就会介绍到。

最后，添加任何标签 (labels)、注意 (notes)、帮助 (helps) 或建议 (suggestions)：

```rust,ignore
#[derive(SessionDiagnostic)]
#[error(parser::return_type_arrow)]
struct ReturnTypeArrow {
    #[primary_span]
    #[suggestion(applicability = "machine-applicable", code = "->")]
    span: Span,
}
```

在本例中，只有一个建议：将 `:` 替换为 `->`。

在完成之前，还必须将诊断消息 [添加到翻译资源][adding-translation-resources] 中。

[adding-translation-resources]: #添加翻译资源

有关诊断派生宏的更多文档，请参考 `rustc` 开发指南的 [diagnostic structs] 一章。

#### 手动实现 `SessionDiagnostic`

有些诊断太复杂，无法使用派生宏从诊断类型生成诊断。此时，可以手动实现 `SessionDiagnostic`。

使用与诊断派生宏相同的类型，像下面那样来手动实现 `SessionDiagnostic`：

```rust,ignore
use rustc_errors::{fluent, SessionDiagnostic};

struct ReturnTypeArrow { span: Span }

impl SessionDiagnostic for ReturnTypeArrow {
    fn into_diagnostic(self, sess: &'_ rustc_session::Session) -> DiagnosticBuilder<'_> {
        sess.struct_span_err(
            self.span,
            fluent::parser::return_type_arrow,
        )
        .span_suggestion_short(
            self.span,
            fluent::parser::suggestion,
            "->".to_string(),
            Applicability::MachineApplicable,
        )
    }
}
```

不是像在原始诊断发出逻辑中那样对消息使用字符串，而是使用引用翻译资源的类型化标识符。

现在我们只需将诊断消息[添加到翻译资源][adding-translation-resources]。

#### PR 案例

针对移植到使用诊断派生宏或手动编写的诊断的更多示例，请参考以下 PR：

* <https://github.com/rust-lang/rust/pull/98353>
* <https://github.com/rust-lang/rust/pull/98415>
* <https://github.com/rust-lang/rust/pull/97093>
* <https://github.com/rust-lang/rust/pull/99213>

更多示例，请参考标记为 [`A-Translation`] 的 PR。

[`A-translation`]: https://github.com/rust-lang/rust/issues?q=is%3Aopen+label%3AA-translation+sort%3Aupdated-desc

#### 添加翻译资源

在诊断派生宏或手动实现的类型化标识符中，其 slug 都需要与翻译资源中的消息相对应。

`rustc` 使用的是 [Fluent]，这是一种非对称翻译系统。

编译器中发出诊断的每个 crate 都有一个对应的 Fluent 资源 `compiler/rustc_error_messages/locales/en-US/$crate.ftl`。

[Fluent]: http://projectfluent.org/

需要将错误消息添加到此资源，然后宏会生成对应于该消息的类型化标识符。

对于上述示例，我们应该向 `compiler/rustc_error_messages/locales/en-US/parser.ftl` 添加以下 Fluent 内容：

```fluent
parser_return_type_arrow = return types are denoted using `->`
    .suggestion = use `->` instead
```

`parser_return_type_arrow` 将（在 `rustc_error::fluent` 中）生成 `parser::return_type_arrow` 类型，用来与诊断结构体
(diagnostic struct) 和诊断构建器 (diagnostic builder) 一起使用。

子诊断 (subdiagnostics) 主要是 Fluent 消息的“属性”，习惯上，属性的名称是子诊断的类型，例如
`suggestion`，但如果一种子诊断存在多个时，属性的名称可以更改。

现在，Fluent 资源包含了该消息，我们的诊断程序被移植了！

使用插值的更复杂的消息能够引用诊断类型中的其他字段，当手动实现时，这些字段作为参数提供。

有关更多示例，请参考 `rustc` 开发指南中的 [diagnostic translation] 文档。

### (三) 移植诊断

你已经大致知道要做什么了，现在你需要找到一些诊断程序来移植。

有很多诊断需要移植，所以诊断工作组已经将工作分开，以避免任何人与其他人从事相同的诊断工作。

但现在，参与的人并不多，所以只需选择一个 crate 并开始移植 :)

请在你创建的任何 PR 中添加 [`A-translation`] 标签，这样我们就可以跟踪谁做出了贡献！

如果 PR 不是 `triagebot` 自动标记的话，你可以使用 `rustbot` 来标记 PR：

```text
@rustbot label +A-translation
```

你还可以指定诊断工作组的成员来审查你的 PR，做法是发布包含以下内容的评论（或将其包括在 PR 描述中）：

```text
r? rust-lang/diagnostics
```

即使你不确定如何继续，也尝试一下，你可以在 [`#t-compiler/wg-diagnostics`] 寻求帮助，或者联系 [`@davidtwco`]。

## FAQ

### 有人想要诊断翻译功能吗？

是！有些语言社区喜欢本地资源，有些则不喜欢（这些社区的偏好也会有所不同）。

例如，中文社区拥有成熟的编程语言资源生态系统，不需要懂任何英语。[^Chinese-speaking communities]

[^Chinese-speaking communities]: 译者注：嗯，说法有点绝对了 :)

### 翻译 xx 不是更有价值吗？

在 Rust 项目中有许多不同的领域，国际化将是有益的。

诊断并没有优先于项目的任何其他部分，只是编译器团队对支持这一功能感兴趣。

### 把编译器开发人员的时间花在其他地方不是更好吗？

编译器实现不是零和游戏：编译器其他部分的工作不会受到这些工作的影响，从事诊断翻译的工作也不会阻止贡献者从事其他工作。

### 翻译会是可选的吗？

是可选的。如果你不想看翻译，你就不需要使用它们。

### 使用者将如何选择语言？

使用者将如何选择使用翻译后的错误消息尚未确定。
