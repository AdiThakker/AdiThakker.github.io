---
layout:     post
title:      Leveraging GitHub Copilot for automating File-Based SQL Generation
date:       2024-10-22
summary:    A quick guide on how to leverage GitHub Copilot to generate SQL files. 
categories: GitHub, Copilot, VSCode
---

This past weekend, I had a fascinating discussion with a few colleagues about GitHub Copilot Chat. We explored how it could assist in code generation, specifically by reading instructions from one file, data from another, and generate code (in our case SQL statements) based on those inputs.

As we delved deeper, I discovered that by utilizing GitHub Copilot Chat [participants or agents](https://docs.github.com/en/copilot/using-github-copilot/asking-github-copilot-questions-in-your-ide#using-keywords-in-your-prompt) like `@Workspace`, [slash commands](https://docs.github.com/en/copilot/using-github-copilot/asking-github-copilot-questions-in-your-ide#slash-commands) and [variables or scopes](https://docs.github.com/en/copilot/using-github-copilot/asking-github-copilot-questions-in-your-ide#chat-variables) such as `#file`, we can achieve dynamic and automated orchestration workflows where SQL queries are generated based on **externalized configuration and data files**.

## Chat Participants and Slash Commands

Chat Participants and Slash Commands, such as `@Workspace` and others shown below, are specialized tools that integrate with Copilot to provide additional functionalies and task automation simplifying your prompt. 

![Setup]({{site.url}}/images/copilot-img-1.png){:height="500px" width="700px"}

## Variables / Scopes

Variables or Scopes, such as `#file`, are used to provide specific context or environment, thereby enabling Copilot to understand the scope of the prompt and generate responses accordingly.

Equipped with the above, Copilot Chat becomes a powerful ally, taking care of the repetitive aspects of generating SQL queries and allowing developers to focus on the business logic.

## Workflow Setup

Let’s say we have two files:

1. instructions.txt – This file contains rules or instructions on how to structure SQL queries for specific workspace agents.

2. data.csv – This file contains the raw data (e.g., table names, column names) that needs to be used in the SQL queries.


Our goal is to get GitHub Copilot Chat to:

1. Read the instructions from instructions.txt.

2. Extract relevant data from data.csv.

3. Use both the instructions and the data to generate custom SQL statements.


## Example: The Input Files
Here’s what the files might look like:

>instructions.txt

```txt
SELECT statement for agent workspaces should follow this format:
- FROM the table [workspace_name]
- SELECT all columns where status = 'active'
- The query should be limited to 100 results.

If a workspace has more than 1 table, join them on the 'id' field.
```

>data.csv

```csv
workspace_name,table1,table2
SalesWorkspace,sales_data,NULL
HRWorkspace,employees,NULL
FinanceWorkspace,transactions,account_details
```

## Instructions for Copilot Chat

In Copilot Chat, we can start by instructing it to read the instructions from `instructions.txt`. Then, we ask it to process `data.csv` to generate the corresponding SQL queries.


***Note: Please make sure to open the files in your workspace before running the Copilot Chat commands.***

Following screenshot shows the Copilot Chat commands and the generated SQL queries:

![Setup]({{site.url}}/images/copilot-img-2.png){:height="500px" width="700px"}

### Making it dynamic

The above setup can easily be adapted for more complex scenarios. For instance, if you needed to add more rules or handle additional tables or joins, you could simply update the `instructions.txt` file and let Copilot Chat regenerate the queries accordingly.

Similarly, if the `data.csv` file is updated with new workspaces, Copilot Chat can regenerate the corresponding SQL queries, making this process both scalable and maintainable.

***Note: your actual IDE experience may vary. For me omitting the @Workspace in the prompt failed to generate the SQL queries. Also you might be able to accomplish similar results using different variation of the prompt. The above example just highlights how we accomplished our task.***

So there you have it! We quickly saw how GitHub Copilot Chat can be used to automate SQL generation based on file-based inputs. 
This approach can be extended to other use cases where you need to generate code based on externalized configuration or data files.such as generating API endpoints, handling database migrations, etc.

Happy coding with Copilot Chat! 🚀

