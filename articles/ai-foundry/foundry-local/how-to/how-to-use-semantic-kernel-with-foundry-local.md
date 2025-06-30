---
title: Use Semantic Kernel with Foundry Local
titleSuffix: Foundry Local
description: Learn how to integrate Microsoft Semantic Kernel with Foundry Local for on-device AI inference in .NET applications.
manager: scottpolly
ms.service: azure-ai-foundry
ms.subservice: foundry-local
ms.custom: build-2025
ms.topic: how-to
ms.author: jburchel
ms.reviewer: samkemp
ms.date: 06/28/2025
zone_pivot_groups: foundry-local-semantic-kernel
author: jonburchel
reviewer: samuel100
---

# Use Semantic Kernel with Foundry Local

[!INCLUDE [foundry-local-preview](./../includes/foundry-local-preview.md)]

This guide shows you how to integrate Microsoft Semantic Kernel with Foundry Local to run AI models locally in your .NET applications. Semantic Kernel is Microsoft's open-source SDK that helps orchestrate AI models, prompts, and planning in .NET applications.

By combining Semantic Kernel with Foundry Local, you can:

- Run AI models completely offline with low latency
- Keep sensitive data on-device for privacy and compliance
- Eliminate ongoing cloud inference costs
- Build responsive AI experiences with instant model responses

## Prerequisites

- Foundry Local installed. See the [Get started with Foundry Local](../get-started.md) article for installation instructions.
- .NET 6.0 or later
- Basic familiarity with C# and .NET development

::: zone pivot="programming-language-python"
[!INCLUDE [Python](../includes/use-semantic-kernel/python.md)]
::: zone-end
::: zone pivot="programming-language-csharp"
[!INCLUDE [C#](../includes/use-semantic-kernel/csharp.md)]
::: zone-end

## How it works

Foundry Local exposes an OpenAI-compatible REST API endpoint on your local machine. This means Semantic Kernel can treat the local model just like an OpenAI or Azure OpenAI model endpoint.

The integration works by:

1. **Model Selection**: Foundry Local automatically selects the optimal model variant for your hardware (CPU, GPU, or NPU)
2. **Local Endpoint**: The Foundry Local service provides an OpenAI-compatible API locally
3. **Simple Integration**: Semantic Kernel connects to this local endpoint using the standard OpenAI connector

## Model recommendations

For optimal performance with Semantic Kernel, consider these model recommendations:

- **Phi-4-mini**: Excellent for general chat and reasoning tasks with fast inference
- **Qwen2.5-0.5B**: Ultra-lightweight model for simple tasks and resource-constrained environments. You're encourage to experiment with the coder variant for code generation tasks.
- **DeepSeek-R1-Distill-Qwen-1.5B**: Good balance of capability and performance for code generation

Run `foundry model list` to see all available models for your hardware configuration.

## Best practices

- **Model Management**: Use model aliases (like `phi-4-mini`) rather than specific variant names for better cross-platform compatibility
- **Resource Monitoring**: Monitor system resources when running larger models locally
- **Error Handling**: Implement proper error handling for model loading and inference failures
- **Caching**: Leverage Foundry Local's model caching to avoid repeated downloads

## Troubleshooting

### Model not found errors

If you encounter model not found errors, ensure the model is properly loaded:

```bash
foundry model load phi-4-mini
foundry cache list  # Verify the model is cached
```

### Connection errors

If Semantic Kernel can't connect to the local endpoint:

1. Verify the Foundry Local service is running: `foundry service status`
2. Check the endpoint URL matches the service address
3. Ensure no firewall is blocking local connections

### Performance optimization

For better performance:

- Use GPU-accelerated models when available
- Adjust batch sizes based on your hardware capabilities
- Consider model quantization for memory-constrained environments

## Next steps

- [Explore the Semantic Kernel documentation](https://learn.microsoft.com/semantic-kernel/)
- [Learn about Foundry Local architecture](../concepts/foundry-local-architecture.md)
- [Integrate with other inference SDKs](how-to-integrate-with-inference-sdks.md)
- [Compile custom Hugging Face models](how-to-compile-hugging-face-models.md)
