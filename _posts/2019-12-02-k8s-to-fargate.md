---
layout: post
title: Joining Squawk Squad as a volunteer to help with cloud platforms
summary: In this post we take a look at how I helped save cost by moving a NestJS API running in k8s to a task definition in AWS Fargate
tags: [aws, fargate]
---

## Senior Volunteer Person (VP) for Technology Squawking

Back in October 2019 I was asked if I wanted to help out with a social enterprise [Squawk Squad](https://squawksquad.co.nz), they were having some challenges with their cloud deployment and wanted some advice.

I was looking for more than a side project that could have some impact and not just me hacking around in my spare time.

This was something I could help with outside of work but use my background to help make a difference. I have done volunteering in the past but it has always felt like they were teaching me to do something rather than me at my best.

> Squawk Squad is a social enterprise that aims to connect and engage a country in the protection and growth of our native bird life.

> Squawk Squad connects people with sanctuaries via a web-app that gives them the ability to collectively fund sensor-connected traps in aid of sanctuary projects. The funders can see where the traps are deployed in the sanctuary and are notified via email in real-time when their trap activates. This indicates the positive impact that their investment is having on native birdlife. Funders also receive a “Squawker profile” which shows where they are on the nation-wide leaderboard - your points depend on how many pests you have trapped! This gives you an exact measure of the difference you are making towards our native birdlife.

I spoke to the team and their mission really inspired me to join up and help out.

## Getting to know you

So I spent some time understanding the code base and deployment platform.

At it's core the app has a static frontend written in React, backend written with [NestJS](https://nestjs.com) and data stored in PostgreSQL.

The data from the physical traps is handled by a third party and they sent this over to the backend API.

Stripe is used as payment provided and SendGrid for email notifications.

In terms of infrastructure, the static site is hosted on CloudFront/Amazon S3, backend running in a container on EKS and PostgreSQL running on Amazon RDS. AWS Cognito is being used for authentication.

Source was all in GitLab which was also used for CI/CD. Frontend would be pushed to S3 and the backend would be built and pushed to ECR, then deployed to EKS.

## Kicking goals

The goal of my initial involvement was looking to see if there were any opportunities to save cost. The cloud bill has been steadily rising and needed some TLC.

A breakdown of main monthly costs were as follows:
* EKS: $144
* EC2: $85
* RDS: $65
* ELB: $36

Total monthly costs was ~$380 or ~$13 a day.

This was running both staging and prod, so we had a 2 node cluster for EKS running a couple of t2.micro's and 2 RDS instance, one for staging and one for prod.

So we had some overhead in EKS that I didn't think was necessary. If we could move that to something like Fargate then we could drop EKS and EC2 in favour of a per container cost.

We would need to add some additional services to make up for what EKS was providing like SSL certs and secret management but we could use Certificate Manager and Secret Manager to do this.

## Repeatability is the name of the game

I also want to build out the new deployment using a template and use Infrastructure as Code.

We are trying to attract more developers to help out with this project and having the deployment as an artifact would really help people get up to speed and understand what was going on.

However, there are some things that I like to keep outside the deployment templates like SSL certs and secrets so we built these manually for now.

I then used the AWS CDK to produce my build artifacts. I was also able to import my certificates from ACM, secrets from ASM and records from Route53.

One nice thing that I found whist using the CDK and Fargate is that when using Secret Manager, you can specify that the environment variables are secrets and the will be hidden from the Fargate service. This way we can complete separate our responsibilities and just have a few key people in charge of secrets and the developers don't have to worry about this.

Here is a code snippet on how the Fargate service imports the secrets:
```typescript
    //Create Fargate Task Definition
    //Use image from ECR, setup logging to CloudWatch with custom prefix, get config.environment  variables from Secret Mgr
    const fargateTask = fargateTaskDefinition.addContainer('ss-ts-container', {
      image: ecs.ContainerImage.fromEcrRepository(ecrRepo, imageTag),
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: `$ss-api-${config.environment}-logs`,
      }),
      environment: {
        NODE_ENV: config.environment,
        AWS_DEPLOY: secret.secretValueFromJson('AWS_DEPLOY').toString(),
        yes_definitely_send_mail: (config.environment === 'prod') ? 'yesplease' : 'nothanks'

      },
      secrets: {
        SS_API_ENV: ecs.Secret.fromSecretsManager(secret)
      }
    });
```

So here I used `ecs.Secret.fromSecretsManager(secret)` which pulls in the secrets, however, there were two problems with this. The first issue was secrets, we had about 5 secret values and storing these individually in ASM would be a couple of $ a month and the whole point was to reduce cost. The second issue was that if I changed the code to support a single ASM secret it would break local development. I still needed a way to run the solution locally offline, on the old stack and migrate to Fargate.

ASM lets you store multiple values in a single secret, ok not the best solution for versioning if you just want to change one of them but it helps keep the cost down to only use one. Calling ASM would return all of these as JSON, so I just need to parse this into environment variables when deploying on AWS.
We were using `dotenv` in the backend to manage environment variables, so I could put a switch in to use the `.env` file for local development and parse the JSON if it was being deployed on AWS.
The solution was to use another environment variable!
I would check for the existence `AWS_DEPLOY` and if this was present then it would parse the ASM JSON string and if not would use the local file.

```typescript
    dotenv.config();
    //add logic to check for AWS ENV variables delivered as JSON string from Secret Manager
    if ('AWS_DEPLOY' in process.env) {
      try {
        let secrets = JSON.parse(process.env['SS_API_ENV']);
        this.vars = secrets as any;
        if (process.env.NODE_ENV != 'prod') {
          console.log(process.env['SS_API_ENV']);
        }
      } catch (error) {
        console.log(error);
        throw "Unable to parse AWS Environment";
      }
    }
```

## Just the facts

So with the code changes done we cut over staging to the new system and all was well except for sending emails! Turns out I had forgotten that k8s stores it's secret values as base64!! But that was a quick fix after a mild panic.

So a little over a month after I got involved we cut over prod to the new Fargate solution and cleaned up the EKS resources. Now the bill looks like this:

[<img src="{{ site.baseurl }}/images/2019-12-02-k8s-to-fargate/bill.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-12-02-k8s-to-fargate/bill.png")

* RDS: $65
* Fargate: $65
* ALB: $36

Total monthly cost: ~$170 or $5 a day! So we cut the bill in half.

I know this is going to change the world but I felt good to me to use my knowledge to help an amazing cause.

This is the just the start, we are going to see if we can bring down the costs further by using Lambda and DynamoDB for a serverless backend.