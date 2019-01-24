---
layout: post
title: Bulk loading and partitioning with Cosmos DB
summary: A quick practical walk through of creating a scalable partition strategy for Cosmos DB, bulk loading data and putting it through it's paces, all using .NET Core SDK
---

### Getting the to the *part* of it

If you have used NoSQL based data stores you will no doubt be familiar with partitioning, if not check out [this](https://docs.microsoft.com/en-us/azure/architecture/best-practices/data-partitioning) guide on the topic.

I will be using Cosmos DB for this and there is a specific guide for partitioning which can be found [here](https://docs.microsoft.com/en-us/azure/cosmos-db/partition-data).

The catalyst behind this post was to benchmark some cross partition queries, so choosing the correct partition key is critical here to ensure queries perform well. With Cosmos there is a known issue with cross partition queries using document 'id', more details [here](https://stackoverflow.com/questions/47208981/retrieving-a-document-by-id-is-slow-across-partitions-in-cosmos-db), this issue is being worked on by the product team to make this less of an issue.

The optimum (fastest with least resources) query for a single document is when you know the 'id' and the 'partition key', so what we are going to do now is distribute our data evenly but also be able to figure out the 'partition key' without having to ask Cosmos to search all the partitions.

For the majority of queries we will know the partition key, but for a handful we will need to use a cross partition query. In the case of Cosmos DB and other cloud based data stores, we are charged for the query power (Request Units) and the amount of data stored. So if we are going to do queries that scan large amounts of data we need to do this in the most efficient way, as per the SO post above using 'id' is *NOT* the best option here.

In this example, we are using a super simple device registration store, when we register a new device we create a document and specify and 'id' used by Cosmos and we also duplicate this field to 'uid' in the same doc so we can query by both. Then we have the partition key field which we have called 'device' which is a hash of the actual id, more on that later. So the document looks like this.

```
{
    "id": "47d1fe45-667f-4a8d-9e16-a2caba598172", //Cosmos id
    "uid": "47d1fe45-667f-4a8d-9e16-a2caba598172", //same as id above
    "device": "4FD33DCDE4707D09696B", //hash of id - used as partition key
    "type": "device" //just a ref so we can diff multiple document types in the same collection
}
```

How? Hashing

### Hashtastic

A hash is a function that converts values, commonly used for crypto and checksums, more [here](https://en.wikipedia.org/wiki/List_of_hash_functions). So what if we take some of that crypto tech and use it to create some deterministic values.

The links above are great for partition guidance, however, there is a great talk from Ignite 2018 on this topic and more and is really worth a watch, you can find that [here](https://www.youtube.com/watch?v=rapFud8vu0k).

The code for this can be found [here](https://github.com/msimpsonnz/nosql), but let's walk through the hashing technique we used.

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

### SHA ba dabba doo

So we have some (LOTS) of device id's, these are in a GUID format and don't really have any strong relationship between each other. As per the Ignite video above we want a partition strategy that will provide a good distribution of data, so we have a scenario of 10M device id's but 10M partitions is no good so how would we logically group them, what we can do is break them up in to buckets and shard them into partitions. Great theory but how do we ensure even distribution on seemingly random data? We 'bucket' the data and then hash the device id so it falls into one of the buckets, [this](https://crypto.stackexchange.com/questions/17990/sha256-output-to-0-99-number-range/17994#17994) is a great discussion on the topic.

We first convert the GUID to a ByteArray and then cast it to an integer
```int partitionKey = BitConverter.ToInt32(Guid.NewGuid().ToByteArray(), 0);```

Then we bucket that integer using the modulo operator which is the remainder after division, in this case we are using 256 buckets
```int bucket = partitionKey % 256;```

The we hash
```var partition = sha256.ComputeHash(BitConverter.GetBytes(bucket));```

Once we have the hash we can extract the first 10 characters as a string like this
```
    private static string GetStringFromHash(byte[] hash)
    {
        StringBuilder result = new StringBuilder();
        for (int i = 0; i < 10; i++)
        {
            result.Append(hash[i].ToString("X2"));
        }
        return result.ToString();
    }
```

### X Part Queries

Ok, so now down to actually running a query cross partition, this is being done on Cosmos DB instances with 20K Request Units provisioned.

Query by id, below returns the following:
`Cross Partition Query by Id Duration: 00:00:34.9694223 - RU:51647.73`

```
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

```
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