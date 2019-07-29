---
layout: post
title: Using custom events with AWS Event Bridge
summary: A quick walkthrough of using AWS Event Bridge to publish custom events and get them in S3
tags: [aws, lambda, eventbridge, kinesis, s3, cli]
---

### Event all of the things!

[Amazon Event Bridge](https://aws.amazon.com/eventbridge/) is a new service that allows SaaS applications to publish events to AWS. This is service has been built on [Amazon CloudWatch Events](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html) which is a near real-time stream of events from AWS services. So you could listen into to events that were happening on your AWS accounts, like EC2 Auto Scaling events or CloudTrail actions.

The nice thing with the new Event Bridge model is that we can use the same platform to publish our own custom events and have a common platform, so it's not just for the SaaS providers, you can use it yourself.

So with that in mind I was working with a customer that needed to publish some custom events with the final destination being S3, it would be great if we could filter the "important" events and do this without having this logic in the main application. It was likely to change due to it being a new requirement and we wanted the auth process to be a quick as possible and not have to push changes just to change a reporting function.

So the idea would be to get the application to publish all authentication events to Event Bridge, Kinesis Firehose would then pick these up and get them to S3.

The great thing about AWS is there are so many ways to achieve the same outcome. The above decision came down to wanting something simple and not having to write any more code than we needed to. So yes, we could have used Event Bridge and Lambda to write to S3 but then would have to manage the Lambda code to write the S3 prefix for YYYY/MM/DD/HH which Firehose does by default. We could have written the events to SQS or SNS and used topics to land them in S3, so yep there were other options we considered.

### Building a bridge

I've built a sample for this [here]() but I am going to walk through the main aspects of getting this up and running.

Working backwards, we need the following:
* S3 Bucket to store the events
* Firehose Delivery stream to save to S3
* Event Bridge connected to Firehose

For the S3 Bucket and Firehose I have used the trusty and newly GA'd CDK, to get that wired up with the right permissions in a few lines of code.

This is the CDK Stack, we don't have CloudFormation support for Event Bridge so we will setup that up using the SDK.

```typescript
//Create S3 bucket for the event store
const bucket = new s3.Bucket(this, "demo-evtbridge-custom");

//Create a IAM role for Firehose to use to put events
const s3Role = new iam.Role(this, "demo-evtbridge-s3-role", {
    assumedBy: new ServicePrincipal('firehose.amazonaws.com')
});

//Give the IAM Role write access to bucket
bucket.grantReadWrite(s3Role);

//Create an IAM role for Event Bridge to use to connect to Firehose
const evtRole = new iam.Role(this, "demo-evtbridge-firehose-role", {
    assumedBy: new ServicePrincipal('events.amazonaws.com'),
    roleName: 'demo-evtbridge-firehose-role'
});

//Give the IAM role access to Firehose
evtRole.addToPolicy(new PolicyStatement({
    resources: ['*'],
    actions: ['firehose:PutRecord','firehose:PutRecordBatch'] }));

//Create the Firehose with connection to S3, assign role and config
const deliveryStream = new firehose.CfnDeliveryStream(this, "stream-evtbridge-custom", {
    deliveryStreamName: 'stream-evtbridge-custom',
    deliveryStreamType: 'DirectPut',
    s3DestinationConfiguration: {
    bucketArn: bucket.bucketArn,
    bufferingHints: {
        intervalInSeconds: 300,
        sizeInMBs: 5
    },
    compressionFormat: 'UNCOMPRESSED',
    roleArn: s3Role.roleArn
    }

});
```

So now we need a new Event Bridge - Event Bus, you get a "default" Event Bus out of the gate which is used for CloudWatch Events. So we will create a new one.

```python
accountId = sys.argv[1]
region = sys.argv[2]
evtbridgeBus = sys.argv[3]
evtbridgeRule = 'customRule'
evtbridgeRulePattern = '{\n  "source": [\n    "login.success"\n  ]\n}'
evtTargetArn = f'arn:aws:firehose:{region}:{accountId}:deliverystream/stream-evtbridge-custom'
evtTargetRoleArn = f'arn:aws:iam::{accountId}:role/demo-evtbridge-firehose-role'

createEB = client.create_event_bus(
    Name=evtbridgeBus,
)
print(createEB)

put_rule = client.put_rule(
    Name=evtbridgeRule,
    EventPattern=evtbridgeRulePattern,
    EventBusName=evtbridgeBus
)
print(put_rule)

createTarget = client.put_targets(
    Rule=evtbridgeRule,
    EventBusName=evtbridgeBus,
    Targets=[
        {
            'Id': 'someTargetId',
            'Arn': evtTargetArn,
            'RoleArn': evtTargetRoleArn
        },
    ]
)
```

Done! Now we just need to send some events. I have a dummy Python script that sends a stream of events that have random logon events, we only care about `success` so the Event Bridge rule will pick this up in the pattern match and only send these to Firehose.

```python
event = random.choice(['success', 'failed', 'refresh'])
print(event)

data = {
    "customEvent": {
        "loginEvent": event
    }
}
json_string = json.dumps(data)
print(json_string)

source = f'login.{event}'
print(source)

putEvent = client.put_events(
    Entries=[
        {
            'Source': source,
            'DetailType': 'string',
            'Detail': json_string,
            'EventBusName': 'evtCustomAuth'
        },
    ]
)
print(putEvent)
```

Awesome so we can see from below that we are sending events randomly - success, failed and refresh

[<img src="{{ site.baseurl }}/images/2019-07-27-EventBridge-Custom/client.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-07-27-EventBridge-Custom/client.png")

Now lets look at the S3 output and we can see only the `success` message have come through!

[<img src="{{ site.baseurl }}/images/2019-07-27-EventBridge-Custom/s3.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-07-27-EventBridge-Custom/s3.png")