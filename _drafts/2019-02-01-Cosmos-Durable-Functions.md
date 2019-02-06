---
layout: post
title: Using Durable Functions to bulk load data into Cosmos
summary: A quick practical walk through bulk loading data into Cosmos DB using Durable Functions
---

I've been working with Cosmos DB a fair bit recently, so check out my previous posts around [partitioning](https://msimpson.co.nz/Cosmos-Partition) and [cross partition queries](https://msimpson.co.nz/Cosmos-Cross-Partition).


### Bulk smash!

We have created our Cosmos DB service, database and collection (using a specified partition key), detailed link for this via CLI is [here](https://docs.microsoft.com/en-us/azure/cosmos-db/scripts/create-database-account-collections-cli?toc=%2Fcli%2Fazure%2Ftoc.json#sample-script)

```
az Cosmos DB collection create \
    --resource-group $resourceGroupName \
    --collection-name $containerName \
    --name $accountName \
    --db-name $databaseName \
    --partition-key-path /device \
    --throughput 1000
```

Great so now we build our .NET Core app, grab the Cosmos SDK and build a connection in code, I reused most of this from the [Cosmos DB BulkExecutor library for .NET sample](https://github.com/Azure/azure-cosmosdb-bulkexecutor-dotnet-getting-started), which as the name suggests is a great library for importing a lot of data. I was looking for 10M+ documents for this project so needed something that could get that done in a hurry.

