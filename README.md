# Store Azure OpenAI Secrets in KeyVault for Microsoft Fabric Notebooks
# Version : Loy Sikdar
Microsoft Fabric does not support Entra ID authentication to Azure OpenAI resources within a Fabric Notebook. Users must use key-based authentication. To do so securely, Azure OpenAI keys must be stored in Azure KeyVault and accessed using the Service Principal for the current user. Eventually this will include the Fabric Workspace Identity.

This repository deploys an Azure KeyVault via [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/) for use with Microsoft Fabric Notebooks and Workspace Identity. The template creates an Azure OpenAI account with the latest models and stores its secrets in the KeyVault for secure access from Fabric Notebooks.

## Features

- 🔐 **Azure KeyVault** with proper access policies for Fabric Workspace Identity
- 🤖 **Azure OpenAI** with GPT-4.1-mini and Text-Embedding-3-Large model deployments  
- 🔑 **Automatic secret storage** of OpenAI endpoint and API key in KeyVault
- 🏷️ **Resource tagging** with owner, environment, and workspace information
- 🔒 **Role-based access** for User and Fabric workspace to OpenAI
- 🔍 **Automatic workspace discovery** during deployment
- 🚀 **One-command deployment** using Azure Developer CLI
- 📋 **Pre-configured access policies** for seamless Fabric integration

## Prerequisites

1. [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)
2. [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
3. An Azure subscription with permissions to create resources (e.g. Subscription OWNER)
4. A Microsoft Fabric workspace **with Workspace Identity enabled**

## Quick Start

### 1. Initialize the Project

```bash
azd init .
```

When prompted:

- **Enter a new environment name** (e.g., `my-fabric-kv`) - This will be used as your resource group name after "rg-" prefix

### 2. Deploy Infrastructure

```bash
azd provision
```

During deployment, you'll be prompted to:

- **Enter your Fabric workspace name** - The deployment will automatically look up the workspace Service Principal
- **Choose your Azure location** - Select the region for your resources. Ideally the same region as your Fabric Capacity.

The deployment will:

- Automatically discover your user information for resource tagging  
- Look up your Fabric workspace Service Principal
- Create a resource group named "rg-{your-environment-name}"
- Deploy KeyVault and OpenAI resources with proper access policies
- Deploy Azure KeyVault with proper access policies
- Deploy Azure OpenAI with latest GPT-4.1-mini and Text-Embedding-3-Large models
- Configure role-based access for you and your Fabric workspace
- Store OpenAI secrets in KeyVault

## Using Deployment Outputs in Microsoft Fabric Notebooks

After deployment, your template outputs all the configuration values you need to connect to Azure OpenAI from your Fabric Notebooks. These outputs are displayed automatically at the end of deployment. If you need to see them again, simply run `azd provision` - it will skip deployment and show the output values.

### 📋 Available Output Values

The deployment provides these ready-to-use values:

#### Primary deployment outputs you'll use in Fabric Notebooks

- **`KEYVAULT_URI`**: KeyVault endpoint (e.g., `"https://kv-my-keyvault.vault.azure.net/"`)
- **`KEYVAULT_OPENAI_ENDPOINT`**: Secret name for OpenAI endpoint (e.g., `"openai-endpoint"`)
- **`KEYVAULT_OPENAI_API_KEY`**: Secret name for API key (e.g., `"openai-api-key"`)
- **`OPENAI_GPT_MODEL`**: GPT model deployment name (e.g., `"gpt-4.1-mini"`)
- **`OPENAI_EMBEDDING_MODEL`**: Embedding model name (e.g., `"text-embedding-3-large"`)

#### Additional outputs for reference

- **`KEYVAULT_NAME`**: KeyVault name (e.g., `"kv-my-keyvault"`)
- **`OPENAI_NAME`**: OpenAI service name (e.g., `"cog-myopenaiaccount"`)
- **`LOCATION`**: Deployment region
- **`TENANT_ID`**: Azure tenant ID
- **`RESOURCE_GROUP`**: Resource group name

### 🚀 Copy-Paste Ready Notebook Code

Use these output values directly in your Fabric Notebooks:

#### Method 1: Using KeyVault Secrets (Recommended for API Key Authentication)

```python
# Packages
%pip install openai

# Imports
from notebookutils import mssparkutils
from openai.lib.azure import AsyncAzureOpenAI

# Variables, copy these directly from azd output in the terminal
KEYVAULT_URI="https://kv-my-keyvault.vault.azure.net/"
KEYVAULT_OPENAI_ENDPOINT="openai-endpoint"
KEYVAULT_OPENAI_API_KEY="openai-api-key"
OPENAI_GPT_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-large"

# Get the actual OpenAI endpoint and key
OPENAI_ENDPOINT=mssparkutils.credentials.getSecret(KEYVAULT_URI, KEYVAULT_OPENAI_ENDPOINT)
OPENAI_KEY=mssparkutils.credentials.getSecret(KEYVAULT_URI, KEYVAULT_OPENAI_API_KEY)

OPENAI_API_VERSION="2024-12-01-preview"
OPENAI_EMBEDDING_DIMENSIONS=512


# Initialize Azure OpenAI client using keys from KeyVault
OPENAI_CLIENT = AsyncAzureOpenAI(
    api_version=OPENAI_API_VERSION,
    azure_endpoint=OPENAI_ENDPOINT,
    api_key=OPENAI_KEY
)

# Define function to generate embeddings for vector search
async def generate_embeddings(text):
    
    response = await OPENAI_CLIENT.embeddings.create(
        input = text, 
        model = OPENAI_EMBEDDING_MODEL,
        dimensions = OPENAI_EMBEDDING_DIMENSIONS)
    
    embeddings = response.model_dump()
    return embeddings['data'][0]['embedding']

# Generate Embeddings from Azure OpenAI in a Fabric Notebook
search_text = "Hello from Fabric Notebooks!"
embeddings = await generate_embeddings(search_text.strip())
print(embeddings)
```

### � Retrieving Output Values Anytime

To see your deployment output values again:

```bash
# Run provision again - it will show the outputs without redeploying
azd provision
```

You can also view them using:

```bash
# Show all environment variables
azd env get-values

# Show only the key outputs
azd env get-values | Select-String "KEYVAULT_URI|KEYVAULT_OPENAI_ENDPOINT|KEYVAULT_OPENAI_API_KEY|OPENAI_GPT_MODEL|OPENAI_EMBEDDING_MODEL"
```

### �💡 Key Benefits of Using Deployment Outputs

1. **No Hard-Coding**: Use the exact model deployment names from your infrastructure
2. **Environment Consistency**: Same code works across dev/test/prod environments
3. **Easy Updates**: Change model versions in Bicep, redeploy, and use new output values
4. **Secure Access**: KeyVault integration with Fabric Workspace Identity

## Deployed Models

The template deploys the latest OpenAI models:

- **GPT-4.1-mini**: Latest conversational AI model optimized for speed and efficiency
- **Text-Embedding-3-Large**: Advanced embeddings model for semantic search and similarity

## Resource Tagging

All deployed resources are automatically tagged with:

- **Owner**: Your Azure user principal name (email)
- **Environment**: The azd environment name you specified
- **WorkspaceName**: Your Fabric workspace name (if provided)
- **ManagedBy**: "azd" (indicates deployment via Azure Developer CLI)

These tags help with resource management, cost tracking, and governance.

## Clean Up

To remove all deployed resources:

```bash
azd down
```

## Security Notes

- **Role-based access**: Fabric workspace has direct OpenAI access via managed identity (no API keys needed)
- **KeyVault access policies**: Configured for least privilege access
- **OpenAI API keys**: Stored securely in KeyVault
- **Resource tagging**: All resources tagged with owner and workspace information
- **Soft delete**: Enabled for KeyVault recovery. Use `azd down --purge --force` for hard delete
- **Principle of least privilege**: Each component has only the minimum required permissions

### Common Issues

**Workspace not found during deployment:**

- Ensure you have the correct permissions to list Enterprise Applications
- Verify the workspace name is spelled correctly
- You can skip workspace setup and set the Object ID manually later

**OpenAI models not available:**

- GPT-4.1-mini and Text-Embedding-3-Large may not be available in all regions
- Check Azure OpenAI model availability in your chosen region
- You can update the models in `infra/modules/openai.bicep` if needed

**Role assignment failures:**

- Ensure you have sufficient permissions in the subscription. You must be OWNER or have RBAC permissions.
- Verify the Fabric workspace Service Principal exists
- Check that the workspace is properly configured in Fabric

### Getting Help

**Note: This is best-effort supported. Not an officially supported product.**

For additional help:

- Check the [Azure Developer CLI documentation](https://learn.microsoft.com/azure/developer/azure-developer-cli/)
- Review [Microsoft Fabric documentation](https://learn.microsoft.com/fabric/)
- Open an issue in this repository

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
