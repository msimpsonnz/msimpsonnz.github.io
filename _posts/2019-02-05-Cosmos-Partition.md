---
layout: post
title: Partitioning with Cosmos DB
summary: A quick practical walk through of creating a scalable partition strategy for Cosmos DB
---

#### *Disclaimer*
Partitioning is *HARD*!!
The below was a around a specific use case and was interesting so thought I would share.
Your requirements might not match, for example, we were unlikely to ever exceed the 10GB limit of a single partition in Cosmos. For example, average document size was 6Kb, this would allow of for ~1.667M documents per partition and for example with 256 partitions this would give us ~426M documents! This is 10x what we are designing for so we are good...at this stage of our architecture.

Check out my previous [article](https://msimpson.co.nz/Cosmos-Cross-Partition) around cross partition queries for more details.


### Getting the to the *part* of it ... *Part II*

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

Now we can store the document with a unique partition key and also use the same function to compute the partition on the fly and build this into subsequent queries, making them as qucik as they can be!