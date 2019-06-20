---
layout: post
title: Getting API Gateway to post direct to SQS
summary: In this post we refactor our CDK app to have API Gateway post directly to SQS and remove the need for Lambda to do this.
tags: [aws, cdk, api-gateway, sqs]
---

### Do more with less

In the [last post] we looked at using CDK to develop a serverless pipeline to deploy code. So now it's time to iterate and see if we can improve things.

I remember seeing at post from [Richard Boyd] on [Mastering API Gateway in 105 easy steps] which is a great read on how powerful API Gateway can be integrating with a laundry list of AWS services out fo the box.

So I figured we could drop one of my Lambda functions and get API Gateway to post direct to SQS. I thought this was going to be simple, but it turns out API Gateway is a bit of a beast and requires some request/response transforms and mappings, but we got there in the end and thought I would share.

This is what the main CDK file looks like now with regards to API Gateway and SQS.

```
    //Create an IAM Role for API Gateway to assume
    const apiGatewayRole = new iam.Role(sharedStack, 'ApiGatewayRole', {
    assumedBy: new ServicePrincipal('apigateway.amazonaws.com')
    });

    //Create an empty response model for API Gateway
    var model :EmptyModel = {
    modelId: "Empty"
    };

    //Create a method response for API Gateway using the empty model
    var methodResponse :MethodResponse = {
    statusCode: '200',
    responseModels: {'application/json': model}
    };

    //Add the method options with method response to use in API Method
    var methodOptions :MethodOptions = {
    methodResponses: [
        methodResponse
    ]
    };

    //Create intergration response for SQS
    var integrationResponse :IntegrationResponse = {
    statusCode: '200'
    };

    //Create integration options for API Method
    var integrationOptions :IntegrationOptions = {
    credentialsRole: apiGatewayRole,
    requestParameters: {
        'integration.request.header.Content-Type': "'application/x-www-form-urlencoded'"
    },
    requestTemplates: {
        'application/json': 'Action=SendMessage&QueueUrl=$util.urlEncode("' + sqsQueue.queueUrl + '")&MessageBody=$util.urlEncode($input.body)'
    },
    integrationResponses: [
        integrationResponse
    ]
    };

    //Create the SQS Integration
    const apiGatewayIntegration = new apigw.AwsIntegration({ 
    service: "sqs",
    path: sharedStack.env.account + '/' + sqsQueue.queueName,
    integrationHttpMethod: "POST",
    options: integrationOptions,
    });

    //Create the API Gateway
    const apiGateway = new apigw.RestApi(sharedStack, "Endpoint");

    //Create a API Gateway Resource
    const msg = apiGateway.root.addResource('msg');

    //Create a Resource Method
    msg.addMethod('POST', apiGatewayIntegration, methodOptions);

    //Grant API GW IAM Role access to post to SQS
    sqsQueue.grantSendMessages(apiGatewayRole);
```

Check out the full source [here]

Now I can remove the Lambda function, pipeline and most importantly save some $$$

[last post]: https://msimpson.co.nz/AWS-CDK
[Richard Boyd]: https://twitter.com/rchrdbyd
[Mastering API Gateway in 105 easy steps]: https://rboyd.dev/e41a775d-3dd6-4a9f-ab45-b01f3bddab83
[here]: https://github.com/msimpsonnz/cdk-ci-cd