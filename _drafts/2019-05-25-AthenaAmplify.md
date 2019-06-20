---
layout: post
title: Athena Amplify'd
summary: In this post we take a look at the Amplfiy framework as a quick and secure way to build a front end for AWS services
tags: [aws, amplify, cognito, api-gateway, athena, lambda]
---

### ~~Mystify me~~ Amplify me!

> Amplify: An opinionated, category-based client framework for building scalable mobile and web apps

I have seen a few people thinking [Amplify](https://aws-amplify.github.io/docs/) is just for mobile...this could be due to the fact it appears under the "Mobile" section of the AWS Console! But we can see from above that is is also for web apps too.

So what do we have here => "an opinionated framework", I really like how this is being called out, I think it's important for this to be stated upfront. I have spent many hours with other frameworks figuring things out when there are hundreds of ways to accomplish the sam thing and multiple versions of best practice, so it's great that we have most of the complex thinking done for us and we can get on a build something.

It is also deeply linked with so many AWS services and all just a cli command away
* Auth - using Amazon Cognito => `amplify add auth`
* Hosting - using Amazon CloudFront and Amazon S3 => `amplify hosting add`
* API - using AWS AppSync or Amazon API Gateway => `amplify add api`
* Analytics - using Amazon Pinpoint => `amplify add analytics`
* Interactions - using Amazon Lex to create chatbots => `amplify add interactions`
* Storage - using Amazon S3 or Amazon DynamoDB => `amplify add storage`

That is what I love about Amplify, so much of the heavy lifting is taken care of for me and why I have started using it for demos and proof of concepts in my world and we are seeing production implementations take off too.

It really is changing how we can approach taking an idea to something secure and scalable, measured in hours and days not weeks and months!

### Security by obscurity be gone!

With building little apps to prove something out it is often a trap to get something working at the expense of doing it with security in mind.
Now adding auth to a web app is not that "hard" (comparatively speaking), we can "lock the front door" so to speak by making sure all calls are authenticated to the endpoint, but what about the backend API?

This is where I think Amplify really shines, I can add auth and get that working with all the Amazon Cognito goodness and then I add my backend API, whether I'm using Amazon AppSync or Amazon API Gateway we simply ask for backend calls to be authenticated and Amplify takes care of wiring up authentication to the same Cognito pool. I can then use the `API` component and the framework takes care of authentication headers and even the URL signatures!

### Wrap your app around me

So back to the title of this blog and the inspiration behind it. We were looking for options to provide end users access to Athena without going through the AWS Console, essentially some sort of wrapper that could be secured and something we could stand up in a few hours to prove it out and work through the viability.

So a few hours later...
I found a super handy Express implementation for Athena called...wait for it...[Athena Express](https://www.npmjs.com/package/athena-express) so I spun this up in a quick and dirty Lambda function and worked like a dream.

I'll share the code [here](https://github.com/msimpsonnz/misc-aws/tree/master/amp-athena) but its super basic stuff and the Lambda function is a disaster and SQL injection waiting to happen!

But this was about rapid prototyping to prove out the idea, we can clean up the implementation later, for now we have a working concept iterate on.

Hope you enjoyed...now go build something!