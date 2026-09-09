---
title: "Develop a Rust Macro for Automating Data Extraction"
date: "2025-07-16 00:00:00 +0800"
lang: "en"
tags: ["Rust", "Macros", "Programming", "AI", "Learning"]
permalink: "/articles/posts/learning-rust-macros.html"
translation_key: "learning-rust-macros"
source_url: "https://techlab.fun/articles/posts/learning-rust-macros.html"
---

Not just procedural macros, but macros in general, has been a difficult topic for me since my first hands-on experience with Rust. I never understood why they needed such complex syntax and abstraction layers. My perspective didn't change until I started trying to improve my crate's ergonomics.

## The Problem: Too Much Boilerplate

I created a crate for easily populating Rust structs by leveraging LLMs. In the past, when I needed to extract data or something structural from an LLM, I would need to define a struct, set up boilerplate code for calling the LLMs, and then write my prompts. This was distracting when coding. Therefore, I made [secretary](https://github.com/AspadaX/secretary).

The end result is amazing. I can now skip all these repetitive steps and simply define a struct like this:

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

/// Example data structure for extracting product information
#[derive(Task, Serialize, Deserialize, Debug)]
struct ProductExtraction {
    /// Product data fields with specific extraction instructions
    #[task(instruction = "Extract the product name or title")]
    pub name: String,

    #[task(instruction = "Extract key features or description")]
    pub description: String,

    #[task(instruction = "Determine if the product is in stock (true/false)")]
    pub in_stock: bool,

    pub details: Details,
}
```

## Understanding Procedural Macros

If you've used `serde` or `clap`, you'll notice the attribute annotations above the struct and fields. In Rust, these are procedural macros. During compilation, these macros are expanded to generate additional code before the main compilation phase, so that the generated implementations can be used during runtime without manually writing all of them. The main purpose of using a macro is to reduce boilerplate code and minimize the chance of repetitive errors.

Rust has two kinds of macros. The first type is declarative macros, created with `macro_rules!`. This is what you usually see when using `vec![]`, `println!()`, or `info!()`. You just declare them and use them in your project. The second type is called procedural macros (sometimes abbreviated as "proc macros"). Procedural macros need to be set up as an independent crate and have syntax that's more "Rusty" than declarative macros. To use a procedural macro, you need to include it in your `Cargo.toml` as if it's a library. Despite the nuances between them, they all do the same thing - manipulate code before compilation.

## Core Concept: Code as Data

Here's the key insight that helped me understand macros: if a function processes data as input and produces processed data as output, a macro processes code as input and produces transformed code as output. 

Let me illustrate this with a practical example. In the following simplified code snippet, we're using an LLM to extract data for us. The raw text is data input, and the result struct is processed data:
```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync + 'static>> {
    // Create a task instance
    let task = ProductExtraction::new();

    // Additional instructions for the LLM
    let additional_instructions = vec![
        "Be precise with numerical values".to_string(),
        "Use 'Unknown' for missing information".to_string(),
        "Ensure boolean values are accurate".to_string(),
    ];

    // Example product description text
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

The `result` variable at the end of the snippet is our processed data. Now, in a macro, we don't process data - we process code. When I mark a struct with derive macros and traits like this:

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

I'm expecting the macro to generate trait implementations for `Task`, `Serialize`, `Deserialize`, and `Debug` specifically for this struct. This process happens before compilation, so when we actually run the code, all four traits' methods will be ready for use. With a macro like this, my crate's users don't need to manually implement the relevant code.

## Learning Approach: Using AI as a Collaborative Tool

But I had to implement the macros, and before that, I needed to learn them first. In the past, I would need to go through lots of examples and documentation to figure out the basics. Now we have LLMs. I used an LLM as my tutor while learning. I first used an LLM to query against my codebase and asked about using a macro approach to generate the code. Then I did some research into the documentation to get a basic idea of macros in Rust.

I found the Rust Book to be an excellent starting point for learning new concepts. This initial research gave me a basic understanding of macros and was sufficient for writing instructions to an LLM to generate a basic macro for me. The first macro code in my crate was largely done by AI, but that only solved basic cases.

In my experience, AI can help kickstart development, but I found it insufficient for handling complex edge cases. After I released the crate and started using it in my projects, I discovered that proper macro design can handle complex scenarios including nested structs and field validation, even when the AI-generated code initially seemed limited.

I found that working with AI as a collaborative tool, rather than relying on it entirely, proved most effective. As I mentioned earlier, I didn't just let the AI do the work - I did research beforehand. It's like hiring someone to do work for me. I can't just hand it over and expect them to do everything and know everything. If I don't have the knowledge, things will derail, and that applies to AI coders too.

## The Second Learning Phase: Diving Deeper

When I started refactoring my crate, I approached learning macros with AI assistance once again. However, this time I found myself much more comfortable with the deeper aspects of macro development. The concepts that once seemed foggy began to crystallize.

Having AI-generated macros that were tightly connected to my specific use cases made it much easier to understand why and how each macro feature was implemented. This hands-on experience with real code was invaluable for building deeper understanding. In a nutshell, a procedural macro takes the marked code and processes it. In the following code snippet from my crate, it takes the marked code (the struct marked with `Task`) as an `input` variable, which is typed as `TokenStream`. A TokenStream is a representation of Rust code as a stream of tokens that can be manipulated programmatically:

```rust
#[proc_macro_derive(Task, attributes(task))]
pub fn derive_task(input: TokenStream) -> TokenStream {
    let input: DeriveInput = parse_macro_input!(input as DeriveInput);
    let name: &syn::Ident = &input.ident;
    let mut expanded: proc_macro2::TokenStream = proc_macro2::TokenStream::new();

    // Extract field information for generating instructions
    let fields: &syn::punctuated::Punctuated<syn::Field, syn::token::Comma> = match &input.data {...};

    // Validate
    if let Err(validation_error) = validate_field_requirements(fields) {
        match validation_error {...}
    }

    // Add `where` clause to fields with Task impl
    let task_field_types: Vec<_> = fields
        .iter()
        .filter(|field| classify_field_type(&field.ty) == FieldCategory::PotentialTask)
        .map(|field| &field.ty)
        .collect();
    let trait_bounds: proc_macro2::TokenStream = if !task_field_types.is_empty() {
        quote! {...}
    } else {
        quote! {}
    };

    // Generate field instructions and expansion logic for normal json generation
    let field_expansions: Vec<proc_macro2::TokenStream> = implement_build_instruction_json(fields);

    // Generate field processing code for distributed generation
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

For simplicity, I removed the implementation details from the code block above, but the actual macro includes extensive validation, error handling, and field classification logic.

## Three Core Concepts of Procedural Macros

Through my experience, I discovered three core concepts that are essential for understanding procedural macros:

**1. TokenStream**
A TokenStream is a representation of Rust code as a stream of tokens that can be manipulated programmatically. A procedural macro takes a TokenStream as input, modifies it, and eventually returns it as a TokenStream back to the compiler. Therefore, everything we do in the macro will eventually become compilable Rust code, no matter how fancy the intermediate process may appear.

**2. The `quote!` Macro**
The `quote!` macro allows you to write Rust code that will be generated as tokens. It converts Rust syntax into a TokenStream. Inside the `quote!` macro, you can write the code that you want to generate. The return value of `quote!` is a TokenStream, and you can combine multiple TokenStreams together, which makes modularization in macros possible.

**3. Syntax Tree Manipulation**
The ability to analyze and manipulate the original code structure using the parsed syntax tree allows you to extract information about fields, types, and attributes. This enables you to use the struct's metadata to implement any methods or traits you want automatically.

## Practical Implementation Examples

Let me show you how these concepts work together in practice. Here's how I implement the Default trait automatically: 
```rust
pub fn implement_default(
    name: &Ident,
    fields: &syn::punctuated::Punctuated<syn::Field, syn::token::Comma>,
) -> TokenStream {
    // Assign default values to each field
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

This example demonstrates several important macro concepts working together:

**Identifiers in Rust Macros**
An `ident` represents an identifier - things like variable names, function names, struct names, etc. Here, `field.ident.as_ref().unwrap()` extracts the name of each field (like `name`, `description`, `in_stock`) from the struct definition.

**Quote Macro Repetition Syntax**
The `#(#field_defaults),*` syntax is a powerful `quote!` macro feature for generating repetitive code. The `#(...)` denotes a repetition block, `#field_defaults` is the variable to repeat over (our Vec of field assignments), and `,*` means "separate each item with a comma, and repeat zero or more times". So if our struct has fields `name`, `description`, and `in_stock`, this pattern expands to:

```rust
Self {
    name: Default::default(),
    description: Default::default(),
    in_stock: Default::default(),
}
```

**Variable Interpolation**
The `#name` represents the name of the struct we marked with `Task`. The `#` symbol interpolates variables defined outside the `quote!` macro into the generated code. This automation means users no longer need to manually implement Default for their structs - the macro handles it automatically! Following this pattern, you can implement any trait or method for the original struct.

## Conclusion

There are many aspects I didn't cover in this post, such as setting up a derive crate for your project. These details are readily available online and through AI tools. What I wanted to share here are the key insights I gained while learning Rust metaprogramming and the journey that led me there.

After my second round of learning, I was able to dig much deeper into the subject. This experience reinforced that hands-on experience with real use cases is crucial for understanding complex concepts like macros. When I encountered obstacles, returning to fundamental documentation always helped me break through to the next level.

The secretary crate is fully open-sourced under the MIT license. I hope this post and the crate will be helpful to you in your own Rust journey. Feel free to leave feedback!
