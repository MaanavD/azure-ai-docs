---
ms.service: azure-ai-foundry
ms.subservice: foundry-local
ms.custom: build-2025
ms.topic: include
ms.date: 06/28/2025
ms.author: jburchel
reviewer: samuel100
ms.reviewer: samkemp
author: jonburchel
---

## Create a new project

Create a new C# console application:

```bash
dotnet new console -n SemanticKernelFoundryLocal
cd SemanticKernelFoundryLocal
```

## Install NuGet packages

Install the required NuGet packages:

```bash
dotnet add package Microsoft.SemanticKernel
dotnet add package Microsoft.SemanticKernel.Connectors.OpenAI
dotnet add package Microsoft.AI.Foundry.Local
```

## Download and load a model

First, download and load a model using the Foundry Local CLI:

```bash
foundry model download phi-4-mini
foundry model load phi-4-mini
```

Alternatively, start the service which will automatically handle the model:

```bash
foundry service start --model phi-4-mini
```

## Create a simple chat application

Replace the contents of `Program.cs` with the following code:

```csharp
using Microsoft.AI.Foundry.Local;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.OpenAI;

// Model alias - Foundry Local automatically selects the best variant for your hardware
const string modelAlias = "phi-4-mini";

// Initialize Foundry Local manager and start the model
var manager = await FoundryLocalManager.StartModelAsync(modelAlias);
var modelInfo = await manager.GetModelInfoAsync(modelAlias);

// Create Semantic Kernel builder and add OpenAI chat completion service
var builder = Kernel.CreateBuilder();
builder.AddOpenAIChatCompletion(
    modelId: modelInfo.ModelId,
    endpoint: manager.Endpoint,
    apiKey: manager.ApiKey
);

var kernel = builder.Build();

// Get the chat completion service
var chatService = kernel.GetRequiredService<IChatCompletionService>();

// Create chat history with system message
var chatHistory = new ChatHistory();
chatHistory.AddSystemMessage("""
    You are a helpful AI assistant running locally on the user's device.
    You can help with coding questions, general knowledge, and creative tasks.
    Be concise but informative in your responses.
    """);

Console.WriteLine("Semantic Kernel + Foundry Local Chat");
Console.WriteLine("Type 'exit', 'quit', or 'bye' to end the conversation");
Console.WriteLine(new string('-', 50));

// Main chat loop
while (true)
{
    Console.Write("User: ");
    var userInput = Console.ReadLine();

    if (string.IsNullOrWhiteSpace(userInput) ||
        userInput.Equals("exit", StringComparison.OrdinalIgnoreCase) ||
        userInput.Equals("quit", StringComparison.OrdinalIgnoreCase) ||
        userInput.Equals("bye", StringComparison.OrdinalIgnoreCase))
    {
        Console.WriteLine("\nGoodbye!");
        break;
    }

    // Add user message to chat history
    chatHistory.AddUserMessage(userInput);

    // Get response from the local model
    var response = await chatService.GetChatMessageContentAsync(chatHistory);

    if (!string.IsNullOrEmpty(response.Content))
    {
        Console.WriteLine($"Assistant: {response.Content}");

        // Add assistant response to chat history for context
        chatHistory.AddAssistantMessage(response.Content);
    }
    else
    {
        Console.WriteLine("Assistant: [No response received]");
    }
}
```

## Run the application

Build and run your application:

```bash
dotnet run
```

## Example output

```text
Semantic Kernel + Foundry Local Chat
Type 'exit', 'quit', or 'bye' to end the conversation
--------------------------------------------------
User: What are the advantages of local AI inference?
Assistant: Local AI inference offers several key advantages:

1. **Privacy**: Your data stays on your device, ensuring complete privacy
2. **Speed**: No network latency means faster responses
3. **Offline capability**: Works without internet connection
4. **Cost-effective**: No per-request API charges
5. **Reliability**: No dependency on cloud service availability
6. **Control**: Full control over model selection and behavior

User: exit
Goodbye!
```

## Advanced example with streaming

For a more responsive experience, you can use streaming responses:

```csharp
using Microsoft.AI.Foundry.Local;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;

const string modelAlias = "phi-4-mini";

// Initialize Foundry Local
var manager = await FoundryLocalManager.StartModelAsync(modelAlias);
var modelInfo = await manager.GetModelInfoAsync(modelAlias);

// Create kernel with OpenAI chat completion
var builder = Kernel.CreateBuilder();
builder.AddOpenAIChatCompletion(
    modelId: modelInfo.ModelId,
    endpoint: manager.Endpoint,
    apiKey: manager.ApiKey
);

var kernel = builder.Build();
var chatService = kernel.GetRequiredService<IChatCompletionService>();

// Create chat history
var chatHistory = new ChatHistory();
chatHistory.AddSystemMessage("You are a helpful AI assistant.");

Console.WriteLine("Streaming Chat with Semantic Kernel + Foundry Local");
Console.WriteLine("Type 'exit' to quit");
Console.WriteLine(new string('-', 50));

while (true)
{
    Console.Write("User: ");
    var userInput = Console.ReadLine();

    if (string.IsNullOrEmpty(userInput) || userInput.ToLower() == "exit")
        break;

    chatHistory.AddUserMessage(userInput);

    Console.Write("Assistant: ");

    // Stream the response
    var fullResponse = "";
    await foreach (var chunk in chatService.GetStreamingChatMessageContentsAsync(chatHistory))
    {
        if (!string.IsNullOrEmpty(chunk.Content))
        {
            Console.Write(chunk.Content);
            fullResponse += chunk.Content;
        }
    }
    Console.WriteLine("\n");

    // Add the complete response to chat history
    if (!string.IsNullOrEmpty(fullResponse))
    {
        chatHistory.AddAssistantMessage(fullResponse);
    }
}
```

## Using Semantic Kernel functions

Semantic Kernel's function calling capabilities work seamlessly with Foundry Local. Here's an example that clearly demonstrates when functions are being called:

```csharp
using Microsoft.AI.Foundry.Local;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.OpenAI;
using System.ComponentModel;

// Define a plugin with utility functions that clearly show when they're called
public class UtilityPlugin
{
    [KernelFunction, Description("Get the current date and time")]
    public string GetCurrentTime()
    {
        var currentTime = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");
        Console.WriteLine($"🔧 FUNCTION CALLED: GetCurrentTime() -> {currentTime}");
        return currentTime;
    }

    [KernelFunction, Description("Reverse the order of characters in text")]
    public string ReverseText([Description("The text to reverse")] string text)
    {
        var reversed = new string(text.Reverse().ToArray());
        Console.WriteLine($"🔧 FUNCTION CALLED: ReverseText('{text}') -> '{reversed}'");
        return reversed;
    }

    [KernelFunction, Description("Count the number of words in text")]
    public int CountWords([Description("The text to count words in")] string text)
    {
        var wordCount = text.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
        Console.WriteLine($"🔧 FUNCTION CALLED: CountWords('{text}') -> {wordCount}");
        return wordCount;
    }

    [KernelFunction, Description("Generate a random number between min and max")]
    public int GenerateRandomNumber([Description("Minimum value")] int min, [Description("Maximum value")] int max)
    {
        var random = new Random();
        var result = random.Next(min, max + 1);
        Console.WriteLine($"🔧 FUNCTION CALLED: GenerateRandomNumber({min}, {max}) -> {result}");
        return result;
    }
}

class Program
{
    static async Task Main(string[] args)
    {
        const string modelAlias = "phi-4-mini";

        // Initialize Foundry Local
        var manager = await FoundryLocalManager.StartModelAsync(modelAlias);
        var modelInfo = await manager.GetModelInfoAsync(modelAlias);

        // Create kernel and add services
        var builder = Kernel.CreateBuilder();
        builder.AddOpenAIChatCompletion(
            modelId: modelInfo.ModelId,
            endpoint: manager.Endpoint,
            apiKey: manager.ApiKey
        );

        var kernel = builder.Build();

        // Add the utility plugin
        kernel.Plugins.AddFromType<UtilityPlugin>();

        // Configure execution settings to enable function calling
        var executionSettings = new OpenAIPromptExecutionSettings
        {
            ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions
        };

        // Test function calling with clear examples
        var testPrompts = new[]
        {
            "What time is it right now?",
            "Can you reverse the text 'Hello World'?",
            "How many words are in this sentence: 'The quick brown fox jumps'?",
            "Generate a random number between 1 and 100",
            "Please get the current time, then reverse the text 'Semantic Kernel'"
        };

        foreach (var prompt in testPrompts)
        {
            Console.WriteLine($"\n📝 Question: {prompt}");
            Console.WriteLine("📋 Function calls (if any):");

            var result = await kernel.InvokePromptAsync(prompt, new KernelArguments(executionSettings));
            Console.WriteLine($"✅ Final Answer: {result}");
            Console.WriteLine(new string('-', 60));
        }
    }
}
```

This example demonstrates function calling more clearly because:

1. **Function Call Logging**: Each function prints when it's called with input/output
2. **Non-trivial Functions**: Functions like getting current time and generating random numbers can't be easily done by the model alone
3. **Multiple Function Calls**: Some prompts require calling multiple functions in sequence
4. **Clear Visual Indicators**: Uses emojis and formatting to make function calls obvious

## Expected output

When you run this example, you'll see output like:

```text
📝 Question: What time is it right now?
📋 Function calls (if any):
🔧 FUNCTION CALLED: GetCurrentTime() -> 2025-06-28 14:30:25
✅ Final Answer: The current time is 2025-06-28 14:30:25.

📝 Question: Can you reverse the text 'Hello World'?
📋 Function calls (if any):
🔧 FUNCTION CALLED: ReverseText('Hello World') -> 'dlroW olleH'
✅ Final Answer: The reversed text is 'dlroW olleH'.

📝 Question: Generate a random number between 1 and 100
📋 Function calls (if any):
🔧 FUNCTION CALLED: GenerateRandomNumber(1, 100) -> 42
✅ Final Answer: I generated a random number for you: 42.
```

> [!NOTE]
> If you don't see the "🔧 FUNCTION CALLED" messages, it means the model is answering without using the functions. This can happen if:
>
> - The model doesn't recognize it needs to use a function
> - The function descriptions aren't clear enough
> - The model thinks it can answer directly without function calls
> - Function calling isn't properly configured
>
> Function calling support varies by model. Phi-4-mini has good function calling capabilities, but some smaller models may not support it reliably.

## Alternative approach with explicit function prompting

If the automatic function calling doesn't work consistently, you can explicitly mention the available functions in your prompts:

```csharp
// Alternative approach: Explicitly list available functions in the system prompt
var systemPrompt = """
You are a helpful AI assistant with access to the following functions:
- GetCurrentTime(): Gets the current date and time
- ReverseText(text): Reverses the order of characters in the given text
- CountWords(text): Counts the number of words in the given text
- GenerateRandomNumber(min, max): Generates a random number between min and max

When a user asks for something that can be done with these functions, you MUST use the appropriate function rather than trying to answer directly. Always call the function when available.
""";

// Test with more explicit prompts
var explicitPrompts = new[]
{
    "Use the GetCurrentTime function to tell me what time it is",
    "Use the ReverseText function to reverse 'Hello World'",
    "Use the CountWords function to count words in 'The quick brown fox jumps'",
    "Use the GenerateRandomNumber function to get a random number between 1 and 100"
};

foreach (var prompt in explicitPrompts)
{
    Console.WriteLine($"\n📝 Explicit Question: {prompt}");
    Console.WriteLine("📋 Function calls (if any):");

    var arguments = new KernelArguments(executionSettings) { ["input"] = prompt };
    var result = await kernel.InvokePromptAsync($"{systemPrompt}\n\nUser: {{$input}}", arguments);
    Console.WriteLine($"✅ Final Answer: {result}");
    Console.WriteLine(new string('-', 60));
}
```

## Performance tips

- **Model Selection**: Choose models appropriate for your hardware capabilities
- **Memory Management**: Monitor memory usage when running larger models
- **Batch Processing**: Process multiple requests together when possible
- **Async/Await**: Use async patterns for better application responsiveness
