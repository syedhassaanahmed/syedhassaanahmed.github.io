---
layout:	post
title:	"Building single source of truth using Serverless and NoSQL"
date:	2018-04-05
---

  We recently worked with a retail customer to help them accelerate their cloud journey from on-prem to Microsoft Azure. As with all our engagements, we learned a good deal of lessons and have some findings to share. My colleague [Dag König](https://channel9.msdn.com/Events/Speakers/dag-knig) and I summarized the outcomes and presented them in [Stockholm Azure Meetup (March 2018)](https://www.meetup.com/Stockholm-Azure-Meetup/events/247951748/). In this post, we’ll dive deeper into how we built an [event-driven](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven) data ingestion pipeline, leveraging Serverless compute and NoSQL databases.

#### Table of Contents
* [The Challenge](https://medium.com/p/bca6c9d45eeb#e250)
* [Data ingestion](https://medium.com/p/bca6c9d45eeb#cd79)
* [Data storage](https://medium.com/p/bca6c9d45eeb#121c)
* [Data distribution](https://medium.com/p/bca6c9d45eeb#e107)
* [Load Testing](https://medium.com/p/bca6c9d45eeb#98dd)
* [Measuring performance](https://medium.com/p/bca6c9d45eeb#af1d)
* [E2E Architecture](https://medium.com/p/bca6c9d45eeb#b6b6)
* [Conclusion](https://medium.com/p/bca6c9d45eeb#f8b5)

### The Challenge
Customer has a large and complicated data repository which is maintained by multiple systems — let’s call it ***Masterdata***. Changes in Masterdata are synced between many systems. Today’s on-prem solution suffers from inconsistencies, lost entity changes, poor performance at peaks.​ Customer wants to build a high scale cloud-hosted *“Single source of truth”* repository, including mechanisms for Masterdata changes to get pushed to 3rd party consumers.​ The solution needs to support an unpredictable load with a maximum of 1500 requests/sec.

We approached the problem by dividing it into these logical concerns;

### Data ingestion
When it comes to large scale data ingestion in Azure, [Event Hubs](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-what-is-event-hubs) provide a data streaming platform, capable of receiving and capturing millions of events per second. However the customer had enterprise high-value messaging requirements such as transactions, ordering and dead-lettering. Luckily we have such a service at our disposal in [Azure Service Bus](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview). We could have easily skipped Service Bus and directly persisted data to the storage, however the [queue-based load leveling](https://docs.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) pattern allows us to decouple ingestion from storage, adds resilience in the form of retries, as well as enables asynchronous processing of ingested data.

Instead of directly exposing Service Bus to the source systems, we wanted a REST API with friendly URL. [Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-overview) allows us to develop event-driven Serverless applications by providing a multitude of [triggers and bindings](https://docs.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings). All functions have exactly one trigger which defines how the function is invoked. In just few lines below, we were able to create an HTTP trigger function which inserts a JSON-serialized product entity to the Service Bus Queue called productsQueue.
```cs
[FunctionName(nameof(HttpIngressFunc))]
[return: ServiceBus("productsQueue", Connection = "SERVICEBUS_CONNECTION")]
public static string Run([HttpTrigger(AuthorizationLevel.Function, "post")]
    dynamic product, ILogger log)
{
    return product.ToString();
}
```

In case the output binding to Service Bus failed, an Exception will be raised from above function. Currently its being handled in client application (not shown in this post but represented by the [*Load generator*](https://medium.com/p/bca6c9d45eeb#b6b6)) with retries using [Polly](https://github.com/App-vNext/Polly). If the issue becomes more prominent in future, we plan to implement [Dead-Lettering](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-queues#application-level-dead-lettering) in above function.

### Data storage
Our next big challenge was to decide where to store the data. From prior experience with RDBMS, customer highlighted the challenges they faced including maintenance of normalized schema, scaling issues, complexity of queries. There was some temptation around Graph databases, however customer didn’t require traversing a network of relationships — something Graph databases are exceptionally good at.

Due to changing nature of the data as consequence of evolving business requirements, we chose a document database. Document databases in Azure can be provisioned as 3rd party —[ MongoDB Atlas](https://docs.atlas.mongodb.com/reference/microsoft-azure/), as well as first-party PaaS — [Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction). Customer’s affinity towards PaaS and the elastic-scale, low-latency capabilities tipped this decision in Cosmos DB’s favor.

Continuing our pipeline from Service Bus, we connected another function which gets triggered when a product appears in the Service Bus Queue and inserts that product to Cosmos DB.
```cs
private static readonly DocumentClient DocumentClient = CreateDocumentClient();

[FunctionName(nameof(CosmosDbIngressFunc))]
public static async Task Run([ServiceBusTrigger("productsQueue", Connection = "SERVICEBUS_CONNECTION")]
    string productJson, ILogger log)
{
    var product = JsonConvert.DeserializeObject<Document>(productJson);
    await UpsertProductAsync(product, log);
}
```

The function above is self-explanatory with a notable exception that [DocumentClient](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.documents.client.documentclient?view=azure-dotnet) avoids [**improper instantiation anti-pattern**](https://docs.microsoft.com/en-us/azure/architecture/antipatterns/improper-instantiation/). Even though DocumentClient implements [IDisposable](https://msdn.microsoft.com/en-us/library/system.idisposable%28v=vs.110%29.aspx) interface, its intended to be instantiated once and reused throughout the application lifetime. The use of static client is further explained by our colleague [Matías Quaranta](https://medium.com/u/502ce9095864) in [this Cookbook](https://medium.com/@Ealsur/azure-cosmos-db-functions-cookbook-static-client-874072aef28e).

Here is how we instantiated the DocumentClient
```cs
private static DocumentClient CreateDocumentClient()
{
    var endpoint = Environment.GetEnvironmentVariable("COSMOSDB_ENDPOINT", EnvironmentVariableTarget.Process);
    var authKey = Environment.GetEnvironmentVariable("COSMOSDB_KEY", EnvironmentVariableTarget.Process);

    return new DocumentClient(new Uri(endpoint), authKey, null, ConsistencyLevel.ConsistentPrefix);
}
```

A noteworthy parameter to DocumentClient constructor is ConsistencyLevel.ConsistentPrefix*. *[Consistent prefix](https://docs.microsoft.com/en-us/azure/cosmos-db/consistency-levels#consistency-levels) guarantees that reads never see out of order writes. If products arrived in order p1, p2, p3, then a client sees either p1 or p1,p2 or p1,p2,p3, but never out of order like p1,p3 or p2,p1,p3 — a reasonable compromise between Strong and Eventual Consistencies.

Looking deeper into how we upsert data into Cosmos DB;
```cs
private static async Task UpsertProductAsync(Document product, ILogger log)
{
    var collectionLink = UriFactory.CreateDocumentCollectionUri("masterdata", "product");

    product.SetPropertyValue("partitionKey", product.GetPropertyValue<string>("productGroupId"));

    var response = await DocumentClient.UpsertDocumentAsync(collectionLink, product);
    log.LogMetric("product_RU", response.RequestCharge);
}
```

In terms of throughput and storage, Cosmos DB collections can be created as *fixed* or *unlimited*. Fixed collections have a maximum limit of 10 GB and 10,000 [RU](https://docs.microsoft.com/en-us/azure/cosmos-db/request-units)/s throughput. For our design to be future-proof, we chose unlimited collections. To create an unlimited collection we must specify a [partition key](https://docs.microsoft.com/en-us/azure/cosmos-db/partition-data) — an important decision that **can’t be changed after a collection is created**. We must pick a property name that has a wide range of values so that data is evenly distributed. For this reason, we explicitly enhanced our data with a new property partitionKey and populated it with productGroupId (assuming this field has a wide range of evenly distributed values)*.*

Note the last statement in above method; Every response from Cosmos DB includes a custom header — x-ms-request-charge which contains RU consumed for the request. The header is also accessible through Cosmos DB .NET SDK as RequestCharge property. We logged this value for telemetry reasons (more on this in [measuring performance](https://medium.com/p/bca6c9d45eeb#af1d) section below).

### Data distribution
Once we had our data ingested and stored, we wanted to get notified as CRUD operations happen on storage. The [**Change Feed**](https://docs.microsoft.com/en-us/azure/cosmos-db/change-feed)** **support in Cosmos DB feels like a match made in heaven for this very scenario —triggering notification when a document is operated on. Here is how we created a change feed trigger function.
```cs
[FunctionName(nameof(ChangeFeedFunc))]
public static Task Run([CosmosDBTrigger(
        databaseName: "masterdata",
        collectionName: "product",
        ConnectionStringSetting = "COSMOSDB_CONNECTION",
        LeaseCollectionName = "leases", 
        CreateLeaseCollectionIfNotExists = true)]
    IReadOnlyList<Document> input, 
    [OrchestrationClient] DurableOrchestrationClient starter, 
    ILogger log)
{
}
```

Next step was to forward these events to tens of consumers through HTTP Webhooks. Customer had a requirement to be able to read consumer URL and HTTP headers from a table, so that consumers can be dynamically added and configured.

[Logic Apps](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-overview) is an extremely productive service, when it comes to building data integrations and B2B communication in the cloud. In-fact in our first attempt, it took less than an hour and 6 [Logic App Actions](https://docs.microsoft.com/en-us/azure/connectors/apis-list) to build the entire workflow of picking changes from change feed, to reading consumer configuration from [Azure Table storage](https://docs.microsoft.com/en-us/azure/cosmos-db/table-storage-overview), to finally dispatching those changes to Webhooks with retry logic. However the [per-action pricing model](https://azure.microsoft.com/en-us/pricing/details/logic-apps/) of Logic Apps meant **400k changes * 6 actions * 50 consumers = 3000$ !!!**

Fortunately, we have another way of defining stateful Serverless workflows in Azure besides Logic Apps — [Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-overview)! This extension to Azure Functions lets us define workflow in code using an ***orchestrator function***. Orchestrator functions can only be triggered via an [OrchestrationTrigger](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-bindings)*. *Best place to start this trigger in our case was from inside change feed function.
```cs
var products = input.Select(x => x.ToString());
return starter.StartNewAsync(nameof(ConsumerEgressFuncs.OrchestrateConsumersFunc), products);
```

Here is the OrchestrationTrigger function;
```cs
[FunctionName(nameof(OrchestrateConsumersFunc))]
public static async Task OrchestrateConsumersFunc([OrchestrationTrigger] DurableOrchestrationContext ctx)
{
    var changedProducts = ctx.GetInput<IEnumerable<string>>();

    var retryOptions = new RetryOptions(firstRetryInterval: TimeSpan.FromSeconds(5),
        maxNumberOfAttempts: 3);

    var consumers = Environment.GetEnvironmentVariable("CONSUMERS", EnvironmentVariableTarget.Process)
        ?.Split('|');

    var parallelTasks = consumers.Select(x => CallSendToConsumerActivityAsync(ctx, retryOptions, x, changedProducts));

    await Task.WhenAll(parallelTasks);
}
```

We obtained the Cosmos DB product json sent earlier from change feed, then read the list of consumers — (*For simplicity we’ve chosen to store consumer HTTP endpoints as a pipe **|** delimited string in Application Settings.* *In reality they reside in Table Storage*). After that we create an [ActivityTrigger](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-bindings#activity-triggers) per consumer. ActivityTrigger enables us to author stateless functions that can be asynchronously called by orchestrator functions. The Azure Functions Scale Controller [automatically scales](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-perf-and-scale#auto-scale) them out by adding VMs (on consumption plan). When we call activity functions we can specify a [**retry policy**](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-error-handling#automatic-retry-on-failure) for error-handling (similar to Polly) as we’ve done above with the object retryOptions.
```cs
public static async Task CallSendToConsumerActivityAsync(DurableOrchestrationContext ctx,
            RetryOptions retryOptions, string consumerUrl, IEnumerable<string> changedProducts)
{
    try
    {
        await ctx.CallActivityWithRetryAsync(nameof(SendToConsumerFunc), retryOptions, (consumerUrl, changedProducts));
    }
    catch
    {
        //TODO: TEMPORARILY MARK THE CONSUMER AS BANNED IN CONSUMERDB
    }
}
```

We’ve deliberately wrapped the creation of ActivityTrigger in another method so that once retries are exhausted, we can temporarily ban problematic consumers. (*In-fact consumer URLs point to an HTTP trigger function which randomly returns* 500 Internal Server Error). As expected, here is what goes on in activity function;
```cs
[FunctionName(nameof(SendToConsumerFunc))]
public static async Task SendToConsumerFunc([ActivityTrigger] (string consumerUrl, IEnumerable<string> changedProducts) consumerData, 
            ILogger log)
{
    foreach (var product in consumerData.changedProducts)
    {
        var content = new StringContent(product, Encoding.UTF8, "application/json");
        await HttpClient.PostAsync(consumerData.consumerUrl, content);
    }
}
```

### Load Testing
Its all fun and games until the inevitable question — **“But, Will it scale?” **The only way to find out is by measuring. Our colleague [Ville Rantala](https://blog.vjrantal.net/) has already done a great job of [**Load testing with Azure Container Instances and wrk**](https://blog.vjrantal.net/2017/08/10/load-testing-with-azure-container-instances-and-wrk/). Leveraging his work, we created a simple bash script to generate load against the HTTP trigger function which was created earlier for data ingestion.
```bash
SCRIPT_URL=https://raw.githubusercontent.com/syedhassaanahmed/azure-event-driven-data-pipeline/master/load-test/products.lua
WRK_OPTIONS="-t1 -c100 -d2m -R1500 --latency"
WRK_IMAGE=syedhassaanahmed/wrk2-with-online-script

az container create -g $RESOURCE_GROUP --name $CONTAINER_NAME --image $WRK_IMAGE \
    -l westeurope --restart-policy Never -e \
    SCRIPT_URL="$SCRIPT_URL" \
    TARGET_URL="$TARGET_URL" \
    WRK_OPTIONS="$WRK_OPTIONS" \
    WRK_HEADER="content-type: application/json"
```

The above script uses containerized version of [wrk2](https://github.com/giltene/wrk2) *(wrk2 is wrk modified to produce a constant throughput load). *WRK\_OPTIONS specified above, make sure the benchmark runs for 2 minutes, using a single thread, keeping 100 HTTP connections open, with a throughput of ~1500 requests per second. In addition to TARGET\_URL the container also takes SCRIPT\_URL as an environment variable, which points to a [LuaJIT](http://luajit.org/) script. Script allows us to generate dynamic HTTP request body;
```lua
wrk.method = "POST"

local productCount = 0

request = function()
  local productGroupId = productCount % 2000 -- there are 2k product groups
  local productText = string.random(5) .. " Ä Ö Å ä ö å" -- test Swedish chars
  local longText = "This is a very very long text intentionally added to exceed One Kilobyte boundary."
  
  body = string.format([[
  {
    "id": "product_%s",
    "productGroupId": "productGroup_%s",
    "productText": "%s",
    "productStatusId": 11,
    "temperatureMax": -18,
    "temperatureMin": -23,
    "expiryDate": "19.02.2018 03:56:29",
    "VERY_LONG_ATTRIBUTE_1": "%s",
    ...
    "VERY_LONG_ATTRIBUTE_10": "%s"
  }
  ]], productCount, productGroupId, productText, 
  longText, ..., longText)
  
  productCount = (productCount + 1) % 100000 -- there are 100k products
  
  return wrk.format(wrk.method, wrk.path, wrk.headers, body)
end
```

### Measuring performance
With trivial efforts, we were able to add [Application Insights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-overview) to all Azure Functions Apps and have a powerful tool for measuring performance of entire flow. Application Insights provides [live metrics streaming](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-live-stream), designed to have minimal latency.

![](/img/1*mrCLR7n8SBHmVXH0nx1zjQ.png)Live metrics stream from Data ingestion Azure Functions AppThrough the use of Live Metrics, we discovered and made useful enhancements to our solution.

* Azure Functions can be [hosted](https://docs.microsoft.com/en-us/azure/azure-functions/functions-scale) in 2 different modes: *Consumption plan* and [*App Service plan*](https://docs.microsoft.com/en-us/azure/app-service/azure-web-sites-web-hosting-plans-in-depth-overview). Consumption plan automatically scales out as necessary to handle load, hence we initially chose it for all function apps. Although the Azure Functions team has made [**significant improvements in auto-scaling of HTTP trigger functions**](https://www.azurefromthetrenches.com/azure-functions-significant-improvements-in-http-trigger-scaling/) on consumption plan, we required even more aggressive ramp up and instead chose to host HTTP function on App Service Plan. (*For 1500 req/sec, four *[*S3*](https://azure.microsoft.com/en-us/pricing/details/app-service/)* instances were chosen*). Consumption plan can be migrated to App Service plan with the following [Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/overview?view=azurermps-5.5.0) snippet.
```powershell
Select-AzureRmSubscription -SubscriptionId "<SUBSCRIPTION_ID>"
Set-AzureRmWebApp -Name "<FUNCTION_APP>" -ResourceGroupName "<RESOURCE_GROUP>" -AppServicePlan "<NEW_APP_SERVICE_PLAN>"
```
* As our colleague [Martin Šimeček](https://deedx.cz/) [pointed out](https://codez.deedx.cz/projects/high-throughput-azure-functions-on-http/), disabling Azure Functions built-in portal logging by removing AzureWebJobsDashboard setting improves the performance!
* In order to achieve higher Service Bus throughput, we created it with [**Express option and partitioning enabled**](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-performance-improvements#express-queues-and-topics).
* In order for telemetry collection to not compromise solution performance, we enabled Application Insights sampling by configuring the Azure Functions host.json file;
```json
{
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 5
      }
    }
  }
}
```
* As mentioned during data storage, we logged Cosmos DB RequestCharge as custom metric — product_RU. The following Application Insights Analytics query allows us to project this value; *(The data is also available in Cosmos DB metrics blade albeit with some delay)*
```sql
customMetrics
| where timestamp > datetime("2018-03-25T17:25:00")
    and name == "product_RU"
| summarize avg(value) by bin(timestamp, 10s)
| render timechart
```
* By default, all Cosmos DB data is indexed. When dealing with large documents, its best to employ a custom [indexing policy](https://docs.microsoft.com/en-us/azure/cosmos-db/indexing-policies) which only indexes select properties. The following policy reduces our RU cost by only indexing productText;
```json
{
    "indexingMode": "consistent",
    "automatic": true,
    "excludedPaths": [
        {
            "path": "/"
        }
    ],
    "includedPaths": [
        {
            "path": "/productText/?",
            "indexes": [
                {
                    "kind": "Hash",
                    "dataType": "String",
                    "precision": -1
                }
            ]
        }
    ]
}
```
* By default Azure Functions use [*Gateway Mode*](https://docs.microsoft.com/en-us/azure/cosmos-db/performance-tips#networking) and HTTPS to connect to Cosmos DB. For best performance we can switch connectivity to *Direct Mode* and TCP by [modifying](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-cosmosdb-v2#hostjson-settings) the host.json file like below;
```json
{
  "extensions": {
    "cosmosDB": {
        "connectionMode": "Direct",
        "protocol": "Tcp"
    }
  }
}
```

### E2E Architecture
With all components of our design in place, we ended up with this overall architecture;

![](/img/1*ljTHA96DW4TkrqSrcaZXsw.png)

### Conclusion
The entire code is available on [GitHub](https://github.com/syedhassaanahmed/azure-event-driven-data-pipeline) and we’ve deliberately kept it as generic as possible, in order for it to be a reusable solution. For single-click repeatable deployments, there is also an [ARM template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-create-first-template) available.
  