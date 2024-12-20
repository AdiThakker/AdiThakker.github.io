---
layout:     post
title:      Exploring Sementic Kernel
date:       2025-12-20
summary:    A follow up to one of my previous post on how to leverage Semantic Kernel to automatically generate SQL. 
categories: Gen AI, Semantic Kernel
---

In my [previous blog post]({{site.url}}/VsCode-CoPilot-CodeGeneration), we explored how **GitHub Copilot Chat** can dynamically generate SQL queries based on instructions and data files. This is all well and good, but what if we want to build this functionality into a custom application?  

Today, we’ll build a same **Custom SQL generation application** using **Semantic Kernel** and  **Ollama - Local LLM**.

This approach not only gives you complete control over the LLM but also provides an excellent learning opportunity to understand:
- How Semantic Kernel orchestrates AI workflows.
- How to use local LLMs like **Ollama** for privacy and cost savings.
- Ways to mimic **Visual Studio Code functionality** (e.g., file-based workflows, dynamic prompts) in your custom application.

---

## **What is Semantic Kernel?**

**Semantic Kernel** is an open-source SDK by Microsoft that bridges traditional programming and AI-driven workflows. It enables developers to:
- **Orchestrate AI workflows**: Combine natural language prompts, custom logic, and code execution seamlessly.
- **Leverage Semantic Functions**: Dynamically inject variables into prompts for real-time processing.
- **Extend Capabilities with Plugins**: Create reusable, modular skills for tasks like file handling or API integrations.

### **Core Concepts**
1. **Kernel**:
   - The central component that acts as the orchestrator for LLM interactions.
   - [Learn more about the Kernel](https://learn.microsoft.com/en-us/semantic-kernel/concepts/what-is-the-kernel)

2. **Plugins**:
   - Extend the Kernel's functionality by encapsulating reusable tasks (e.g., file handling or data preprocessing).
   - [Learn more about Plugins](https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins)

3. **Semantic Functions**:
   - Prompts with placeholders (e.g., `{{variable}}`) that can be dynamically rendered with runtime data.
   - [Learn more about Semantic Functions](https://learn.microsoft.com/en-us/semantic-kernel/concepts/prompts/prompt-template-syntax)

4. **Kernel Arguments**:
   - A flexible structure for passing variables into functions or prompts.
   - [Learn more about Kernel Arguments](https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins-and-functions)

---

## **What is Ollama?**

**Ollama** is a lightweight local runtime for hosting large language models (LLMs) on your machine. It allows you to:
- Run LLMs **locally** for enhanced privacy.
- Avoid cloud-based API costs, enabling free usage after setup.
- Use models like **phi3**, optimized for on-premises use cases.

---

## **Getting Started with Ollama on Windows**

### **1. Install Ollama**
1. Download the **Ollama installer** for Windows from the [official website](https://ollama.ai/download).
2. Follow the installation steps to set up Ollama on your machine.

### **2. Install the Model**
Once Ollama is installed, use the following PowerShell command to download and install the **phi3** model:
```powershell
ollama pull phi3