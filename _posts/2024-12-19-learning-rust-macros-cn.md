---
title: "开发一个用于自动化数据提取的 Rust 宏"
date: "2024-12-19 00:00:00 +0800"
lang: "zh-CN"
tags: ["Rust", "Macros", "Programming", "AI", "Learning"]
permalink: "/cn/articles/posts/learning-rust-macros.html"
translation_key: "learning-rust-macros"
source_url: "https://techlab.fun/cn/articles/posts/learning-rust-macros.html"
---

不仅仅是过程宏 (procedural macros)，宏这个概念总体来说，自从我第一次实际接触 Rust 以来，一直是一个困难的话题。我一直不理解为什么它们需要如此复杂的语法和抽象层。直到我开始尝试提升我的 crate 的可用性 (ergonomics)，我的观点才改变。

## 问题：太多样板代码

我创建了一个 crate，利用大型语言模型 (LLMs) 轻松生成 Rust 结构体。过去，当我需要从 LLM 中提取数据或某些结构化内容时，我需要定义一个结构体，设置调用 LLM 的样板代码，然后编写我的提示词。这在编码时很容易分散注意力。因此，我制作了 [secretary](https://github.com/AspadaX/secretary)。

最终结果很惊人。现在我可以跳过所有这些重复步骤，只需简单定义一个结构体，如下所示：

```rust
#[derive(Task, Serialize, Deserialize, Debug)]
struct Details {
    #[task(instruction = "Extract the price as a float")]
    pub price: f64,

    #[task(instruction = "Extract the product category or type")]
    pub category: String,

    #[task(instruction = "Extract the brand name if mentioned")]
    pub brand: Option<String>,
}

/// 用于提取产品信息的示例数据结构
#[derive(Task, Serialize, Deserialize, Debug)]
struct ProductExtraction {
    /// 带有特定提取指令的产品数据字段
    #[task(instruction = "Extract the product name or title")]
    pub name: String,

    #[task(instruction = "Extract key features or description")]
    pub description: String,

    #[task(instruction = "Determine if the product is in stock (true/false)")]
    pub in_stock: bool,

    pub details: Details,
}
```

## 理解过程宏

如果你使用过 `serde` 或 `clap`，你会注意到结构体和字段上面的属性标注。在 Rust 中，这些是过程宏。在编译期间，这些宏在主要编译阶段之前会被扩展，生成额外的代码，以便运行时可以使用生成的实现，而无需手动编写所有这些。主要使用宏的目的是减少样板代码并最小化重复错误的机会。

Rust 有两种宏。第一种是声明宏 (declarative macros)，使用 `macro_rules!` 创建。这就是你在使用 `vec![]`、`println!()` 或 `info!()` 时通常看到的。你只需声明它们并在项目中使用它们。第二种称为过程宏 (procedural macros，有时缩写为 "proc macros")。过程宏需要设置为独立的 crate，并且语法比声明宏更 "Rust 化"。要使用过程宏，你需要像库一样将其包含在你的 `Cargo.toml` 中。尽管它们之间有细微差别，但它们都做同样的事情——在编译前操纵代码。

## 核心概念：代码即数据

在学习宏的时候我意识到：函数处理数据并输出结果，而宏则处理代码并输出转换后的代码。

让我用一个实际示例来说明。在下面的简化代码片段中，我们使用 LLM 为我们提取数据。原始文本是数据输入，结果结构体是处理后的数据：

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync + 'static>> {
    // 创建任务实例
    let task = ProductExtraction::new();

    // LLM 的额外指令
    let additional_instructions = vec![
        "Be precise with numerical values".to_string(),
        "Use 'Unknown' for missing information".to_string(),
        "Ensure boolean values are accurate".to_string(),
    ];

    // 示例产品描述文本
    let product_text = "
        Apple MacBook Pro 16-inch - $2,499
        
        The latest MacBook Pro features the powerful M3 Pro chip, 
        16GB unified memory, and 512GB SSD storage. Perfect for 
        professional video editing and software development.
        
        Category: Laptop Computer
        Status: In Stock
        Brand: Apple
    ";

    let llm = OpenAILLM::new(
        &std::env::var("SECRETARY_OPENAI_API_BASE").unwrap(),
        &std::env::var("SECRETARY_OPENAI_API_KEY").unwrap(),
        &std::env::var("SECRETARY_OPENAI_MODEL").unwrap(),
    )?;

    println!("Making async request to LLM...");
    let result: ProductExtraction = llm
        .async_generate_data(&task, product_text, &additional_instructions)
        .await?;
    println!("Generated Data Structure: {:#?}", result);

    Ok(())
}
```

代码片段末尾的 `result` 变量是我们处理后的数据。现在，在宏中，我们不处理数据——我们处理代码。当我使用派生宏 (derive macros) 和 trait 标记结构体时，如下所示：

```rust
#[derive(Task, Serialize, Deserialize, Debug)]
struct ProductExtraction {
    #[task(instruction = "Extract the product name or title")]
    pub name: String,

    #[task(instruction = "Extract key features or description")]
    pub description: String,

    #[task(instruction = "Determine if the product is in stock (true/false)")]
    pub in_stock: bool,

    pub details: Details,
}
```

我期望宏为这个结构体生成 `Task`、`Serialize`、`Deserialize` 和 `Debug` 的 trait 实现。这个过程发生在编译之前，因此当我们实际运行代码时，所有四个 trait 的方法都将准备好使用。有了这样的宏，我的 crate 用户无需手动实现相关代码。

## 学习方法：使用 AI 作为协作工具

但我必须实现这些宏，而在那之前，我需要先学习它们。过去，我需要浏览大量示例和文档来掌握基础知识。现在有了 LLM。我在学习时使用 LLM 作为我的导师。我首先使用 LLM 查询我的代码库，并询问使用宏方法生成代码。然后我对文档进行了一些研究，以获得 Rust 中宏的基本概念。

我发现 Rust Book 是学习新概念的绝佳起点。这个初步研究给了我宏的基本理解，并足以编写指令给 LLM 为我生成一个基本宏。我的 crate 中的第一个宏代码主要是由 AI 完成的，但那只解决了基本情况。

根据我的经验，AI 可以帮助启动开发，但对于处理复杂边缘情况，我发现它力不从心。在我发布 crate 并开始在我的项目中使用它后，我发现适当的宏设计可以处理复杂场景，包括嵌套结构体和字段验证，即使 AI 生成的代码最初显得有限。

我发现将 AI 作为协作工具，而不是完全依赖它，证明是最有效的。正如我之前提到的，我不仅仅让 AI 做工作——我在事先做了研究。这就像雇佣某人替我做事。我不能只是交给他们让他们干。如果我自己事先不清楚如何实现，事情就会朝着错误的方向发展，这也适用于 AI 生成代码。

## 第二个学习阶段：深入探索

当我开始重构我的 crate 时，我再次在 AI 的辅助下学习宏。然而，这次我发现自己对宏开发的更深层方面要熟悉得多。曾经显得模糊的概念开始变得清晰。

拥有与我的特定用例紧密相关的 AI 生成宏，使得理解每个宏功能的为什么和如何实现变得容易得多。这种与真实代码的实际经验对于建立更深的理解是无价的。简而言之，过程宏获取标记的代码并处理它。在我的 crate 中的以下代码片段中，它将标记的代码（用 `Task` 标记的结构体）作为 `input` 变量，类型为 `TokenStream`。TokenStream 是 Rust 代码作为可编程操作的令牌流的表示：

```rust
#[proc_macro_derive(Task, attributes(task))]
pub fn derive_task(input: TokenStream) -> TokenStream {
    let input: DeriveInput = parse_macro_input!(input as DeriveInput);
    let name: &syn::Ident = &input.ident;
    let mut expanded: proc_macro2::TokenStream = proc_macro2::TokenStream::new();

    // 提取字段信息以生成指令
    let fields: &syn::punctuated::Punctuated<syn::Field, syn::token::Comma> = match &input.data {...};

    // 验证
    if let Err(validation_error) = validate_field_requirements(fields) {
        match validation_error {...}
    }

    // 为具有 Task 实现的字段添加 `where` 子句
    let task_field_types: Vec<_> = fields
        .iter()
        .filter(|field| classify_field_type(&field.ty) == FieldCategory::PotentialTask)
        .map(|field| &field.ty)
        .collect();
    let trait_bounds: proc_macro2::TokenStream = if !task_field_types.is_empty() {
        quote! {...}
    } else {
        quote! {};
    };

    // 为正常 JSON 生成生成字段指令和扩展逻辑
    let field_expansions: Vec<proc_macro2::TokenStream> = implement_build_instruction_json(fields);

    // 为分布式生成生成字段处理代码
    let distributed_field_processing: Vec<proc_macro2::TokenStream> =
        implement_field_processing_code(fields);

    expanded.extend(implement_default(&name, &fields));
    expanded.extend(implement_task(
        &name,
        &trait_bounds,
        &distributed_field_processing,
    ));
    expanded.extend(quote! {...});

    TokenStream::from(expanded)
}
```

为了简单起见，我从上面的代码块中删除了实现细节，但实际宏包括广泛的验证、错误处理和字段分类逻辑。

## 过程宏的三个核心概念

通过我的经验，我发现了理解过程宏的三个核心概念：

**1. TokenStream**
TokenStream 是 Rust 代码作为可编程操作的令牌流的表示。过程宏将 TokenStream 作为输入，修改它，并最终将其作为 TokenStream 返回给编译器。因此，我们在宏中做的所有事情最终都会成为可编译的 Rust 代码，无论中间过程看起来多么花哨。

**2. `quote!` 宏**
`quote!` 宏允许你编写将作为令牌生成的 Rust 代码。它将 Rust 语法转换为 TokenStream。在 `quote!` 宏内部，你可以编写你想要生成的代码。`quote!` 的返回值是一个 TokenStream，你可以组合多个 TokenStream，这使得宏中的模块化成为可能。

**3. 语法树操作**
使用解析的语法树分析和操作原始代码结构的能力允许你提取关于字段、类型和属性的信息。这使你能够使用结构体的元数据自动实现你想要的任何方法或 trait。

## 实际实现示例

让我向你展示这些概念如何协同工作。以下是我如何自动实现 Default trait：

```rust
pub fn implement_default(
    name: &Ident,
    fields: &syn::punctuated::Punctuated<syn::Field, syn::token::Comma>,
) -> TokenStream {
    // 为每个字段分配默认值
    let field_defaults: Vec<_> = fields
        .iter()
        .map(|field| {
            let field_name: &syn::Ident = field.ident.as_ref().unwrap();
            quote! {
                #field_name: Default::default()
            }
        })
        .collect();

    quote! {
        impl Default for #name {
            fn default() -> Self {
                Self {
                    #(#field_defaults),*
                }
            }
        }
    }
}
```

这个示例展示了几个重要的宏概念协同工作：

**Rust 宏中的标识符**
`ident` 表示标识符——如变量名、函数名、结构体名等。这里，`field.ident.as_ref().unwrap()` 从结构体定义中提取每个字段的名称（如 `name`、`description`、`in_stock`）。

**Quote 宏重复语法**
`#(#field_defaults),*` 语法是 `quote!` 宏的一个强大功能，用于生成重复代码。`#(...)` 表示重复块，`#field_defaults` 是要重复的变量（我们的字段分配 Vec），` ,*` 表示“用逗号分隔每个项，并重复零次或多次”。因此，如果我们的结构体有字段 `name`、`description` 和 `in_stock`，这个模式会扩展为：

```rust
Self {
    name: Default::default(),
    description: Default::default(),
    in_stock: Default::default(),
}
```

**变量插值**
`#name` 表示用 `Task` 标记的结构体的名称。`#` 符号将 `quote!` 宏外部定义的变量插值到生成的代码中。这种自动化意味着用户不再需要手动为他们的结构体实现 Default——宏会自动处理！按照这个模式，你可以为原始结构体实现任何 trait 或方法。

## 结论

在我的第二轮学习之后，我能够更深入地研究这个主题。这种经验强化了我的认识：实际经验与真实用例对于理解像宏这样的复杂概念至关重要。当我遇到障碍时，回到基础文档总是帮助我突破到下一个水平。

secretary crate 在 MIT 许可下完全开源。我希望这篇文章和 crate 在你的 Rust 之旅中对你有帮助。请随时留下反馈！
