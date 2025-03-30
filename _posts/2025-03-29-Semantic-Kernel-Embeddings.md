---
layout:     post
title:      Semantic Kernel RAG - (Post 1)
date:       2025-03-29
summary:    An extension to one of my previous post on how to build a Retrieval Augmented Generation (RAG) Workflow with Semantic Kernel’s In-Memory Store 
categories: Gen AI, Semantic Kernel, RAG
---

In my [previous blog post]({{site.url}}/Exploring-Semantic-Kernel), we explored how **Semantic Kernel** can be used to generate SQL queries based on instructions and data files. It demonstrated how to leverage **Ollama - Local LLM** to create a custom SQL generation application.

That works well for small or static datasets, but what if your data grows? How do we keep the Large Language Model (LLM) grounded?

That’s where Retrieval Augmented Generation (RAG) comes into play. Instead of cramming all your data into a single prompt, you selectively retrieve only the relevant pieces of information at query time. This approach is more efficient, scales better, and greatly reduces hallucination.

In this post, we’ll implement a basic RAG flow using Semantic Kernel’s built-in in-memory store—no external database required! We’ll:

 - Ingest lines from data.csv into the kernel’s in-memory store.

 - Embed each line with Ollama (or your chosen model).

 - Search for the most relevant lines on demand.

 - Combine those lines into a final prompt for text generation.


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
In the above snippet, you load the entire contents of instructions.txt and data.csv each time. That’s fine for small demos, but it doesn’t scale. Enter RAG!

## **Adding an In-Memory Store & Embeddings**

To implement RAG, we need to store our data in memory and retrieve only the relevant lines at query time. You also need to embed each line to later compare it with the user’s query by similarity. This is where we will leverage Semantic Kernel’s built-in in-memory store and Ollama's ***all=minilm*** embedding model as shown below:

```csharp

var builder = Kernel
                .CreateBuilder()
                .AddOllamaTextEmbeddingGeneration("all-minilm", new Uri("http://localhost:11434"))
                .AddOllamaTextGeneration("phi3", new Uri("http://localhost:11434"))
                .AddInMemoryVectorStore();
```
We then load our data.csv file into the in-memory store and embed each line with Ollama. Here’s how you can do that:

```csharp

var kernel = builder.Build();
var memory = kernel.GetRequiredService<InMemoryVectorStore>();
var embedding = kernel.GetRequiredService<ITextEmbeddingGenerationService>();

var data = await File.ReadAllTextAsync("Plugin/data.csv");

var lines = data.Split('\n', StringSplitOptions.RemoveEmptyEntries);
var collection = memory.GetCollection<string, Data<string>>("Data");
await collection.CreateCollectionIfNotExistsAsync();

int idx = 0;
foreach (var line in lines)
{
    await collection.UpsertAsync(new Data<string>
    {
        Category = "data",
        Key = $"{idx++}",
        Text = line,
        TextEmbedding = await embedding.GenerateEmbeddingAsync(line)
    });
}    
```

In the above code snippet, we first create a collection in the in-memory store and then iterate over each line in data.csv. For each line, we generate an embedding and store it in the collection, using the **embedding.GenerateEmbeddingAsync** call. We leverage  the custom **Data** class to store the line and its embedding.

```csharp

internal sealed class Data<TKey>
{
    [VectorStoreRecordKey]
    public required TKey Key { get; set; }

    [VectorStoreRecordData]
    public required string Category { get; set; }

    [VectorStoreRecordData]
    public required string Text { get; set; }

    [VectorStoreRecordVector(1536)]
    public ReadOnlyMemory<float> TextEmbedding { get; set; }
}

```


## **Searching for Relevant Lines**

Now that we have our data stored and embedded, we can search for the most relevant lines at query time.  We will first embed the user’s query and then search for the closest matches in our in-memory store. Here’s how you can do that:

```csharp

var query = "Generate a query for the Sales workspace";
var queryEmbedding = await embedding.GenerateEmbeddingAsync(query);

var search = await collection.VectorizedSearchAsync(queryEmbedding, new VectorSearchOptions { Top = 1 });
var results = await search.Results.AsAsyncEnumerable().ToListAsync();
var csvData = results?.First()?.Record?.Text;

```

We then build a final prompt from the best matches, passing it back to the LLM for a more context-aware SQL generation. **Note:** This is same as the code snippet we started with in our previous post.

```csharp

var sqlPlugin = kernel.CreateFunctionFromPrompt(prompt);
var response = await sqlPlugin.InvokeAsync(kernel, new KernelArguments { ["instructions"] = instructions, ["csvData"] = csvData});

Console.WriteLine("Generated SQL:");
Console.WriteLine(response);

```

## **Conclusion**

By taking advantage of Semantic Kernel’s built-in in-memory store, you already have a workable solution for Retrieval Augmented Generation. Here’s why this approach helps:

 - **No Extra Dependencies:** Everything runs locally, stored in memory.

 - **Faster Setup:** No database or external service to configure.

 - **Improved Accuracy:** The LLM sees only relevant lines from your data.

That said, this is just a starting point. Now, this is a simple example with a few lines of data to understand the concept. Also, the in-memory store is limited to the current session. If you need your embeddings to persist across sessions or handle hundreds of thousands of lines, you can seamlessly extend this pattern to an external vector store. 

Next Step: External Vector Stores

In our next post, we’ll look at how to replace this ephemeral in-memory store with a persistent vector database—like Milvus or QDrant, enabling truly scalable RAG.

All the source code for this is available [here](https://github.com/AdiThakker/SemanticKernel.Embeddings)

Stay tuned for more!