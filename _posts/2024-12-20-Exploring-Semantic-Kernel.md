---
layout:     post
title:      Exploring Semantic Kernel
date:       2024-12-20
summary:    An extension to one of my previous post on how to leverage Semantic Kernel to automatically generate SQL. 
categories: Gen AI, Semantic Kernel
---

In my [previous blog post]({{site.url}}/VsCode-CoPilot-CodeGeneration), we explored how **GitHub Copilot Chat** can dynamically generate SQL queries based on instructions and data files. It demonstrated how tools like `@Workspace`, slash commands, and scoped variables such as `#file` could orchestrate workflows efficiently. 

This is all well and good, but it's all within an IDE. What if we want to build this functionality into a custom application?  This blog post builds on that by leveraging **Semantic Kernel**  and  **Ollama - Local LLM** to create a custom SQL generation application.

## **Why Extend to a Custom Application?**

GitHub Copilot Chat is tightly integrated into IDEs, which makes it convenient for developers. However, a custom application provides as excellent learning opportunity to understand how Gen AI workflow works and also allows for:  
- **Full Model Control**: Host and manage your own LLM (e.g., Ollama) for complete autonomy.  
- **Enhanced Privacy**: Keep your data local without relying on cloud services.  
- **Extensibility**: Adapt the workflow to generate not only SQL queries but also API endpoints, test cases, or infrastructure code.  
- **Cost Efficiency**: Eliminate cloud API usage fees by running LLMs locally.  

With **Semantic Kernel**, we can replicate and extend Copilot’s functionality, integrating prompt templates, dynamic arguments, and file-based workflows into a standalone application.


## **What is Semantic Kernel?**

**[Semantic Kernel](https://github.com/microsoft/semantic-kernel)** is an open-source SDK from Microsoft designed to orchestrate AI-driven workflows. It provides abstraction to AI models, plugins, and semantic functions, enabling developers to: 

- **Integrate Natural Language Understanding**: Combine AI-driven tasks like text generation or summarization into applications.  
- **Use Plugins and Functions**: Build modular and reusable workflows with plugins or prompt-based semantic functions.  
- **Leverage AI Models Flexibly**: Work with cloud-based services like OpenAI or local LLMs like Ollama.  

## **What is Ollama?**

**[Ollama](https://github.com/ollama/ollama)** is a lightweight local runtime for hosting large language models (LLMs) on your machine. It provides:  
- **Privacy**: All data processing happens locally, ensuring complete control over sensitive information.  
- **Cost Efficiency**: Avoid cloud API fees by running LLMs like **phi3** locally.  
- **Performance Optimization**: Ollama is designed to run efficiently on modern hardware, making it ideal for developers needing on-premises AI capabilities.  

So with that, let's dive into how we can leverage **Semantic Kernel** and **Ollama** to build a custom SQL generation application.

**NOTE:** All the source code is available [here](https://github.com/AdiThakker/SemanticKernel.Plugins) for reference.  


## **Getting Started with Ollama on Windows**

### **Install Ollama**
1. Download the **Ollama installer** for Windows from the [official website](https://ollama.ai/download).
2. Follow the installation steps to set up Ollama on your machine.

### **Install the Model**
Once Ollama is installed, use the following PowerShell command to download and install the **phi3** model:
```powershell
ollama pull phi3
```

### **Verify and run the Model**
```powershell
ollama list
ollama run phi3
```

Following is a screenshot of the installed model:
![Setup]({{site.url}}/images/sk-1.png){:height="500px" width="900px"}

**NOTE**: By default, Ollama runs at http://localhost:11434. You can now integrate this local LLM with Semantic Kernel.


## **Wofkflow Setup**

All the related files are available in the [Plugin](https://github.com/AdiThakker/SemanticKernel.Plugins/tree/main/SemanticKernel.Plugins/Plugin) folder. Let's revisit the files from the previous post:

### **instructions.txt**

This file contains rules or instructions on how to structure SQL queries.

```plaintext
SELECT statement for agent workspaces should follow this format:
- FROM the table [workspace_name]
- SELECT all columns where status = 'active'
- The query should be limited to 100 results.

If a workspace has more than 1 table, join them on the 'id' field.
```

### **data.csv**

This file contains the raw data (e.g., table names, column names) that needs to be used in the SQL queries.

```plaintext
workspace_name,table1,table2
SalesWorkspace,sales_data,NULL
HRWorkspace,employees,NULL
FinanceWorkspace,transactions,account_details
```

### **prompt.txt**

This file contains the prompt template for generating SQL queries. It uses placeholders `{{$instructions}}` and `{{$csvData}}` to dynamically replace the content of `instructions.txt` and `data.csv`.

```plaintext
You are an SQL assistant. Based on the instructions and data provided, generate SQL queries.  

### Instructions:  
{{$instructions}}  

### Data:  
{{$csvData}}  

Rules:  
- If only one table exists, select all columns where status = 'active', limited to 100 rows.  
- If two tables exist, join them on the 'id' field and apply the same rules.  
```

## **Semantic Kernel Integration**

Following is the code snippet to integrate Semantic Kernel with Ollama for generating SQL queries:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.TextGeneration;
using System.IO;

class Program
{
    static async Task Main(string[] args)
    {
        // Step 1: Initialize Semantic Kernel with Ollama Text Generation
        var builder = Kernel
            .CreateBuilder()
            .AddOllamaTextGeneration("phi3", new Uri("http://localhost:11434"));

        var kernel = builder.Build();

        // Step 2: Load external files
        var instructions = await File.ReadAllTextAsync("Plugin/instructions.txt");
        var csvData = await File.ReadAllTextAsync("Plugin/data.csv");
        var prompt = await File.ReadAllTextAsync("Plugin/prompt.txt");

        // Step 3: Create a semantic function from the prompt template
        var sqlPlugin = kernel.CreateFunctionFromPrompt(prompt);

        // Step 4: Generate SQL by invoking the function with dynamic arguments
        var response = await sqlPlugin.InvokeAsync(kernel, new KernelArguments
        {
            ["instructions"] = instructions,
            ["csvData"] = csvData
        });

        // Step 5: Output the generated SQL
        Console.WriteLine("Generated SQL:");
        Console.WriteLine(response);
    }
}
```

The above code snippet leverages semantic kernel to create a function from the prompt template **CreateFunctionFromPrompt**. It then invokes the function with dynamic arguments from the external files
The place holders `{{$instructions}}` and `{{$csvData}}` are replaced with the content of `instructions.txt` and `data.csv` respectively. For more details on Semantic Kernel concepts, refer to the [official documentation](
https://learn.microsoft.com/en-us/semantic-kernel/concepts/semantic-kernel-components?pivots=programming-language-csharp).

## **Running the Application**
Following is the screenshot of the generated SQL:
![Setup]({{site.url}}/images/sk-2.png){:height="500px" width="900px"}


## Final Thoughts
By combining Semantic Kernel and Ollama, we’ve created a flexible, extensible workflow for SQL generation. Whether automating SQL queries, generating API endpoints, or creating infrastructure-as-code scripts, this setup provides an excellent foundation for extending the template for other use cases.

What would you build with Semantic Kernel? Let me know in the comments below! 🚀

Happy coding!