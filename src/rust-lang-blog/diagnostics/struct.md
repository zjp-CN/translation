# 诊断和子诊断结构体

> 原文来自《Rustc 开发指南》：[Diagnostic and subdiagnostic structs]

[Diagnostic and subdiagnostic structs]: https://rustc-dev-guide.rust-lang.org/diagnostics/diagnostic-structs.html

rustc 有两个诊断派生宏 (diagnostic derives)，可用于创建简单的诊断，建议在合适的地方使用：
`#[derive(SessionDiagnostic)]` 和 `#[derive(SessionSubdiagnostic)]`。

使用派生宏创建的诊断可以翻译成不同的语言，并且每种语言都有一个唯一标识诊断的开头 (slug)。

## `SessionDiagnostic`

除了使用 `DiagnoticBuilder` 接口创建和发出诊断外，还可以使用 `SessionDiagnotic` 派生宏。

`#[derive(SessionDiagnotics)]` 只适用于不需要太多逻辑来决定是否添加附加子诊断的简单诊断。

### 示例

考虑如下所示的 `field already declared` 的[定义][field already declared]：

[field already declared]: https://github.com/rust-lang/rust/blob/bbe9d27b8ff36da56638aa43d6d0cdfdf89a4e57/compiler/rustc_typeck/src/errors.rs#L65-L74

```rust,ignore
#[derive(SessionDiagnostic)]
#[error(typeck::field_already_declared, code = "E0124")]
pub struct FieldAlreadyDeclared {
    pub field_name: Ident,
    #[primary_span]
    #[label]
    pub span: Span,
    #[label = "previous-decl-label"]
    pub prev_span: Span,
}
```

`SessionDiagnostic` 只能应用于结构体。每个 `SessionDiagnostic` 都必须有一个属性应用于结构本身：
- 要么使用 `#[error(...)]` 定义错误
- 要么使用 `#[warning(...)]` 定义警告

如果错误具有错误代码（例如“E0624”），则可以使用 `code` 子属性指定。

指定 `code` 不是必需的，但如果你要移植的诊断使用 `DiagnoticBuilder`，那么如果有错误码的话，使用
`SessionDiagnostic` 时应保留错误码。

`#[error(...)]` 和 `#[warning(...)]` 都必须提供一个开头 (slug) 作为第一个位置参数（即一个通向
`rustc_errors::fluent::*` 中的条目的路径）。

slug 唯一地标识了诊断，也是编译器知道要发出什么错误消息的方式（在编译器的默认 locale 中，或者在用户要求的 
locale）。有关如何编写可翻译的错误消息以及如何生成 slug 条目的详细信息，请参阅 [translation]。

[translation]: ./translation.md

在我们的示例中，`field already declared` 诊断的 Fluent 消息如下所示：

```fluent
typeck-field-already-declared =
    field `{$field_name}` is already declared
    .label = field already declared
    .previous-decl-label = `{$field_name}` first declared here
```

`typeck-field-already-declared` 是示例中的 slug，后面跟着诊断消息。

没有标注属性的 `SessionDiagnostic` 的每个字段都可以作为 Fluent 消息中可用的变量，例子中的
`field_name`。如果不需要，可以将字段标为 `#[skip_arg]`。

在类型为 `Span` 的字段上使用 `#[primary_span]` 属性表示诊断的主要范围，该范围将包含诊断的主要消息。

诊断不仅仅有它们的主要消息，通常还包括标签 (labels)、注意 (notes)、帮助 (help) 消息和建议
(suggestions)，所有这些也可以在 `SessionDiagnotic` 中指定。

`#[label]`、`#[help]`、`#[note]` 都可以应用于 `Span` 类型的字段。

应用这些属性中的任何一个都将创建该 `Span` 相应的子诊断。这些属性会在主 Fluent 消息相关的 Fluent
属性中查找它们的诊断消息。示例中的 `#[label]` 将查找 `typeck-field-already-declared.label`
（其消息为 “fieldAlternated”）。

如果有多个相同类型的子诊断，则这些属性还可以接收要查找的属性名称（例如，示例中的 `prese-decl-label`）。

以下类型在使用 `SessionDiagnotic` 派生宏时有特殊行为：

* 所有应用于 `Option<T>` 的属性，仅当值为 `Some(...)` 时才会发出子诊断
* 所有应用于 `Vec<T>` 的属性，将针对其每个元素重复

`#[help]` 和 `#[note]` 也可以应用于结构体本身，此时，它们的工作方式与应用于字段时完全相同，只是子诊断不会有 
`Span`。它们应用于 `()` 类型的字段也是一样的效果：当与 `Option` 类型组合时，表示可选的 `#[note]`/`#[help]` 子诊断。

可使用以下四个字段属性之一发出建议：

* `#[suggestion(message = "...", code = "...", applicability = "...")]`
* `#[suggestion_hidden(message = "...", code = "...", applicability = "...")]`
* `#[suggestion_short(message = "...", code = "...", applicability = "...")]`
* `#[suggestion_verbose(message = "...", code = "...", applicability = "...")]`

`#[suggeston*]` 必须应用于 `Span` 字段或 `(Span, MachineApplicability)` 字段。与其他字段属性类似：
* `message` 在消息中指定了 Fluent 属性，默认为 `.suggestion`
* `code` 指定了建议替换的代码，并且是一个格式字符串，例如 `{field_name}` 将被结构体的 `field_name` 字段的值替换，而不是 Fluent 的标识符
* `applicability` 指定了属性中的适用性，当字段的类型包含 `Applicablity` 时不能使用

最后， `SessionDiagnotic` 派生宏将生成一个 `SessionDiagnotic` 的实现，如下所示：

```rust,ignore
impl SessionDiagnostic for FieldAlreadyDeclared {
    fn into_diagnostic(self, sess: &'_ rustc_session::Session) -> DiagnosticBuilder<'_> {
        let mut diag = sess.struct_err_with_code(
            rustc_errors::DiagnosticMessage::fluent("typeck-field-already-declared"),
            rustc_errors::DiagnosticId::Error("E0124")
        );
        diag.set_span(self.span);
        diag.span_label(
            self.span,
            rustc_errors::DiagnosticMessage::fluent_attr("typeck-field-already-declared", "label")
        );
        diag.span_label(
            self.prev_span,
            rustc_errors::DiagnosticMessage::fluent_attr("typeck-field-already-declared", "previous-decl-label")
        );
        diag
    }
}
```

[use it]: https://github.com/rust-lang/rust/blob/eb82facb1626166188d49599a3313fc95201f556/compiler/rustc_typeck/src/collect.rs#L981-L985

我们已经定义了自己的诊断，那如何[使用][use it]它呢？非常简单，只需创建结构体的一个实例并将其传递给 `emit_err`（或 `emit_warning`）：

```rust,ignore
tcx.sess.emit_err(FieldAlreadyDeclared {
    field_name: f.ident,
    span: f.span,
    prev_span,
});
```

### 属性

`#[derive(SessionDiagnostic)]` 支持以下属性：

#### `#[error | warning]`

`#[error(slug, code = "...")]` 或 `#[warning(slug, code = "...")]`：
* 应用于结构体
* 强制需要的
* 定义了结构体表示错误或警告
* Slug：强制需要
  * 唯一标识诊断，并与其 Fluent 消息相对应
  * 通向 `rustc_errors::fluent` 中的条目的路径
    * 在模块中总是以 Fluent 的资源名称开头，通常是诊断所在 crate 的名字
    * 例如 `rustc_errors::fluent::typeck::field_already_declared`，其中 `rustc_errors::fluent`
      在属性中隐含，所以只需要 `typeck::field_already_declared`
  * 见 [translation]
* `code = "..."`：可选的
  * 指定错误码

#### `#[note]`

`#[note]` 或 `#[note = "..."]` （可选的）
  * 应用于结构体或 `Span`/`()` 字段
  * 增加一个用于注意的子诊断
  * 值是 note 消息的 Fluent 属性（相对于被 `slug` 指定的 Fluent 消息来说）
    * 默认是 Fluent 的 `note` 属性
  * 如果应用到 `Span` 字段，则创建一个 spanned note

#### `#[help]`

`#[help]` 或 `#[help = "..."]` （可选的）
  * 应用于结构体或 `Span`/`()` 字段
  * 增加一个提供帮助信息的子诊断
  * 值是 help 消息的 Fluent 属性（相对于被 `slug` 指定的 Fluent 消息来说）
    * 默认是 Fluent 的 `help` 属性
  * 如果应用到 `Span` 字段，则创建一个 spanned help

#### `#[label]`

`#[label]` 或 `#[label = "..."]` （可选的）
  * 应用于 `Span` 字段
  * 增加一个标签子诊断
  * 值是 label 消息的 Fluent 属性（相对于被 `slug` 指定的 Fluent 消息来说）
    * 默认是 Fluent 的 `label` 属性

#### `#[suggestion*]`

`#[suggestion{,_hidden,_short,_verbose}(message = "...", code = "...", applicability = "...")]` （可选的）
  * 应用于 `(Span, MachineApplicability)` 或 `Span` 字段
  * 增加一个建议子诊断
  * `message = "..."` （强制需要）
    * 值是 suggestion 消息的 Fluent 属性（相对于被 `slug` 指定的 Fluent 消息来说）
    * 默认是 Fluent 的 `suggestion` 属性
  * `code = "..."` （强制需要）
    * 值是建议替换的代码的格式字符串
  * `applicability = "..."` （可选的）
    * 必须是 `machine-applicable`、 `maybe-incorrect`、`has-placeholders` 或 `unspecified` 中之一

#### `#[subdiagnostic]`

`#[subdiagnostic]`
  * 应用于实现了 `AddToDiagnostic` 的类型 （从 `#[derive(SessionSubdiagnostic)]`）
  * 增加一个子诊断结构体
* `#[primary_span]` （可选的）
  * 应用于 `Span` 字段
  * 表示诊断的主要范围 (primary span)
* `#[skip_arg]` （可选的）
  * 应用于任何字段
  * 防止该字段作为诊断参数提供

## `SessionSubdiagnostic`

在编译器中，常见的是编写一个函数，在该函数适用的情况下，有条件地将特定的子诊断添加到错误中。

通常，这些子诊断可以使用诊断结构体来表示，即使整个诊断不能使用结构体。

此时，可以使用 `SessionSubdiagnostic` 派生宏来将部分诊断（例如，注意、标签、帮助或建议）表示为结构体。

### 示例

考虑如下所示的“expected return type”标签的[定义][expected return type]：

[expected return type]: https://github.com/rust-lang/rust/blob/e70c60d34b9783a2fd3171d88d248c2e0ec8ecdd/compiler/rustc_typeck/src/errors.rs#L220-L233

```rust,ignore
#[derive(SessionSubdiagnostic)]
pub enum ExpectedReturnTypeLabel<'tcx> {
    #[label(typeck::expected_default_return_type)]
    Unit {
        #[primary_span]
        span: Span,
    },
    #[label(typeck::expected_return_type)]
    Other {
        #[primary_span]
        span: Span,
        expected: Ty<'tcx>,
    },
}
```

与 `SessionDiagnotic` 不同，`SessionSubdiagnostic` 可以应用于结构体或枚举体。

放置在结构类型上的属性都可以放置在枚举体成员上（反之亦然）。

`SessionSubdiagnostic` 应该有一个属性应用于结构体或者应用于枚举的每个成员上，这些属性有：

* `#[label(...)]` 定义一个标签
* `#[note(...)]` 定义一个注意
* `#[help(...)]` 定义一个帮助
* `#[suggestion{,_hidden,_short,_verbose}(...)]` 定义一个建议

以上所有内容都必须提供一个 slug 作为第一个位置参数（`rustc_error::fluent::*` 中的条目的路径）。

slug 唯一地标识了诊断，也是编译器知道要发出什么错误消息的方式（在编译器的默认 locale 中，或者在用户要求的 
locale）。有关如何编写可翻译的错误消息以及如何生成 slug 条目的详细信息，请参阅 [translation]。

举例来说，`expected return type` 标签的 Fluent 消息如下所示：

```fluent
typeck-expected-default-return-type = expected `()` because of default return type

typeck-expected-return-type = expected `{$expected}` because of return type
```

在（类型为 `Span` 的）字段上使用 `#[primary_span]` 属性表示子诊断的主要跨度。

只有标签或建议才需要主跨度，因为它们不能缺少跨度。

所有没有属性标注的类型/成员在 Fluent 消息中都可用作变量使用。如果不需要，可以将字段标注为 `#[skip_arg]`。

与 `SessionDiagnostic` 一样，`SessionSubdiagnostic` 也支持 `Option<T>` 和 `Vec<T>` 字段。

对类型/成员使用以下一种属性发出建议：

* `#[suggestion(message = "...", code = "...", applicability = "...")]`
* `#[suggestion_hidden(message = "...", code = "...", applicability = "...")]`
* `#[suggestion_short(message = "...", code = "...", applicability = "...")]`
* `#[suggestion_verbose(message = "...", code = "...", applicability = "...")]`

设置建议时，需要在一个字段上设置 `#[primary_span]`；而建议可以有以下子属性：

* `message` 指定消息的 Fluent 属性，默认为`.suggeston`
* `code` 指定建议替换的代码，是一个格式字符串（例如，`{field_name}` 会被替换为结构体的 `field_name` 字段的值），而不是一个 Fluent 的标识符
* `applaiblity` 用来指定属性中的适用性，当字段的类型包含 `Applicablity` 时不能使用它。

适用性也可以通过 `#[applicability]` 属性指定为一个 `Applicablity` 类型的字段。

最后， `SessionSubdiagnostic` 派生宏将生成一个 `SessionSubdiagnostic` 的实现，如下所示：

```rust,ignore
impl<'tcx> AddToDiagnostic for ExpectedReturnTypeLabel<'tcx> {
    fn add_to_diagnostic(self, diag: &mut rustc_errors::Diagnostic) {
        use rustc_errors::{Applicability, IntoDiagnosticArg};
        match self {
            ExpectedReturnTypeLabel::Unit { span } => {
                diag.span_label(span, DiagnosticMessage::fluent("typeck-expected-default-return-type"))
            }
            ExpectedReturnTypeLabel::Other { span, expected } => {
                diag.set_arg("expected", expected);
                diag.span_label(span, DiagnosticMessage::fluent("typeck-expected-return-type"))
            }

        }
    }
}
```

一旦定义了子诊断，就可以通过将其传递给诊断上的 `subdiagnostic` 函数（ [示例 1] 和 [示例 2]
），或通过将其赋给诊断结构体标注了 `#[subdiagnostic]` 的字段这两种方式使用子诊断。

[示例 1]: https://github.com/rust-lang/rust/blob/e70c60d34b9783a2fd3171d88d248c2e0ec8ecdd/compiler/rustc_typeck/src/check/fn_ctxt/suggestions.rs#L556-L560
[示例 2]: https://github.com/rust-lang/rust/blob/e70c60d34b9783a2fd3171d88d248c2e0ec8ecdd/compiler/rustc_typeck/src/check/fn_ctxt/suggestions.rs#L575-L579

### 属性

`#[derive(SessionSubdiagnostic)]` 支持以下属性：

#### `#[label | help | note]`

`#[label(slug)]`、 `#[help(slug)]` 或 `#[note(slug)]`：
* 应用于结构体或枚举体成员；结构体/枚举体成员属性互斥
* 强制需要的
* 定义一个表示标签/帮助/注意的类型
* slug （强制需要的）
  * 唯一标识诊断，并与其 Fluent 消息相对应
  * 通向 `rustc_errors::fluent` 中的条目的路径
    * 在模块中总是以 Fluent 的资源名称开头，通常是诊断所在 crate 的名字
    * 例如 `rustc_errors::fluent::typeck::field_already_declared`，其中 `rustc_errors::fluent`
      在属性中隐含，所以只需要 `typeck::field_already_declared`

#### `#[suggeston]`

`#[suggestion{,_hidden,_short,_verbose}(message = "...", code = "...", applicability = "...")]`：
* 应用于结构体或枚举体成员；结构体/枚举体成员属性互斥
* 强制需要的
* 定义一个表示建议的类型
* `message = "..."` （强制需要的）
  * 值是 suggestion 消息的 Fluent 属性（相对于被 `slug` 指定的 Fluent 消息来说）
  * 默认是 Fluent 的 `suggestion` 属性
* `code = "..."` （强制需要的）
  * 值是建议替换的代码的格式字符串
* `applicability = "..."` （可选的）
  * 与字段上的 `#[applicability]` 互斥
  * 值是建议的适用性
  * 必须是以下一种字符串：
    * `machine-applicable`
    * `maybe-incorrect`
    * `has-placeholders`
    * `unspecified`

#### `#[primary_span]`

`#[primary_span]` 
* 对 label 和 suggeston 是强制需要的，对其他是可选的
* 应用于 `Span` 字段
* 表明子诊断的主要跨度

#### `#[applicability]`

`#[applicability]`
* 可选的
* 只应用于 suggeston
* 表明建议的适用性

#### `#[skip_arg]`

`#[skip_arg]`
* 可选的
* 应用于任何字段
* 防止该字段作为诊断参数提供
