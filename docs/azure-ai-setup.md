# Azure AI Configuration Guide

This guide explains how to configure Azure AI (Azure OpenAI) for use with VibSDK through Cloudflare AI Gateway.

## Prerequisites

Before configuring Azure AI, ensure you have:

1. **Azure OpenAI Service** set up in your Azure account
   - An Azure OpenAI resource created
   - API keys available (found in Azure Portal → Azure OpenAI → Keys and Endpoint)
   - At least one model deployed (e.g., GPT-4, GPT-3.5-turbo)

2. **Cloudflare AI Gateway** configured
   - AI Gateway created in your Cloudflare account
   - `CLOUDFLARE_AI_GATEWAY_URL` and `CLOUDFLARE_AI_GATEWAY_TOKEN` set

## Configuration Steps

### 1. Get Your Azure OpenAI API Key

1. Go to [Azure Portal](https://portal.azure.com)
2. Navigate to your **Azure OpenAI** resource
3. Go to **Keys and Endpoint** section
4. Copy one of your API keys (either KEY 1 or KEY 2)
5. Note your **Endpoint URL** (format: `https://<resource-name>.openai.azure.com`)

### 2. Configure Environment Variables

#### For Platform-Level Configuration (All Users)

Add to your `.dev.vars` (local) or set as a secret in Cloudflare Workers (production):

```bash
AZURE_AI_API_KEY="your-azure-openai-api-key-here"
```

#### For User-Level Configuration (BYOK - Bring Your Own Key)

Users can provide their own Azure AI API keys through the platform's secret management system. The environment variable for BYOK is:

```bash
AZURE_AI_API_KEY_BYOK="user-provided-key"
```

**Note:** BYOK keys are stored per-user in encrypted storage, not as environment variables.

### 3. Configure Cloudflare AI Gateway

Azure AI must be routed through Cloudflare AI Gateway. Ensure you have:

```bash
CLOUDFLARE_AI_GATEWAY_URL="https://gateway.ai.cloudflare.com/v1/<account-id>/<gateway-name>/"
CLOUDFLARE_AI_GATEWAY_TOKEN="your-cloudflare-api-token"
```

The AI Gateway will route requests to Azure OpenAI using the provider name `azure-ai`.

### 4. Configure Models in Code

Azure AI models need to be configured in `worker/agents/inferutils/config.ts`. Models must follow the format:

```
azure-ai/<model-name>
```

#### Example Model Configuration

Add Azure AI models to `worker/agents/inferutils/config.types.ts`:

```typescript
AZURE_GPT_4: {
    id: 'azure-ai/gpt-4',
    config: {
        name: 'Azure GPT-4',
        size: ModelSize.LARGE,
        provider: 'azure-ai',
        creditCost: 5, // Adjust based on your pricing
        contextSize: 8192,
    }
},
AZURE_GPT_35_TURBO: {
    id: 'azure-ai/gpt-3.5-turbo',
    config: {
        name: 'Azure GPT-3.5 Turbo',
        size: ModelSize.REGULAR,
        provider: 'azure-ai',
        creditCost: 1.5,
        contextSize: 16384,
    }
},
```

Then update your agent configurations in `worker/agents/inferutils/config.ts` to use Azure models:

```typescript
const PLATFORM_AGENT_CONFIG: AgentConfig = {
    // ... other configs
    blueprint: {
        name: AIModels.AZURE_GPT_4, // Use Azure model
        reasoning_effort: 'high',
        max_tokens: 20000,
        fallbackModel: AIModels.AZURE_GPT_35_TURBO,
        temperature: 1.0,
    },
    // ... rest of configs
};
```

### 5. Verify Provider is Enabled

The system automatically detects Azure AI if `AZURE_AI_API_KEY` is set. You can verify this by checking:

```typescript
// In worker/api/controllers/modelConfig/byokHelper.ts
// Azure AI is included in the platform enabled providers list
const providerList = [
    'anthropic',
    'openai',
    'google-ai-studio',
    'cerebras',
    'groq',
    'azure-ai', // ✅ Azure AI is supported
];
```

## How It Works

### Request Flow

1. **Model Request**: When a model with provider `azure-ai` is requested
2. **API Key Resolution**: System looks for `AZURE_AI_API_KEY` (or user's BYOK key)
3. **Gateway Routing**: Request is routed through Cloudflare AI Gateway with provider override `azure-ai`
4. **Azure OpenAI**: Gateway forwards request to Azure OpenAI using your API key
5. **Response**: Response flows back through the gateway to your application

### Key Components

- **Provider Name**: `azure-ai` (used in model IDs and routing)
- **Environment Variable**: `AZURE_AI_API_KEY` (platform) or `AZURE_AI_API_KEY_BYOK` (user)
- **Gateway Integration**: Routes through Cloudflare AI Gateway at `/azure-ai` endpoint
- **Model Format**: `azure-ai/<model-name>` (e.g., `azure-ai/gpt-4`)

## Troubleshooting

### Issue: Models Not Available

**Solution**: Ensure:
1. `AZURE_AI_API_KEY` is set in environment variables
2. Cloudflare AI Gateway is properly configured
3. Models are added to `config.types.ts` with `azure-ai/` prefix
4. Agent configs reference the Azure models correctly

### Issue: Authentication Errors

**Solution**: 
1. Verify your Azure OpenAI API key is valid
2. Check that your Azure OpenAI resource is active
3. Ensure the model name matches what's deployed in Azure
4. Verify Cloudflare AI Gateway token has proper permissions

### Issue: Gateway Routing Errors

**Solution**:
1. Verify `CLOUDFLARE_AI_GATEWAY_URL` format is correct
2. Ensure `CLOUDFLARE_AI_GATEWAY_TOKEN` is valid
3. Check that Azure AI provider is enabled in your AI Gateway configuration
4. Verify the gateway has access to route to Azure OpenAI

## Additional Notes

- **Model Names**: Azure OpenAI model names may differ from OpenAI's naming. Use the exact model deployment name from your Azure portal.
- **Endpoint Configuration**: The endpoint is automatically handled by Cloudflare AI Gateway when using the `azure-ai` provider.
- **BYOK Support**: Users can provide their own Azure AI keys through the platform's secret management UI.
- **Cost Management**: Azure OpenAI costs are separate from Cloudflare costs. Monitor usage in Azure Portal.

## Example: Complete Setup

```bash
# .dev.vars
CLOUDFLARE_AI_GATEWAY_URL="https://gateway.ai.cloudflare.com/v1/your-account-id/your-gateway-name/"
CLOUDFLARE_AI_GATEWAY_TOKEN="your-cloudflare-api-token"
AZURE_AI_API_KEY="your-azure-openai-api-key"
```

```typescript
// worker/agents/inferutils/config.types.ts
AZURE_GPT_4: {
    id: 'azure-ai/gpt-4',
    config: {
        name: 'Azure GPT-4',
        size: ModelSize.LARGE,
        provider: 'azure-ai',
        creditCost: 5,
        contextSize: 8192,
    }
}
```

```typescript
// worker/agents/inferutils/config.ts
blueprint: {
    name: AIModels.AZURE_GPT_4,
    reasoning_effort: 'high',
    max_tokens: 20000,
    fallbackModel: AIModels.AZURE_GPT_35_TURBO,
}
```

## References

- [Azure OpenAI Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- [Cloudflare AI Gateway Documentation](https://developers.cloudflare.com/ai-gateway/)
- [VibSDK Setup Guide](./setup.md)

