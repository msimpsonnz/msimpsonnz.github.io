---
layout: post
title: Cross Partition queries with Cosmos DB
summary: A quick practical walk through of cross partition queries and how to do it right
---

### Getting the to the *part* of it

If you have used NoSQL based data stores you will no doubt be familiar with partitioning, if not check out [this](https://docs.microsoft.com/en-us/azure/architecture/best-practices/data-partitioning) guide on the topic.

I will be using Cosmos DB for this and there is a specific guide for partitioning which can be found [here](https://docs.microsoft.com/en-us/azure/cosmos-db/partition-data).

The catalyst behind this post was to benchmark some cross partition queries, so choosing the correct partition key is critical here to ensure queries perform well. With Cosmos there is a known issue with cross partition queries using document 'id', more details [here](https://stackoverflow.com/questions/47208981/retrieving-a-document-by-id-is-slow-across-partitions-in-cosmos-db), this issue is being worked on by the product team to make this less of an issue.

In this example, we are using a super simple device registration store, when we register a new device we create a document and specify and 'id' used by Cosmos and we also duplicate this field to 'uid' in the same doc so we can query by both. Then we have the partition key field which we have called 'device' which is a hash of the actual id, so the document looks like this.

```json
{
    "id": "47d1fe45-667f-4a8d-9e16-a2caba598172", //Cosmos id
    "uid": "47d1fe45-667f-4a8d-9e16-a2caba598172", //same as id above
    "device": "4FD33DCDE4707D09696B", //hash of id - used as partition key
    "type": "device" //just a ref so we can diff multiple document types in the same collection
}
```

### X Part Queries

Ok, so now down to actually running a query cross partition, this is being done on Cosmos DB instances with 20K Request Units provisioned.

Query by id, below returns the following:
`Cross Partition Query by Id Duration: 00:00:34.9694223 - RU:51647.73`

```csharp
    public static async Task RunQueryById(DocumentClient _client, IOptions<CosmosConfig> _cosmosConfig, string queryId, bool crossPartition = false)
    {
        var feedOptions = new FeedOptions
        {
            EnableCrossPartitionQuery = crossPartition,
            MaxDegreeOfParallelism = 256
        };
        var query = _client.CreateDocumentQuery<CosmosDeviceModel>(UriFactory.CreateDocumentCollectionUri(_cosmosConfig.Value.DatabaseName, _cosmosConfig.Value.CollectionName), feedOptions)
            .Where(d => d.Id == queryId)
            .AsDocumentQuery();
        var timer = Stopwatch.StartNew();
        var response = await query.ExecuteNextAsync();
        timer.Stop();
        Console.WriteLine($"Cross Partition Query by Id Duration: {timer.Elapsed} - RU:{response.RequestCharge}");

    }
```

Query by id, below returns the following:
`Cross Partition Query by Property Duration: 00:00:11.1021038 - RU:5.74`

```csharp
    public static async Task RunQueryByProp(DocumentClient _client, IOptions<CosmosConfig> _cosmosConfig, string queryId, bool crossPartition = false)
    {
        var feedOptions = new FeedOptions {
            EnableCrossPartitionQuery = crossPartition,
            MaxDegreeOfParallelism = 256
        };
        var query = _client.CreateDocumentQuery<CosmosDeviceModel>(UriFactory.CreateDocumentCollectionUri(_cosmosConfig.Value.DatabaseName, _cosmosConfig.Value.CollectionName), feedOptions)
            .Where(d => d.Uid == queryId)
            .AsDocumentQuery();
        var timer = Stopwatch.StartNew();
        var response = await query.ExecuteNextAsync();
        timer.Stop();
        Console.WriteLine($"Cross Partition Query by Property Duration: {timer.Elapsed} - RU:{response.RequestCharge}");

    }
```

Wow! So just by using a duplicate of the id field and query by that we can improve the query speed by *x3* and reduce requests required by over 50K!!