# 诊断翻译

> 原文来自《rustc 开发指南》：[Errors and Lints - translation]

rustc 的诊断基础设施使用 [Fluent] 来支持的可翻译的诊断。

[Errors and Lints - translation]: https://rustc-dev-guide.rust-lang.org/diagnostics/translation.html
[Fluent]: https://projectfluent.org

## 编写可翻译的诊断

编写可翻译的诊断信息有两种方式：

1. 对于简单诊断，使用诊断（或子诊断）派生宏
    - “简单”诊断是那些在决定发出子诊断时不需要太多逻辑的诊断，因此可以表示为诊断结构体
    - 参考[诊断和子诊断结构体][diagnostic struct]文档
2. 通过 `DiagnoticBuilder` 接口使用类型化的标识符（实现 `SessionDiagnostic`）

添加或更改可翻译的诊断时，无需担心翻译问题，只需更新原始英文消息即可。

目前，每个定义可翻译诊断的 crate 都有自己的 Fluent 资源，如 `parser.ftl` 或 `typeck.ftl`。

[diagnostic struct]: ./diagnostic-structs.md

## Fluent

Fluent 是建立在“非对称本地化”的思想基础上的，其目的是将翻译的表达力与源语言（rustc 的情况是英语）的语法分离。

在诊断翻译之前，rustc 的诊断在很大程度上依赖于插值来构建显示给用户的消息。

插值字符串很难翻译，因为编写听起来自然的翻译可能需要比英语字符串更多、更少或完全不同的插值，
所有这些都需要更改编译器的源代码才能支持。

诊断消息在 Fluent 资源中被定义。给定区域 (locale)，例如 `en-US`，的一些 Fluent 资源的组合被称为 Fluent bundle。

```fluent
typeck-address-of-temporary-taken = cannot take address of a temporary
```

这个例子中， `typeck-address-of-temporary-taken` 是一条 Fluent 消息的标识，对应的是英文诊断消息。

可以编写用另一种语言编写相应的消息（Fluent 资源）。从而，每个诊断都至少有一条 Fluent 消息。

```fluent
typeck-address-of-temporary-taken = cannot take address of a temporary
    .label = temporary value
```

习惯上，子诊断的诊断消息被指定为 Fluent 消息上的“属性”（额外的相关消息，由 `.<attribute-name>`语法表示）。

上面的例子中，`label` 是与添加到该诊断中的标签 (label) 的消息相对应的 `typeck-address-of-temporary-taken` 的属性。

诊断消息通常会在向用户显示的消息中插入其他上下文，例如类型或变量的名称。Fluent 消息的额外上下文被作为诊断的“参数”提供。

```fluent
typeck-struct-expr-non-exhaustive =
    cannot create non-exhaustive {$what} using struct expression
```

上面的示例中，Fluent 消息引用了一个名为 `what` 的参数，该参数预计会存在（稍后详细讨论如何将参数提供给诊断）。

你可以参考 [Fluent] 文档，了解其他用法示例及其语法。

### 编写可翻译消息的指导原则

为了将消息翻译成不同的语言，任何语言所需的所有信息都必须作为参数提供给诊断，而不仅仅是英文消息中所需的信息。

随着编译器团队获得更多编写诊断的经验，这些诊断具有翻译成不同语言所需的所有信息，此部分会更新。

目前，[Fluent] 文档提供了将消息转换为不同语言环境的极佳示例，以及编写代码所提供的信息。

### 编译时验证和类型化的标识符

[`compiler/rustc_error_messages/locales/en-US`]: https://github.com/rust-lang/rust/tree/master/compiler/rustc_error_messages/locales/en-US 

rustc 默认区域 (`en-us`) 的 Fluent 资源位于 [`compiler/rustc_error_messages/locales/en-US`] 目录下。

目前，每个定义了可翻译诊断的 crate 都有自己的 Fluent 资源，如 `parser.ftl` 或 `typeck.ftl`。

rustc 的 `fluent_messages` 宏执行 Fluent 资源的编译时验证，并生成代码以使其更容易在诊断中引用 Fluent 消息。

编译时对 Fluent 资源的验证将在构建编译器时,从 Fluent 资源发出任何解析错误，从而防止无效的 Fluent 
资源导致编译器 panic。如果多个 Fluent 消息具有相同的标识符，则编译时验证也会发出错误。

在 `rustc_error_messages` 中， `fluent_messages` 还会为每条 Fluent 
消息生成一个常量，用于在发出诊断时引用消息，并保证消息的存在。

```rust
fluent_messages! {
    typeck => "../locales/en-US/typeck.ftl",
}
```

例如，给定以下 Fluent 消息：

```fluent
typeck-field-multiply-specified-in-initializer =
    field `{$ident}` specified more than once
    .label = used more than once
    .label-previous-use = first use of `{$ident}`
```

`fluent_messages` 宏将生成：

```rust
pub static DEFAULT_LOCALE_RESOURCES: &'static [&'static str] = &[
    include_str!("../locales/en-US/typeck.ftl"),
];

mod fluent_generated {
    mod typeck {
        pub const field_multiply_specified_in_initializer: DiagnosticMessage =
            DiagnosticMessage::new("typeck-field-multiply-specified-in-initializer");
        pub const label: SubdiagnosticMessage =
            SubdiagnosticMessage::attr("label");
        pub const label_previous_use: SubdiagnosticMessage =
            SubdiagnosticMessage::attr("previous-use-label");
    }
}
```

`rustc_error_messages::fluent_generated` 被重新导出，并主要作为 `rustc_errors::fluent` 使用。

```rust
use rustc_errors::fluent;
let mut err = sess.struct_span_err(span, fluent::typeck::field_multiply_specified_in_initializer);
err.span_label(span, fluent::typeck::label);
err.span_label(previous_use_span, fluent::typeck::previous_use_label);
err.emit();
```

当发出诊断信息时，可以如上所示使用这些常量。

### 编译器内部

为了支持翻译，对 rustc 的诊断内部结构的各个部分进行了修改。

### 消息

[`rustc_error_messages::DiagnosticMessage`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_error_messages/enum.DiagnosticMessage.html

所有 rustc 的传统诊断接口，如 `struct_span_err` 或 `note`，都会接受任何可以转换为
`DiagnosticMessage` 或 `SubdiagnosticMessage` 的消息。

[`rustc_error_messages::DiagnosticMessage`] 可以表示遗留的不可翻译的诊断消息和可翻译的消息。

不可翻译的消息只是 `String`。

可翻译的消息只是一个带有 Fluent 消息标识的 `&'static str`（有时带有额外的属性的 `&'static str`）。

永远不需要与 `DiagnosticMessage` 直接交互：在 Fluent 资源中为每个诊断消息创建 `DiagnosticMessage`
常量（下面将详细描述），或者在诊断派生宏生成代码中创建 `DiagnosticMessage`。

`rustc_error_messages::SubdiagnosticMessage` 是类似的，它可以对应于遗留的不可翻译的诊断消息，也可以对应于
Fluent 消息的属性名称。

发出可翻译的 `SubdiagnosticMessage` 必须结合 `DiagnosticMessage` （使用
`DiagnosticMessage::with_subdiagnostic_message`）。属性名称没有相应的消息标识是没有意义的，这是由
`DiagnosticMessage` 提供的。

`DiagnosticMessage` 和 `SubdiagnosticMessage` 都为任何可以转换为字符串的类型实现了
`Into`，并将这些类型转换为不可翻译的诊断 —— 这保持所有现有的诊断调用正常运行。

### 参数

插入到消息内容中的 Fluent 消息的额外上下文需要被提供给可翻译的诊断。

你可以使用诊断的 `set_arg` 函数，来给诊断提供这个额外上下文。

参数既有名称（例如前面示例中的 “What”），也有值。

参数值的类型为 `DiagnoticArgValue`，该类型只是一个字符串或一个数字。

rustc 中类型可以实现 `IntoDiagnoticArg`，来转换为字符串或数字，常见的 `Ty<'Tcx>` 等类型已经有这样的实现。

`set_arg` 调用在诊断派生宏内直接处理，但在使用诊断构造器 (diagnostic builder) API 时需要手动添加。

### 加载

rustc 默认使用 `en-us`，并区分了在另一个 locale 缺失消息时备用的 `en-US` 包和使用者要求的首选 Fluent 包。

诊断发射器实现了 `Emitter` 特征，该特征有两个函数用于访问备用和首选包，分别为
`fallback_fluent_bundle` 和 `fluent_bundle`。

`Emitter` 也有成员函数，默认实现使用 `fallback_fluent_bundle` 和 `fluent_bundle`
的结果来执行 `DiagnosticMessage` 的翻译。

rustc 中的所有发射器都惰性地加载备用 Fluent 包，只有在第一次转换错误消息时才读取 Fluent 资源并解析它们。

这是出于性能考虑，如果没有发出错误就读取 Fluent 资源是没有意义的。

`rustc_error_messages::fallback_fluent_bundle` 返回一个 `std::lazy::Lazy<FluentBundle>`，它被提供给发射器，并在第一次调用
`Emitter::fallback_fluent_bundle` 时求值。

`Emitter::fluent_bundle` 会返回用户想要的地区的首选 Fluent 
包。此包在转换消息时优先使用，仅当首选包缺少消息或未提供时才使用备选包。

截至 2022 年 6 月，还没有随编译器一起分发语言区域包 (locale bundles) ，但实现了用于加载包的机制。

* `-Ztranslate-additional-ftl` 可用来加载特定资源作为测试用的首选包
* `-Ztranslate-lang` 可提供语言标识（类似于 `en-US`），并会在 `$sysroot/share/locale/$locale/`
  目录中找到的任何 Fluent 的资源，包括用户提供的 sysroot 和任何 sysroot 候选。

首选包当前不会延迟加载，请求之后，无论是否出现错误，都会在编译开始时加载。

如果可以假设加载包不会失败，则可以延迟加载首选包。

如果请求的 locale 缺失、Fluent 文件残缺或消息在多个资源中重复，则包加载可能会失败。

