---
layout:     post
title:      Semantic Kernel RAG - (Post 1)
date:       2025-03-14
summary:    An extension to one of my previous post on how to build a Retrieval Augmented Generation (RAG) Workflow with Semantic Kernel’s In-Memory Store 
categories: Gen AI, Semantic Kernel, RAG
---

In my [previous blog post]({{site.url}}/Exploring-Semantic-Kernel), we explored how **Semantic Kernel** can be used to generate SQL queries based on instructions and data files. It demonstrated how to leverage **Ollama - Local LLM** to create a custom SQL generation application.

That works well for small or static datasets, but what if your data grows? How do we keep the Large Language Model (LLM) grounded?

That’s where Retrieval Augmented Generation (RAG) comes into play. Instead of cramming all your data into a single prompt, you selectively retrieve only the relevant pieces of information at query time. This approach is more efficient, scales better, and greatly reduces hallucination.

In this post, we’ll implement a basic RAG flow using Semantic Kernel’s built-in in-memory store—no external database required! We’ll:

 - Ingest lines from instructions.txt and data.csv into the kernel’s in-memory store.

 - Embed each line with Ollama (or your chosen model).

 - Search for the most relevant lines on demand.

 - Combine those lines into a final prompt for text generation.

This sets the stage for the next post, where we’ll swap out the in-memory store for a vector database.


## **Hightlights of RAG**

 - Selective Context: Provide only the data the model needs for a specific query, avoiding prompt bloat.

 - Reduced Hallucination: Since the LLM sees only real data, it’s less likely to fabricate.

 - Maintainability: Just add or remove lines in your store as your knowledge base evolves.


## **Starting point: Simple SQL Generation**

Below is the short code snippet we started with in our [previous post](https://github.com/AdiThakker/SemanticKernel.Plugins/blob/main/SemanticKernel.Plugins/Program.cs), using .AddOllamaTextGeneration for text generation:

```csharp

using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.TextGeneration;

var builder = Kernel
                .CreateBuilder()
                .AddOllamaTextGeneration("phi3", new Uri("http://localhost:11434"));

var kernel = builder.Build();

var instructions = await File.ReadAllTextAsync("Plugin/instructions.txt");
var csvData = await File.ReadAllTextAsync("Plugin/data.csv");
var prompt = await File.ReadAllTextAsync("Plugin/prompt.txt");

var sqlPlugin = kernel.CreateFunctionFromPrompt(prompt);


var response = await sqlPlugin.InvokeAsync(kernel, new KernelArguments { ["instructions"] = instructions, ["csvData"] = csvData });

Console.WriteLine("Generated SQL:");
Console.WriteLine(response);

```
Here, you load the entire contents of instructions.txt and data.csv each time. That’s fine for small demos, but it doesn’t scale.

## **Configuring Ollama for Embeddings**

To store and retrieve data in the in-memory store by similarity, you need an embedding model. We leverage Ollama's ***all=minilm*** for this purpose as shown below:

```csharp

var builder = Kernel
                .CreateBuilder()
                .AddOllamaTextEmbeddingGeneration("all-minilm", new Uri("http://localhost:11434"))
                .AddOllamaTextGeneration("phi3", new Uri("http://localhost:11434"));

```

## **Leveraging Semantic Kernel’s In-Memory Store**

Following is the screenshot of the generated SQL:
![Setup]({{site.url}}/images/sk-2.png){:height="500px" width="900px"}


## Final Thoughts
By combining Semantic Kernel and Ollama, we’ve created a flexible, extensible workflow for SQL generation. Whether automating SQL queries, generating API endpoints, or creating infrastructure-as-code scripts, this setup provides an excellent foundation for extending the template for other use cases.

What would you build with Semantic Kernel? Let me know in the comments below! 🚀

Happy coding!