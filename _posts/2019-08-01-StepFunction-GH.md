---
layout: post
title: Using webhooks with Step Functions
summary: Automating my Github backup solution using Step Functions
tags: [aws, cdk, lambda, step-functions]
---

### Automate all the things!

In my previous [post](https://msimpson.co.nz/Github-CodeCommit/) I worked on a solution to backup my Github repos to CodeCommit using a scheduled Lambda. I wanted to see if I could automate this using Github webhooks and use Step Functions to build the workflow.

I didn't actually need to use Step Functions in this scenario but I wanted to explore how I could use them with API Gateway and trigger Lambda functions.

Creating the Step Function using CDK and giving it the required access is fairly simple
```typescript
    //Create new Step Function Task
    const submitJob = new sfn.Task(this, 'Submit Job', {
        //This task invokes out Lambda function
        task: new tasks.InvokeFunction(lambdaFn),
        resultPath: '$.guid',
    });
    
    //Create the State Machine for the Step Function
    const state = new sfn.StateMachine(this, 'StateMachine', {
      stateMachineName: "gitbackup-sfn",
      //Provide the job definition from above
      definition: submitJob,
      timeout: Duration.minutes(5)
    });

    //Give Step Function IAM Role permission to invoke Lambda
    lambdaFn.grantInvoke(state.role);
    //Give API Gateway IAM Role permission to execute Step Function
    state.grantStartExecution(apiGatewayRole);
```

Then we create a API Gateway mapping template to transform our webhook from GitHub into a Step Function and pass this into out API Gateway.
```typescript
    //Create integration options for API Method
    var integrationOptions :IntegrationOptions = {
      credentialsRole: apiGatewayRole,
      requestTemplates: {
        'application/json': JSON.stringify({
          input: '$util.escapeJavaScript($input.body)',
          stateMachineArn: state.stateMachineArn
        })
      },
      integrationResponses: [
        integrationResponse
      ]
    };

    //Create the Step Function Integration
    const apiGatewayIntegration = new api.AwsIntegration({ 
      service: "states",
      action: "StartExecution",
      integrationHttpMethod: "POST",
      options: integrationOptions,
    });
```

Now my commits are getting updated as soon as a push to GitHub.

[<img src="{{ site.baseurl }}/images/2019-08-01-StepFunction-GH/sfn.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-08-01-StepFunction-GH/sfn.png")

