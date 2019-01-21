---
layout: post
title: Bulk loading and partitioning with CosmosDB
summary: A quick practical walk through of creating a scalable partition strategy for Cosmos, bulk loading data and putting it through it's paces, all using .NET Core SDK
---

### Getting the to the *part* of it

If you have used NoSQL based data stores you will no doubt be familiar with partitioning, if not check out [this](https://docs.microsoft.com/en-us/azure/architecture/best-practices/data-partitioning) guide on the topic.

I will be using CosmosDB for this and there is a specific guide for partitioning which can be found [here](https://docs.microsoft.com/en-us/azure/cosmos-db/partition-data).

The catalyst behind this post was to benchmark some cross partition queries, so choosing the correct partition key is critical here to ensure queries perform well. With Cosmos there is a known issue with cross partition queries using document 'id', more details [here](https://stackoverflow.com/questions/47208981/retrieving-a-document-by-id-is-slow-across-partitions-in-cosmos-db), this issue is being worked on by the product team to make this less of an issue.

In this example, we are using a super simple device registration store, when we register a new device we create a document and specify and 'id' used by Cosmos and we also duplicate this field to 'uid' in the same doc so we can query by both. Then we have the partition key field which we have called 'device' which is a hash of the actual id, more on that later. So the document looks like this.

```
{
    "id": "47d1fe45-667f-4a8d-9e16-a2caba598172", //Cosmos id
    "uid": "47d1fe45-667f-4a8d-9e16-a2caba598172", //same as id above
    "device": "4FD33DCDE4707D09696B", //hash of id - used as partition key
    "type": "device" //just a ref so we can diff multiple document types in the same collection
}
```

The optimum (fastest with least resources) query for a single document is when you know the 'id' and the 'partition key', so what we are going to do now is distribute our data evenly but also be able to figure out the 'partition key' without having to ask Cosmos to search all the partitions.

How? Hashing

### Hashtastic

A hash is a function that converts values, commonly used for crypto and checksums, more [here](https://en.wikipedia.org/wiki/List_of_hash_functions). So what if we take some of that crypto tech and use it to create some deterministic values. 

The links above are great for partition guidance, 