---
title: "用 Rust 开发一个基于 AI 的命令行工具"
date: "2025-07-29 00:00:00 +0800"
lang: "zh-CN"
tags: ["Rust", "CLI", "Programming", "AI", "Learning"]
permalink: "/cn/articles/posts/develop-ai-tool-in-rust.html"
translation_key: "develop-ai-tool-in-rust"
source_url: "https://techlab.fun/cn/articles/posts/develop-ai-tool-in-rust.html"
---

我开发了一个名为 [you](https://github.com/AspadaX/you) 的命令行工具——这是一个可以从自然语言输入生成并执行 shell 脚本的命令行界面。这个工具解决了开发者的一个常见痛点：忘记终端命令并不得不切换窗口去搜索文档。

## 问题：上下文切换和命令回忆

作为一名开发者，我经常在终端工作时忘记特定的命令语法。不断需要切换到其他窗口搜索文档打断了我的工作流程。我需要一个解决方案，能够让我直接在终端中使用自然语言描述来生成和执行 shell 脚本。

## 评估现有解决方案

在决定构建自定义解决方案之前，我调研了市场上现有的 AI 驱动的命令行工具。最主要的选择包括 Warp Terminal、Open Interpreter 和 VSCode Copilot，每个都提供了不同的 AI 辅助命令行交互方法。

### Warp Terminal 分析

Warp 是一个基于 Rust 开发的终端，集成了 AI 功能，提供响应式性能和出色的用户体验。然而，有几个限制让它不符合我的需求：

- 缺乏远程服务器连接管理功能（截至 2024 年初）
- 缺少我在 macOS 默认终端中经常使用的功能
- AI 功能需要订阅，而我只需要基本的命令生成，这可以通过本地开源模型实现

### Open Interpreter 评估

Open Interpreter 提供了一个不依赖特定终端的命令行工具，支持本地模型且无需订阅费用。然而，在长期使用过程中，我发现了几个性能问题：

- 由于通过 pip 命令安装 Python 依赖项导致启动时间缓慢
- 较大的安装包体积，需要下载所有依赖项
- 基本命令生成任务消耗过多 token
- 依赖网络的安装过程可能需要相当长的时间

该工具的功能过于复杂，超出了我对简单 shell 脚本生成和执行的需求。

## 需求定义和自定义解决方案决策

在评估这些现有解决方案后，我意识到没有一个完全满足我的特定需求。基于以上分析，我确定了理想命令行工具的三个核心要求：

1. **小尺寸**：最小的安装占用空间和快速部署
2. **快速响应**：快速启动和执行时间
3. **可移植性**：无需复杂环境配置的简易安装

所需的核心功能很简单：接受自然语言输入，将其发送到 LLM (Large Language Model)，接收 shell 脚本，并在本地执行。基于这一认识，我认为一个针对性的解决方案将更好地满足我的特定需求。

## 技术选择：为什么选择 Rust

明确定义了这些要求后，我需要选择一个能够在所有三个方面都能交付的技术栈。基于几个技术优势，Rust 成为了最佳选择：

- **小二进制文件大小**：Rust 产生轻量级编译二进制文件（此项目约 2.5MB）
- **性能**：快速执行和最小运行时开销
- **生态系统成熟度**：可用于 LLM 集成和命令行开发的强大库
- **部署简单性**：自包含二进制文件减少安装复杂性

## 开发策略：从现有项目中学习

选择 Rust 作为基础后，我需要高效地实现 LLM 集成和命令行解析。我没有从零开始可能需要花费数周时间学习库的复杂性，而是采取了实用的做法，通过研究成功的开源项目并改进它们成熟的方案。

### 命令行解析

我参考了 [uv](https://github.com/astral-sh/uv)，一个 Python 包管理工具，来学习有效的命令行参数解析方法。通过参考并改进他们的解析器实现，我将开发时间从估计的一周减少到仅仅几分钟的调整。通过学习他人的工作，能够如此快速地完成开发确实令人惊讶。

### LLM 集成

我利用了 async-openai GitHub 仓库中提供的完整[示例](https://github.com/64bit/async-openai)。聊天完成示例作为基础，我将其适应用于 shell 脚本生成，再次通过代码重用和修改节省了大量开发时间。

## 技术实现基础

这种以学习为重点的方法让我基于三个关键的 Rust 库构建，它们构成了工具的骨干：

### async-openai
提供与 OpenAI 兼容的 API，支持 Azure OpenAI，实现与各种 LLM 提供商的无缝集成。

### clap
处理命令行参数解析，具有强大的功能支持和清晰的语法。

### tokio
启用异步编程功能，并在需要时为同步操作提供阻塞 API。

## 功能设计理念

### 双重输出：解释和命令

该工具为每个请求生成解释和 shell 脚本，有两个作用：

1. **用户理解**：解释帮助用户理解生成的命令并从中学习
2. **LLM 推理增强**：基于思维链 (Chain-of-Thought, CoT) 原理，提供解释可能改善 LLM 的推理过程

以下是工具工作方式的快速概览：
```bash
(base) xinyubao@Xinyu-MacBook-Air articles % you -r "find the largest file under the current dir"
>> Cache has been enabled.
>> Your input: (y for executing the command, or type to hint LLM)
    > fd -t f . | xargs ls -lS | head -n 1
        * Use fd to find all files, list them by size in descending order, and show the first (largest) one.
```

### 结构化输出实现

我利用 JSON 模式进行 LLM 响应，以确保对输出的程序化控制。这种结构化方法允许程序直接访问每个字段（解释和 shell_script）中的数据，无需额外解析，显著提高了可靠性和可维护性。

## 高级功能和能力

### 多轮对话

除了单命令生成外，该工具还支持跨多轮的上下文对话。这允许用户在一个会话中发出连续命令，LLM 保持对先前交互的感知，以获得更符合上下文的响应。

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

### 命令重用和持久化

认识到 LLM 是概率性的而生成的命令是确定性的，我实现了一个命令保存和重用系统。用户可以保存特别有效的命令并稍后重用它们，将 LLM 生成的创造性与传统确定性程序的可靠性相结合。

## 最终解决方案的优势

完成的基于 Rust 的命令行工具实现了所有原始要求：

- **轻量级**：2.5MB 编译二进制文件，系统依赖最少
- **快速**：快速启动和响应时间，无 Python 环境开销
- **可移植**：使用 rustls 的单二进制安装消除了对外部 TLS 库的需求
- **易于安装**：用户可以运行安装脚本而无需担心环境配置

该工具成功地在自然语言输入和 shell 命令执行之间架起了桥梁，提供了一个专注的解决方案，优先考虑性能和可用性而非功能复杂性。
