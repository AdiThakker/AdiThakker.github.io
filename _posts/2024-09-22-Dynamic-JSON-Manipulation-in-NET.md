---
layout:     post
title:      Dynamic JSON Manipulation in .NET
date:       2024-09-22
summary:    A quick reference on how to manipulate JSON dynamically in .NET. 
categories: .NET, JSON
---

Recently on a project that i was working on, i was faced with a choice: should we use strongly-typed objects for safety and structure, or should we opt for dynamic JSON manipulation for flexibility and speed? Each approach has its merits, but in this post, we’ll explore how to leverage dynamic JSON manipulation using System.Text.Json—while also touching on when it’s beneficial to use strongly-typed objects.

We’ll walk through a simple example of a common JSON modification task, discuss the reasoning behind immutability and pure functions, and conclude with a comparison of the benefits and drawbacks of each approach.

## Dynamic JSON Manipulation

***NOTE:*** I have simplified the JSON structure for the sake of clarity. The same principles apply to more complex JSON structures (like i had in my actual project).

So, let's consider this JSON structure:

~~~json
{
  "items": [
    { "id": 1, "name": "item1" },
    { "id": 2, "name": "item2" }
  ]
}
~~~

 Let's say we want to  modify the `name` of the first item to `item3`. The dynamic approach allows you to work directly with the JSON structure without converting it into strongly-typed objects. This is particularly useful when dealing with flexible or unpredictable data structures, such as responses from third-party APIs.

### Modifying JSON with JsonDocument and JsonElement

JsonDocument and JsonElement are core to System.Text.Json and provide an efficient way to parse, read, and modify JSON data. Here’s an example of how to modify a JSON structure dynamically:

~~~csharp
public string ModifyJson(string originalJson)
{
    using var jsonDoc = JsonDocument.Parse(originalJson);
    using var stream = new MemoryStream();
    using var writer = new Utf8JsonWriter(stream);

    writer.WriteStartObject();

    foreach (var prop in jsonDoc.RootElement.EnumerateObject())
    {
        if (prop.NameEquals("items"))
        {
            writer.WriteStartArray("items");

            foreach (var item in prop.Value.EnumerateArray())
            {
                item.WriteTo(writer); // Copy original items
            }

            // Copy and modify the first item
            var firstItem = prop.Value[0];
            writer.WriteStartObject();
            foreach (var itemProp in firstItem.EnumerateObject())
            {
                if (itemProp.NameEquals("name"))
                {
                    writer.WriteNumber("name", "item3"); // Change the name property
                }
                else
                {
                    itemProp.WriteTo(writer); // Copy other properties as-is
                }
            }
            writer.WriteEndObject();

            writer.WriteEndArray();
        }
        else
        {
            prop.WriteTo(writer); // Write other unchanged properties
        }
    }

    writer.WriteEndObject();
    return Encoding.UTF8.GetString(stream.ToArray());
}
~~~

In the above snippet:

- JsonDocument is used to parse and read the original JSON.

- Utf8JsonWriter creates a new JSON structure by writing the modified elements.

- The items array is processed by copying all original items, then modifying the first one before appending it to the array.

This method is fast, avoids serialization overhead, and gives you flexibility when dealing with unknown or constantly changing data structures.

Here’s an example using JsonNode to achieve the same result:

~~~csharp
public string ModifyJsonWithJsonNode(string originalJson)
{
    var jsonNode = JsonNode.Parse(originalJson)!;
    var itemsArray = jsonNode["items"]!.AsArray();

    // Clone the first item and modify its "name" property
    var newItem = itemsArray[0]!.Clone();
    newItem["name"] = "item3";

    // Append the modified item to the array
    itemsArray.Add(newItem);

    return jsonNode.ToJsonString();
}
~~~

In this example, JsonNode provides a more concise way to manipulate JSON data. It’s a good choice when you need to work with JSON in a more dynamic and flexible manner.

- JsonNode.Parse parses the original JSON into a mutable structure.

- We retrieve the items array and clone the first element, modifying its id property.

- Finally, we append the modified item to the array and serialize the updated JSON back into a string.


### Benefits of Dynamic JSON Manipulation
***Flexibility:*** Dynamic manipulation is perfect when the structure of your JSON is not well defined or may change frequently. It allows you to work with the JSON directly without needing to update object models constantly. *** In my project we were writing a transformation rule engine where the rules were stored in Text or JSON and the structure of the rules could change frequently.***

***Performance:*** It can be more efficient to work directly with JSON data rather than serializing and deserializing into C# objects, especially for small and quick modifications.

***Simplicity:*** For small tasks or data transformations, working dynamically with JsonDocument or JsonNode offers a lightweight alternative to defining and maintaining strongly-typed objects.

However, dynamic manipulation comes with its own set of challenges. Since there’s no compile-time type checking, it’s easy to introduce runtime errors, and debugging can be more difficult, especially in complex or nested data structures.

### Strongly-Typed Objects

I am not going to cover the Strongly-Typed Objects here, since we all know how to work with them. But I'll list some of their benefits below:

### Benefits of Strongly-Typed Objects:

***Compile-time Safety:*** You get type-checking at compile-time, meaning errors like mistyped properties or incorrect data types are caught before you run the code.

***Readability and Maintainability:*** Object models make your code more self-documenting. Developers can easily understand the structure of the data by looking at the class definitions.

***IDE Support:*** Features like IntelliSense, auto-completion, and refactoring tools improve productivity and reduce the chance of errors.


### Conclusion
Ultimately, both approaches are valuable tools in your .NET development toolkit. Understanding when to use each one will help you make smarter decisions and write more effective code.