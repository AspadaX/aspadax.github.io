---
title: "Develop an AI-Based CLI Tool in Rust"
date: "2025-07-29 00:00:00 +0800"
lang: "en"
tags: ["Rust", "CLI", "Programming", "AI", "Learning"]
permalink: "/articles/posts/develop-ai-tool-in-rust.html"
translation_key: "develop-ai-tool-in-rust"
source_url: "https://techlab.fun/articles/posts/develop-ai-tool-in-rust.html"
---

I developed a CLI tool called [you](https://github.com/AspadaX/you) — a command-line interface that generates and executes shell scripts from natural language input. This tool addresses a common developer frustration: forgetting terminal commands and having to switch windows to search documentation.

## The Problem: Context Switching and Command Recall

As a developer, I frequently found myself forgetting specific command syntax while working in the terminal. The constant need to switch windows and search through documentation disrupted my workflow and reduced productivity. I needed a solution that would allow me to generate and execute shell scripts directly from my terminal using natural language descriptions.

## Evaluating Existing Solutions

Before committing to building a custom solution, I thoroughly researched existing AI-powered CLI tools in the market. The most prominent options included Warp Terminal, Open Interpreter, and VSCode Copilot, each offering different approaches to AI-assisted command line interaction.

### Warp Terminal Analysis

Warp is a Rust-based terminal with integrated AI capabilities that provides responsive performance and an engaging user experience. However, several limitations made it unsuitable for my needs:

- Lack of remote server bookmarking support (as of early 2024)
- Missing features that I regularly used in the macOS default terminal
- Subscription requirement for AI features, while I only needed basic command generation that could be achieved with local open-source models

### Open Interpreter Evaluation

Open Interpreter offers a CLI tool that works independently of terminal choice and supports local models without subscription costs. However, after extended use, several performance issues became apparent:

- Slow startup time due to Python dependency installation via pip commands
- Large installation footprint requiring all dependencies to be downloaded
- High token consumption for basic command generation tasks
- Network-dependent installation process that could take considerable time

The tool's extensive feature set exceeded my requirements for simple shell script generation and execution.

## Requirements Definition and Custom Solution Decision

After evaluating these existing solutions, I realized that none fully met my specific needs. Based on this analysis, I identified three core requirements for an ideal CLI tool:

1. **Small size**: Minimal installation footprint and fast deployment
2. **Fast response**: Quick startup and execution times
3. **Portability**: Easy installation without complex environment configuration

The core functionality needed was straightforward: accept natural language input, send it to an LLM, receive a shell script, and execute it locally. This realization led me to conclude that a custom, focused solution would better serve my specific needs.

## Technology Selection: Why Rust

With these requirements clearly defined, I needed to select a technology stack that could deliver on all three fronts. Rust emerged as the optimal choice based on several technical advantages:

- **Small binary size**: Rust produces lightweight compiled binaries (approximately 2.5MB for this project)
- **Performance**: Fast execution and minimal runtime overhead
- **Ecosystem maturity**: Robust libraries available for LLM integration and CLI development
- **Deployment simplicity**: Self-contained binaries reduce installation complexity

## Development Strategy: Learning from Existing Projects

With Rust selected as the foundation, I faced the challenge of implementing LLM integration and CLI parsing efficiently. Rather than starting from scratch and potentially spending weeks learning library intricacies, I adopted a pragmatic approach by studying successful open-source projects and adapting their proven patterns.

### Command Line Parsing

I referenced [uv](https://github.com/astral-sh/uv), a Python package management tool, to understand effective CLI argument parsing patterns. By copying and adapting their parser implementation, I reduced development time from an estimated a week to just a few minutes of adjustments. It is quiet amazing to realize how fast it can be when I do my works based on others' works and learn stuff from others. 

### LLM Integration

I utilized the complete [examples](https://github.com/64bit/async-openai) provided in the async-openai GitHub repository. The chat completion examples served as a foundation that I adapted for shell script generation, again saving significant development time through code reuse and modification.

## Technical Implementation Foundation

This learning-focused approach led me to build upon three key Rust libraries that form the backbone of the tool:

### async-openai
Provides OpenAI-compatible APIs with Azure OpenAI support, enabling seamless integration with various LLM providers.

### clap
Handles command-line argument parsing with robust feature support and clear syntax.

### tokio
Enables asynchronous programming capabilities and provides blocking APIs for synchronous operations when needed.

## Feature Design Philosophy

### Dual Output: Explanation and Commands

The tool generates both explanations and shell scripts for each request, serving two purposes:

1. **User Understanding**: Explanations help users comprehend the generated commands and learn from them
2. **LLM Reasoning Enhancement**: Based on Chain-of-Draft (CoD) principles, providing explanations may improve the LLM's reasoning process 

Here is a quick overview of how the tool works like:
```bash
(base) xinyubao@Xinyu-MacBook-Air articles % you -r "find the largest file under the current dir"
>> Cache has been enabled.
>> Your input: (y for executing the command, or type to hint LLM)
    > fd -t f . | xargs ls -lS | head -n 1
        * Use fd to find all files, list them by size in descending order, and show the first (largest) one.
```

### Structured Output Implementation

I made use of JSON mode for LLM responses to ensure programmatic control over outputs. This structured approach allows the program to directly access data in each field (explanation and shell_script) without requiring additional parsing, significantly improving reliability and maintainability.

## Advanced Features and Capabilities

### Multi-Round Conversations

Beyond single-command generation, the tool supports contextual conversations across multiple rounds. This allows users to issue sequential commands within one session, with the LLM maintaining awareness of previous interactions for more contextually appropriate responses.

```bash
(base) xinyubao@Xinyu-MacBook-Air articles % you -r
>> Yes, boss. What can I do for you: find the largest file under the current dir
>> Your input: (y for executing the command, or type to hint LLM)
    > fd -t f . | xargs ls -lS | head -n 1
        * Use fd to find all files, list them by size in descending order, and show the largest one.
 y
>> Start executing command: fd -t f . | xargs ls -lS | head -n 1
    -rw-r--r--  1 xinyubao  staff  7312 Jul 29 13:30 post.md
>> Finished executing command: fd -t f . | xargs ls -lS | head -n 1
>> Commands had been executed successfully.
>> Boss, what else can I do for you (type to instruct, e to exit, or enter w to save the commands so far): search in the file for `async-openai`
>> Your input: (y for executing the command, or type to hint LLM)
    > grep -n "async-openai" post.md
        * Search for the string 'async-openai' in the file post.md and show matching lines with line numbers.
 y
>> Start executing command: grep -n "async-openai" post.md
    61:I utilized the complete [examples](https://github.com/64bit/async-openai) provided in the async-openai GitHub repository. The chat completion examples served as a foundation that I adapted for shell script generation, again saving significant development time through code reuse and modification.
    67:### async-openai
>> Finished executing command: grep -n "async-openai" post.md
>> Commands had been executed successfully.
>> Boss, what else can I do for you (type to instruct, e to exit, or enter w to save the commands so far): 
```

### Command Reuse and Persistence

Recognizing that LLMs are probabilistic while generated commands are deterministic, I implemented a command saving and reuse system. Users can save particularly effective commands and reuse them later, combining the creativity of LLM generation with the reliability of traditional deterministic programs.

## Final Solution Benefits

The completed Rust-based CLI tool achieves all original requirements:

- **Lightweight**: 2.5MB compiled binary with minimal system dependencies
- **Fast**: Quick startup and response times without Python environment overhead
- **Portable**: Single binary installation using rustls eliminates the need for external TLS libraries
- **Easy Installation**: Users can run the installation script without worrying about environment configuration

The tool successfully bridges the gap between natural language input and shell command execution, providing a focused solution that prioritizes performance and usability over feature complexity.
