---
layout: post
title: Streaming data using AWS Database Migration Service and Kinesis
summary: In this post we look at streaming data from MySQL all the way to S3, using DMS, Kinesis Streams and Kinesis Firehose
tags: [aws, dms, kinesis]
---

### ~~Center for Disease Control~~ ...nope...the other CDC

In a previous [post](https://msimpson.co.nz/MSK-Logging/) we looked a streaming with Kafka, however, another common pattern we see is getting the changes from a SQL database. Here we are going to use Change Data Capture or CDC, which allows us to stream the changes happening on the database. CDC uses log sequences and checkpoints so the changes can be resumed and replayed, but these differ between database engine so best to check what you need to enable this on your database. In this instance we are using MySQL so we need to change the [Binary Log Format](https://dev.mysql.com/doc/refman/8.0/en/binary-log-setting.html) to 'ROW' which will capture all changes at a row level.

For this example, I have used the following:
* Aurora MySQL - source database
* Database Migration Service - this will connect to Aurora and get the changes
* Kinesis Stream - DMS will write the changes to the stream
* Kinesis Firehose - will stream from Kinesis to S3
* S3 - the location for our streamed data

For the actual data I have used the DMS sample data set which can be found on [GitHub](https://github.com/aws-samples/aws-database-migration-samples) and there is a guide [here](https://docs.aws.amazon.com/dms/latest/sbs/CHAP_On-PremOracle2Aurora.Appendix.SampleDatabase.html) on how to generate some activity to test out streaming changes.
I've spun up an Aurora instance and use the scripts to build a database instance and then taken a snapshot. The code below uses that snapshot to provision a new Aurora instance, so if you are following along you will need to do the same.

For the infrastructure piece, I've used the [CDK](https://aws.amazon.com/cdk/) again! I really love this as it lets me build a repeatable solution that I can share, but cuts the time down of writing CloudFormation or pasting a whole lot of instructions and screen shots from the console.

However, saying that, there are still something things that don't belong in stacks just yet. We need some IAM roles for the DMS, so we set them up manually then deploy via the CDK.

```bash
git clone https://github.com/msimpsonnz/aws-misc.git
cd aws-misc/dms-stream

aws iam create-role --role-name dms-vpc-role --assume-role-policy-document file://dmsAssumeRolePolicyDocument.json

aws iam attach-role-policy --role-name dms-vpc-role --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole

cd cdk
cdk deploy
```

So now lets walk through the CDK stack components to understand how this has been built out.

First we need an Aurora instance running a copy of our snapshot, here we create new parameter group with the Binary Log Format turned on.

```typescript
const auroraParam = new ClusterParameterGroup(this, 'dms-aurora-mysql5.7', {
    family: 'aurora-mysql5.7',
    parameters: {
    binlog_format: 'ROW'
    }
});

const auroraCluster = new CfnDBCluster(this, 'tickets-db', {
    engine: 'aurora-mysql',
    engineVersion: '5.7.12',
    snapshotIdentifier: 'tickets-mysql',
    dbClusterParameterGroupName: auroraParam.parameterGroupName
});
```

Then we create the DMS instance, add Kinesis Stream as the destination and the necessary IAM roles needed

```typescript
const dmsRep = new dms.CfnReplicationInstance(this, 'dms-replication', {
    replicationInstanceClass: 'dms.t2.medium',
    engineVersion: '3.3.0'
});

const stream = new kinesis.Stream(this, 'dms-stream');

const streamWriterRole = new Role(this, 'dms-stream-role', {
    assumedBy: new ServicePrincipal('dms.amazonaws.com')
});

streamWriterRole.addToPolicy(new PolicyStatement({
    resources: [stream.streamArn],
    actions: [
    'kinesis:DescribeStream',
    'kinesis:PutRecord',
    'kinesis:PutRecords'
    ]
}));

const source = new CfnEndpoint(this, 'dms-source', {
    endpointType: 'source',
    engineName: 'aurora',
    username: 'admin',
    password: 'Password1',
    serverName: auroraCluster.attrEndpointAddress,
    port: 3306
});

const target = new CfnEndpoint(this, 'dms-target', {
    endpointType: 'target',
    engineName: 'kinesis',
    kinesisSettings: {
    messageFormat: 'JSON',
    streamArn: stream.streamArn,
    serviceAccessRoleArn: streamWriterRole.roleArn
    }
});
```

We then need to create the DMS replication task, this take the source, destination and a table mapping of the records that we are going to extract.

```typescript
var dmsTableMappings = {
    "rules": [
    {
        "rule-type": "selection",
        "rule-id": "1",
        "rule-name": "1",
        "object-locator": {
        "schema-name": "dms_sample",
        "table-name": "ticket_purchase_hist"
        },
        "rule-action": "include"
    }
    ]
}

new CfnReplicationTask(this, 'dms-stream-repTask', {
    replicationInstanceArn: dmsRep.ref,
    migrationType: 'full-load-and-cdc',
    sourceEndpointArn: source.ref,
    targetEndpointArn: target.ref,
    tableMappings: JSON.stringify(dmsTableMappings)
});
```

Finally we create an S3 bucket for the extracted records and a Kinesis Firehose to get records from the Kinesis Stream and send these to S3

```typescript
const s3bucket = new s3.Bucket(this, 'dms-stream-bucket');
s3bucket.grantReadWrite(firehoseRole);

const firehose = new CfnDeliveryStream(this, 'dms-stream-firehose', {
    deliveryStreamType: 'KinesisStreamAsSource',
    kinesisStreamSourceConfiguration: {
    kinesisStreamArn: stream.streamArn,
    roleArn: firehoseRole.roleArn
    },
    s3DestinationConfiguration: {
    bucketArn: s3bucket.bucketArn,
    bufferingHints: {
        intervalInSeconds: 300,
        sizeInMBs: 5
    },
    compressionFormat: 'GZIP',
    roleArn: firehoseRole.roleArn
    }
});
```

Once the DMS task kicks off the table makes the initial load into S3 and we can see this has been complete and it is ready for replicated data.

[<img src="{{ site.baseurl }}/images/2019-09-13-DMS-Stream/dms-cdc.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-09-13-DMS-Stream/dms-cdc.png")

We then kick off our stored procedure on the MySQL source to generate some DB activity and once these have been processed through the streams we can run a S3 Select to see the updated records landing in S3.

[<img src="{{ site.baseurl }}/images/2019-09-13-DMS-Stream/s3-select.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-09-13-DMS-Stream/s3-select.png")