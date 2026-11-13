<!--
---
name: Azure Functions C# Event Hubs Trigger using Azure Developer CLI
description: This repository contains an Azure Functions Event Hubs trigger quickstart written in C# and deployed to Azure Functions Flex Consumption using the Azure Developer CLI (azd). The sample demonstrates real-time news streaming with sentiment analysis and engagement tracking.
page_type: sample
products:
- azure-functions
- azure-event-hubs
- azure
- entra-id
urlFragment: functions-quickstart-dotnet-azd-eventhub
languages:
- csharp
- bicep
- azdeveloper
---
-->

# Azure Functions with Event Hubs Trigger

An Azure Functions QuickStart project that demonstrates how to use an Event Hubs Trigger with Azure Developer CLI (azd) for quick and easy deployment. This sample showcases a real-time news streaming system with automated content generation and intelligent processing.

## Architecture

This architecture shows how the Azure Function processes news articles through Event Hubs in real-time. The key components include:

- **News Generator (Timer Trigger)**: Automatically generates realistic news articles every 10 seconds and streams them to Event Hubs
- **Azure Event Hubs**: Scalable messaging service that handles high-throughput news streaming with 32 partitions
- **News Processor (Event Hub Trigger)**: Executes automatically when news articles arrive, performing sentiment analysis and engagement tracking
- **Azure Monitor**: Provides logging and metrics for function execution and news analytics
- **Downstream Integration**: Optional integration with other services for search indexing, push notifications, or analytics

This serverless architecture enables highly scalable, event-driven news processing with built-in resiliency and automatic scaling.

## Top Use Cases

1. **Real-time News Processing Pipeline**: Automatically process news articles as they're generated or updated. Perfect for scenarios where you need to analyze sentiment, detect viral content, or trigger notifications when new articles arrive without polling.

2. **Event-Driven Content Management**: Build event-driven architectures where new content automatically triggers downstream business logic. Ideal for content moderation workflows, search index updates, or social media distribution systems.

## Features

- Event Hubs Trigger with high-throughput news streaming (18-48 articles/minute)
- Azure Functions Flex Consumption plan for automatic scaling
- Real-time sentiment and engagement tracking
- Optional VNet integration with private endpoints for enhanced security
- Azure Developer CLI (azd) integration for easy deployment
- Infrastructure as Code using Bicep templates with Azure Verified Modules
- Comprehensive monitoring with Application Insights and OpenTelemetry
- Managed Identity authentication for secure, passwordless access

## Getting Started

### Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0)
- [Azure Functions Core Tools 4.16.0 or later](https://docs.microsoft.com/azure/azure-functions/functions-run-local#install-the-azure-functions-core-tools)
- [Azure Developer CLI (azd)](https://docs.microsoft.com/azure/developer/azure-developer-cli/install-azd)
- [Azurite](https://github.com/Azure/Azurite) for local development
- An Azure subscription

### Quickstart

1. Clone this repository

   ```bash
   git clone https://github.com/Azure-Samples/functions-quickstart-dotnet-azd-eventhub.git
   cd functions-quickstart-dotnet-azd-eventhub
   ```

2. Make sure to run this before calling azd to provision resources so azd can run scripts required to setup permissions

   Mac/Linux:

   ```bash
   chmod +x ./infra/scripts/*.sh 
   ```
  
   Windows:

   ```powershell
   Set-ExecutionPolicy RemoteSigned
   ```

3. Configure VNet settings (optional)

   You can choose whether to enable VNet integration:

   **For simple deployment without VNet (public endpoints):**

   ```bash
   azd env set VNET_ENABLED false
   ```

   **For secure deployment with VNet (private endpoints):**

   ```bash
   azd env set VNET_ENABLED true
   ```

   > **Note:** If you don't set `VNET_ENABLED`, the deployment will ask you to make an explicit choice.

4. Provision Azure resources using azd

   ```bash
   azd provision
   ```

   This will create all necessary Azure resources including:
   - Azure Event Hubs namespace and hub
   - Azure Function App (Flex Consumption)
   - Application Insights for monitoring
   - Storage Account for function app
   - Virtual Network with private endpoints (if `VNET_ENABLED=true`)
   - Other supporting resources
   - local.settings.json for local development with Azure Functions Core Tools, which should look like this:

   ```json
   {
     "IsEncrypted": false,
     "Values": {
       "AzureWebJobsStorage": "UseDevelopmentStorage=true",
       "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
       "EventHubConnection__fullyQualifiedNamespace": "your-eventhubs-namespace.servicebus.windows.net"
     }
   }
   ```

   The `azd` command automatically sets up the required connection strings and application settings.

5. Start the function locally

   ```bash
   cd src
   func start
   ```

   Or use VS Code to run the project with the built-in Azure Functions extension by pressing F5.

6. Test the function locally by watching the automatic news generation

   The News Generator will automatically start creating articles every 10 seconds. You should see console output like:

   ```
   [2024-11-10T10:30:15.123Z] ✅ HIGH-THROUGHPUT: Successfully generated 5 news articles in ~10 seconds
   [2024-11-10T10:30:15.145Z] ✅ Successfully processed article NEWS-20241110-A1B2C3D4 - 'Breaking: Major Discovery in Renewable Energy Technology' by Sarah Johnson
   [2024-11-10T10:30:15.147Z] 🔥 Viral article: NEWS-20241110-E5F6G7H8 - 8,547 views
   [2024-11-10T10:30:15.149Z] 📊 NEWS BATCH SUMMARY: 5 articles | Total Views: 18,432 | Avg Views: 3,686 | Avg Sentiment: 0.34 | Status: [Published: 3, Featured: 2]
   ```

7. Deploy to Azure

   ```bash
   azd up
   ```

   This will build your function app and deploy it to Azure. The deployment process:
   - Checks for any bicep changes using `azd provision`
   - Builds the .NET project using `azd package`
   - Publishes the function app using `azd deploy`
   - Updates application settings in Azure

8. Test the deployed function by monitoring the logs in Azure Portal:
   - Navigate to your Function App in the Azure Portal
   - Go to Functions → NewsGenerator or EventHubsTrigger
   - Check the Monitor tab to verify both functions are working
   - Use Application Insights Live Metrics to see real-time news processing

## Understanding the Code

This sample contains two functions that work together:

### News Generator (Timer Trigger)

Runs every 10 seconds and generates 3-8 realistic news articles, then sends them to Event Hubs. The key configuration:

- **Timer**: `*/10 * * * * *` (every 10 seconds)
- **Output**: Event Hubs output binding to "news" hub
- **Articles**: Realistic content with authors, sources, categories

### News Processor (Event Hubs Trigger)

Triggered automatically when articles arrive in Event Hubs. Performs sentiment and engagement tracking. The key environment variable that configures its behavior is:

- `EventHubConnection__fullyQualifiedNamespace`: The Event Hubs namespace endpoint

These are automatically set up by azd during deployment for both local and cloud environments.

Here's the core implementation of the Event Hubs trigger function:

```csharp
[Function(nameof(EventHubsTrigger))]
public async Task Run([EventHubTrigger("news", Connection = "EventHubConnection")] EventData[] input)
{
    var processedArticles = new List<NewsArticle>();

    foreach (var message in input)
    {
        var article = ParseNewsArticleEvent(message.EventBody.ToString());
        if (article != null)
        {
            processedArticles.Add(article);
        }
    }

    if (processedArticles.Count > 0)
    {
        await _newsService.ProcessNewsArticles(processedArticles);
    }
}
```

### Article data

The generator sends JSON with `articleId`, `title`, `content`, `author`, `source`, `category`, `publishedDate`, `viewCount`, `sentimentScore`, `status`, and `tags`. The processor requires the ID, title, content, author, source, and category to be nonempty and `viewCount` to be nonnegative. `status` is a number: 0 (Draft), 1 (Published), 2 (Featured), or 3 (Archived). Generated view counts and sentiment scores are simulated.

## Monitoring and Logs

The Functions host uses OpenTelemetry. The worker exports OpenTelemetry to Application Insights only when `APPLICATIONINSIGHTS_CONNECTION_STRING` is nonblank. In Azure, the exporter uses the function app's managed identity through the `AzureMonitorExporter__credential` and `AzureMonitorExporter__clientId` app settings.

In Application Insights Logs, find generation and successful processing with:

```kusto
traces
| where message contains "Successfully generated"
    or message contains "Successfully processed article"
| project timestamp, message
| order by timestamp desc
```

Generation alone does not confirm delivery; use the article ID in a `Successfully processed article` log to verify consumption.

## Project Structure

- [`src/`](src/): .NET isolated functions and host configuration
- [`infra/`](infra/): Bicep and deployment scripts
- [`azure.yaml`](azure.yaml): Azure Developer CLI configuration and hooks

## Networking and VNet Integration

When `VNET_ENABLED=true`, the deployment creates separate subnets for the function app, Storage private endpoints, and Event Hubs private endpoints. Private DNS enables VNet access to Event Hubs and Storage Blob and Table; Storage public network access is disabled. The function app uses managed identity to access these services.

Bicep initially disables Event Hubs public network access, but the `azd` postprovision hook enables it with a default-deny rule allowing the deploying client's public IP for local development. On Windows, the hook skips this update if that IP is already listed. Traffic to other services may use public endpoints. Review this exception before production use.

## Clean Up Resources

When you're done testing the sample, you can delete all Azure resources to avoid incurring charges:

```bash
azd down
```

This will:

- Delete all Azure resources created by `azd provision`
- Remove the resource group
- Clean up the local environment configuration

> **Note:** This action is irreversible. Make sure you no longer need the resources before running this command.

## Resources

- [Azure Functions Documentation](https://docs.microsoft.com/azure/azure-functions/)
- [Azure Event Hubs Documentation](https://docs.microsoft.com/azure/event-hubs/)
- [Azure Developer CLI Documentation](https://docs.microsoft.com/azure/developer/azure-developer-cli/)
