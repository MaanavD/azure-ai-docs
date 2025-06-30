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

## Install pip packages

Install the required Python packages:

```bash
pip install semantic-kernel
pip install foundry-local-sdk
pip install openai
```

> [!TIP]
> We recommend using a virtual environment to avoid package conflicts. You can create a virtual environment using either `venv` or `conda`.

## Download and load a model

First, download and load a model using the Foundry Local CLI:

```bash
foundry model download phi-4-mini
foundry model load phi-4-mini
```

Alternatively, you can run a model which will automatically download and load the model:

```bash
foundry model run phi-4-mini
```

## Create a simple chat application

Create a Python file named `semantic_kernel_chat.py` and copy the following code:

```python
import asyncio
from foundry_local import FoundryLocalManager
from openai import AsyncOpenAI
from semantic_kernel.connectors.ai.open_ai import OpenAIChatCompletion, OpenAIChatPromptExecutionSettings
from semantic_kernel.contents import ChatHistory

# The model alias - Foundry Local picks the right variant for your hardware
model_alias = "phi-4-mini"

# Initialize the Foundry Local manager
manager = FoundryLocalManager(model_alias)

# Download and load the model (if not already done)
manager.download_model(model_alias)
manager.load_model(model_alias)

# Create the Semantic Kernel OpenAI chat completion service
service = OpenAIChatCompletion(
    ai_model_id=manager.get_model_info(model_alias).id,
    async_client=AsyncOpenAI(
        base_url=manager.endpoint,
        api_key=manager.api_key,
    ),
)

# Configure execution settings
request_settings = OpenAIChatPromptExecutionSettings()

# System message to define the assistant's behavior
system_message = """
You are a helpful AI assistant running locally on the user's device.
You can help with coding questions, general knowledge, and creative tasks.
Be concise but informative in your responses.
"""

# Create chat history with system message
chat_history = ChatHistory(system_message=system_message)

async def chat() -> bool:
    """Handle a single chat interaction."""
    try:
        user_input = input("User: ")
    except (KeyboardInterrupt, EOFError):
        print("\n\nExiting chat...")
        return False

    if user_input.lower() in ["exit", "quit", "bye"]:
        print("\n\nGoodbye!")
        return False

    # Add user message to chat history
    chat_history.add_user_message(user_input)

    # Get response from the local model
    response = await service.get_chat_message_content(
        chat_history=chat_history,
        settings=request_settings,
    )

    if response:
        print(f"Assistant: {response}")
        # Add assistant response to chat history for context
        chat_history.add_message(response)

    return True

async def main() -> None:
    """Main chat loop."""
    print("Semantic Kernel + Foundry Local Chat")
    print("Type 'exit', 'quit', or 'bye' to end the conversation")
    print("-" * 50)

    # Start the chat loop
    chatting = True
    while chatting:
        chatting = await chat()

if __name__ == "__main__":
    asyncio.run(main())
```

## Run the application

Execute your Python script:

```bash
python semantic_kernel_chat.py
```

## Example output

```text
Semantic Kernel + Foundry Local Chat
Type 'exit', 'quit', or 'bye' to end the conversation
--------------------------------------------------
User: Explain the benefits of running AI models locally
Assistant: Running AI models locally offers several key benefits:

1. **Privacy & Security**: Your data never leaves your device, ensuring complete privacy and meeting strict compliance requirements.

2. **No Internet Required**: Once models are downloaded, you can run inference completely offline.

3. **Low Latency**: Eliminate network delays for near-instantaneous responses.

4. **Cost Savings**: No per-API-call charges after initial setup.

5. **Always Available**: No dependency on cloud service availability or rate limits.

6. **Customization**: Full control over model selection and configuration for your specific use case.

User: exit
Goodbye!
```

## Function calling setup (for reference)

The following example shows how to set up automatic function calling with Semantic Kernel. Note that this may not work reliably with all local models:

```python
# This example demonstrates the setup for automatic function calling
# However, local models may not support this feature reliably
import asyncio
import datetime
from foundry_local import FoundryLocalManager
from openai import AsyncOpenAI
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import OpenAIChatCompletion
from semantic_kernel.functions import kernel_function

class UtilityPlugin:
    """A plugin with utility functions that clearly show when they're called."""

    @kernel_function(name="get_current_time", description="Get the current date and time")
    def get_current_time(self) -> str:
        """Get the current date and time."""
        current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        print(f"🔧 FUNCTION CALLED: get_current_time() -> {current_time}")
        return current_time

    @kernel_function(name="reverse_text", description="Reverse the order of characters in text")
    def reverse_text(self, text: str) -> str:
        """Reverse the given text."""
        reversed_text = text[::-1]
        print(f"🔧 FUNCTION CALLED: reverse_text('{text}') -> '{reversed_text}'")
        return reversed_text

    @kernel_function(name="count_words", description="Count the number of words in text")
    def count_words(self, text: str) -> int:
        """Count words in the given text."""
        word_count = len(text.split())
        print(f"🔧 FUNCTION CALLED: count_words('{text}') -> {word_count}")
        return word_count

async def test_automatic_function_calling():
    # Initialize Foundry Local
    model_alias = "phi-4-mini"
    manager = FoundryLocalManager(model_alias)
    manager.download_model(model_alias)
    manager.load_model(model_alias)

    # Create kernel and add services
    kernel = Kernel()

    # Add the chat completion service
    chat_service = OpenAIChatCompletion(
        ai_model_id=manager.get_model_info(model_alias).id,
        async_client=AsyncOpenAI(
            base_url=manager.endpoint,
            api_key=manager.api_key,
        ),
    )
    kernel.add_service(chat_service)

    # Add the utility plugin
    kernel.add_plugin(UtilityPlugin(), plugin_name="UtilityPlugin")

    # Configure execution settings to enable automatic function calling
    from semantic_kernel.connectors.ai.open_ai import OpenAIChatPromptExecutionSettings
    execution_settings = OpenAIChatPromptExecutionSettings(
        tool_choice="auto",
        tools_to_invoke=kernel.plugins
    )

    # Test function calling with clear examples
    test_prompts = [
        "What time is it right now?",
        "Can you reverse the text 'Hello World'?",
        "How many words are in this sentence: 'The quick brown fox jumps'?"
    ]

    for prompt in test_prompts:
        print(f"\n📝 Question: {prompt}")
        print("📋 Function calls (if any):")

        result = await kernel.invoke_prompt(prompt, execution_settings=execution_settings)
        print(f"✅ Final Answer: {result}")
        print("-" * 60)

# Uncomment to test automatic function calling (may not work with local models)
# if __name__ == "__main__":
#     asyncio.run(test_automatic_function_calling())
```

This example demonstrates the setup for automatic function calling, though results may vary with local models.

## Expected output

When you run this example, you'll see output like:

```text
📝 Question: What time is it right now?
📋 Function calls (if any):
🔧 FUNCTION CALLED: get_current_time() -> 2025-06-28 14:30:25
✅ Final Answer: The current time is 2025-06-28 14:30:25.

📝 Question: Can you reverse the text 'Hello World'?
📋 Function calls (if any):
🔧 FUNCTION CALLED: reverse_text('Hello World') -> 'dlroW olleH'
✅ Final Answer: The reversed text is 'dlroW olleH'.

📝 Question: How many words are in this sentence: 'The quick brown fox jumps'?
📋 Function calls (if any):
🔧 FUNCTION CALLED: count_words('The quick brown fox jumps') -> 5
✅ Final Answer: There are 5 words in the sentence 'The quick brown fox jumps'.
```

> [!IMPORTANT]
> **Function Calling Limitations**: Current testing shows that automatic function calling may not work reliably with all local models through Foundry Local. The model may provide code examples instead of actually calling the functions. This is a known limitation when using local models compared to cloud-based OpenAI models.
>
> The examples below demonstrate how to set up function calling, but you may need to implement manual function orchestration for reliable results.

## Working alternative: Manual function orchestration

Here's a practical approach that works reliably with local models by implementing manual function orchestration:

```python
import asyncio
import datetime
import re
from foundry_local import FoundryLocalManager
from openai import AsyncOpenAI
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import OpenAIChatCompletion

class UtilityPlugin:
    """A plugin with utility functions that clearly show when they're called."""

    def get_current_time(self) -> str:
        """Get the current date and time."""
        current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        print(f"🔧 FUNCTION CALLED: get_current_time() -> {current_time}")
        return current_time

    def reverse_text(self, text: str) -> str:
        """Reverse the given text."""
        reversed_text = text[::-1]
        print(f"🔧 FUNCTION CALLED: reverse_text('{text}') -> '{reversed_text}'")
        return reversed_text

    def count_words(self, text: str) -> int:
        """Count words in the given text."""
        word_count = len(text.split())
        print(f"🔧 FUNCTION CALLED: count_words('{text}') -> {word_count}")
        return word_count

async def manual_function_orchestration():
    """Demonstrate manual function orchestration that works reliably."""
    # Initialize Foundry Local
    model_alias = "phi-4-mini"
    manager = FoundryLocalManager(model_alias)
    manager.download_model(model_alias)
    manager.load_model(model_alias)

    # Create kernel and add services
    kernel = Kernel()
    chat_service = OpenAIChatCompletion(
        ai_model_id=manager.get_model_info(model_alias).id,
        async_client=AsyncOpenAI(
            base_url=manager.endpoint,
            api_key=manager.api_key,
        ),
    )
    kernel.add_service(chat_service)
    
    # Create utility plugin instance
    utils = UtilityPlugin()
    
    # Define function patterns and handlers
    function_patterns = {
        r'time|current time|what time': lambda: utils.get_current_time(),
        r'reverse.*"([^"]+)"': lambda match: utils.reverse_text(match.group(1)),
        r'reverse.*\'([^\']+)\'': lambda match: utils.reverse_text(match.group(1)),
        r'count.*words.*"([^"]+)"': lambda match: utils.count_words(match.group(1)),
        r'count.*words.*\'([^\']+)\'': lambda match: utils.count_words(match.group(1)),
    }
    
    test_prompts = [
        "What time is it right now?",
        "Can you reverse the text 'Hello World'?",
        "How many words are in this sentence: 'The quick brown fox jumps'?"
    ]
    
    for prompt in test_prompts:
        print(f"\n📝 Question: {prompt}")
        print("📋 Function calls (if any):")
        
        # Check if we should call a function based on the prompt
        function_result = None
        for pattern, handler in function_patterns.items():
            match = re.search(pattern, prompt, re.IGNORECASE)
            if match:
                try:
                    # Check if this pattern has capture groups
                    if '(' in pattern and ')' in pattern:
                        function_result = handler(match)
                    else:
                        function_result = handler()
                    break
                except Exception as e:
                    print(f"Function call failed: {e}")
        
        # Get AI response, potentially incorporating function result
        if function_result is not None:
            enhanced_prompt = f"{prompt}\n\nFunction result: {function_result}\n\nPlease provide a natural response incorporating this result."
        else:
            enhanced_prompt = prompt
            
        result = await kernel.invoke_prompt(enhanced_prompt)
        print(f"✅ Final Answer: {result}")
        print("-" * 60)

if __name__ == "__main__":
    asyncio.run(manual_function_orchestration())
```

This manual orchestration approach:

1. **Uses pattern matching** to detect when functions should be called
2. **Actually calls functions** and shows the logged output
3. **Incorporates results** into the AI response
4. **Works reliably** with local models that don't support automatic function calling

## Expected output with manual orchestration

```text
📝 Question: What time is it right now?
📋 Function calls (if any):
🔧 FUNCTION CALLED: get_current_time() -> 2025-06-28 14:30:25
✅ Final Answer: The current time is 2025-06-28 14:30:25.

📝 Question: Can you reverse the text 'Hello World'?
📋 Function calls (if any):
🔧 FUNCTION CALLED: reverse_text('Hello World') -> 'dlroW olleH'
✅ Final Answer: The reversed text 'Hello World' is 'dlroW olleH'.
```

## Alternative approach with explicit function prompting

If the automatic function calling doesn't work consistently, you can explicitly mention the available functions in your prompts:

```python
# Alternative approach: Explicitly list available functions in the system prompt
async def test_explicit_function_calling():
    # Initialize Foundry Local (same setup as main function)
    model_alias = "phi-4-mini"
    manager = FoundryLocalManager(model_alias)
    manager.download_model(model_alias)
    manager.load_model(model_alias)

    # Create kernel and add services
    kernel = Kernel()

    # Add the chat completion service
    chat_service = OpenAIChatCompletion(
        ai_model_id=manager.get_model_info(model_alias).id,
        async_client=AsyncOpenAI(
            base_url=manager.endpoint,
            api_key=manager.api_key,
        ),
    )
    kernel.add_service(chat_service)

    # Add the utility plugin
    kernel.add_plugin(UtilityPlugin(), plugin_name="UtilityPlugin")

    # Configure execution settings
    from semantic_kernel.connectors.ai.open_ai import OpenAIChatPromptExecutionSettings
    execution_settings = OpenAIChatPromptExecutionSettings(
        tool_choice="auto",
        tools_to_invoke=kernel.plugins
    )

    system_prompt = """
You are a helpful AI assistant with access to the following functions:
- get_current_time(): Gets the current date and time
- reverse_text(text): Reverses the order of characters in the given text
- count_words(text): Counts the number of words in the given text

When a user asks for something that can be done with these functions, you MUST use the appropriate function rather than trying to answer directly. Always call the function when available.
"""

    # Test with more explicit prompts
    explicit_prompts = [
        "Use the get_current_time function to tell me what time it is",
        "Use the reverse_text function to reverse 'Hello World'",
        "Use the count_words function to count words in 'The quick brown fox jumps'"
    ]

    for prompt in explicit_prompts:
        print(f"\n📝 Explicit Question: {prompt}")
        print("📋 Function calls (if any):")

        full_prompt = f"{system_prompt}\n\nUser: {prompt}"
        result = await kernel.invoke_prompt(full_prompt, execution_settings=execution_settings)
        print(f"✅ Final Answer: {result}")
        print("-" * 60)

# Uncomment the line below to test explicit function calling instead of the main function
# if __name__ == "__main__":
#     asyncio.run(test_explicit_function_calling())
```
