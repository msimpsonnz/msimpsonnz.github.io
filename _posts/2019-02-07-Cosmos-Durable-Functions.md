---
layout: post
title: Using Durable Functions to bulk load data into Cosmos
summary: A quick practical walk through bulk loading data into Cosmos DB using Durable Functions
---

I've been working with Cosmos DB a fair bit recently, so check out my previous posts around [partitioning](https://msimpson.co.nz/Cosmos-Partition) and [cross partition queries](https://msimpson.co.nz/Cosmos-Cross-Partition).


### Bulk smash!

We have created our Cosmos DB service, database and collection (using a specified partition key), detailed link for this via CLI is [here](https://docs.microsoft.com/en-us/azure/cosmos-db/scripts/create-database-account-collections-cli?toc=%2Fcli%2Fazure%2Ftoc.json#sample-script)

```bash
az Cosmos DB collection create \
    --resource-group $resourceGroupName \
    --collection-name $containerName \
    --name $accountName \
    --db-name $databaseName \
    --partition-key-path /device \
    --throughput 10000
```

Great so now we build our .NET Core app, grab the Cosmos SDK and build a connection in code, I reused most of this from the [Cosmos DB BulkExecutor library for .NET sample](https://github.com/Azure/azure-cosmosdb-bulkexecutor-dotnet-getting-started), which as the name suggests is a great library for importing a lot of data. I was looking for 10M+ documents for this project so needed something that could get that done in a hurry.

### To Console or not to Console?

This code originally started life as a simple console app, [this](https://github.com/msimpsonnz/msft-misc/tree/a3528f4121f15670a69ca10e7cb263ee68286172) is the last commit before I changed tack.

It was nice and all, but it would take a far bit of time to load the documents from my machine over the network.

> Yes! I know I could have used the [Data Migration Tool](https://docs.microsoft.com/en-us/azure/cosmos-db/import-data) but I would not have learnt anything, and that is what this is all about.

So I thought, what would speed this process up?

If I could run my code closer to the database I could reduce the network latency, but also wanted something that could scale out to run in parallel...Durable Functions!!

[Azure Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview) allow you to write stateful functions in a serverless environment. I was fairly sure running a single Function to insert 10M documents would exceed the 10 minute execution [limit](https://docs.microsoft.com/en-us/azure/azure-functions/functions-scale#consumption-plan), so what if I used Durable Functions to batch up my requests and insert 1000 documents at a time, this could scale out and run in parallel.

So we start with a HTTP trigger function that kicks off the process and builds a Durable Orchestrator which is the context the operation will run under.

```csharp
[FunctionName("BulkLoader_HttpStart")]
public static async Task<HttpResponseMessage> HttpStart(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")]HttpRequestMessage req,
    [OrchestrationClient]DurableOrchestrationClient starter,
    ILogger log)
{
    // Function input comes from the request content.
    string instanceId = await starter.StartNewAsync("BulkLoader", null);


    log.LogInformation($"Started orchestration with ID = '{instanceId}'.");


    return starter.CreateCheckStatusResponse(req, instanceId);
}
```

This our Orchestrator function, which looks for a configuration setting on `BatchSize` which is how many batches I want to run, in this case it is 100,000.
The function then creates a `Activity` task for each of those, this will be the actual function that does the work.
The function then waits for all the Activity tasks to complete and returns a result.

```csharp
[FunctionName("BulkLoader")]
public static async Task<bool> RunOrchestrator(
[OrchestrationTrigger] DurableOrchestrationContext context)
{
    int batchSize;
    int.TryParse(Environment.GetEnvironmentVariable("BatchSize"), out batchSize);
    var outputs = new List<bool>();
    var tasks = new Task<bool>[batchSize];
    for (int i = 0; i < batchSize; i++)
    {
        tasks[i] = context.CallActivityAsync<bool>("BulkLoader_Batch", i);
    }

    await Task.WhenAll(tasks);

    return true;
}
```

This is the Activity Task, the function that actually does the work with Cosmos. I use the Bulk Executor library as per above to do the insert of documents.

```csharp
[FunctionName("BulkLoader_Batch")]
public static async Task<bool> Import([ActivityTrigger] int batch, ILogger log)
{
    log.LogInformation($"Prepare documents for batch {batch}");
    string partitionKey = Environment.GetEnvironmentVariable("PartitionKey");
    int docsPerBatch;
    int.TryParse(Environment.GetEnvironmentVariable("DocsPerBatch"), out docsPerBatch);
    List<string> documentsToImportInBatch = CosmosHelper.DocumentBatch(partitionKey, docsPerBatch, batch);
    await BulkImport.BulkImportDocuments(documentsToImportInBatch);
    return true;
}   
```

### Scale to Fail

So this was great, I deployed my Function code and enabled Application Insights so I could see what was going on.
The Function App started scaling out to run the import batches on multiple servers...it scaled out...and out ...and out...
Right about now was when I exceeded the 20K Request Units of Cosmos, so as expected the service started throttling and giving `429` response back to the Functions.

I guess I wasn't expecting this to go so well (and catastrophically bad at the same time) so I didn't have any retry or backoff handling in my application code. Durable Functions kept spinning up new instances and Cosmos kept throttling them, below is a screen shot from App Insights where you can see the outgoing requests match the outgoing dependency failures! This shows `82` servers running my Functions, this peaked at over `100` for a short time and really shows the power and scale of Functions. With that power comes great responsibility, which is non existent in my case!

[<img src="{{ site.baseurl }}/images/2019-02-07-Cosmos-Durable-Functions/peaklivestream.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-02-07-Cosmos-Durable-Functions/peaklivestream.png")

### Try, try, try again

Ok, so I added some retry logic to the Durable Function, which sorted the issue with failed requests. I also tweaked the batch size and number of documents per batch, 1000 batches of 10,000 documents seemed to be the best result, but this was balanced with how much I wanted to scale Cosmos to cope, so there is definitely a balance to find.

This is the retry logic for the new Orchestrator Function

```csharp
[FunctionName("BulkLoader")]
public static async Task<bool> RunOrchestrator(
[OrchestrationTrigger] DurableOrchestrationContext context)
{
    int batchSize;
    int.TryParse(Environment.GetEnvironmentVariable("BatchSize"), out batchSize);

    int firstRetryIntervalVar;
    int.TryParse(Environment.GetEnvironmentVariable("FirstRetrySeconds"), out firstRetryIntervalVar);

    int maxNumberOfAttemptsVar;
    int.TryParse(Environment.GetEnvironmentVariable("MaxNumberOfAttempts"), out maxNumberOfAttemptsVar);

    double backoffCoefficientVar;
    double.TryParse(Environment.GetEnvironmentVariable("BackoffCoefficient"), out backoffCoefficientVar);

    var retryOptions = new RetryOptions(
        firstRetryInterval: TimeSpan.FromSeconds(firstRetryIntervalVar),
        maxNumberOfAttempts: maxNumberOfAttemptsVar);
    retryOptions.BackoffCoefficient = backoffCoefficientVar;

    var outputs = new List<bool>();
    var tasks = new Task<bool>[batchSize];
    for (int i = 0; i < batchSize; i++)
    {
        tasks[i] = context.CallActivityWithRetryAsync<bool>("BulkLoader_Batch", retryOptions, i);
    }

    await Task.WhenAll(tasks);
    return true;
}
```