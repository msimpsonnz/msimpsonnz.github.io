---
layout: post
title: Improving application response times with Amazon AppStream
summary: In this post we look at using Amazon AppStream to provide better response times to remote users for sensitive internal apps
tags: [aws, cdk, appstream2, single-sign-on]
---

### Streaming isn't just for movies

With [Amazon AppStream](https://aws.amazon.com/appstream2/) we have a fully managed service that hosts desktop applications and "streams" them to the client using a browser.

The scenario we were looking at was providing remote workers on a different continent access to internal applications. We looked at a few options, caching and reverse proxy, however, there are potentially 100's of apps and its not a one size fits all, some of these are full desktop apps too.

So if we provision a AppStream resource in Sydney we can hop on the AWS network backbone to Singapore, where the apps are hosted.

I scaffolded out this for testing and used the CDK again to generate the templates, the source can be found [here](https://github.com/msimpsonnz/aws-misc/tree/master/app-stream/cdk).

The above contains three stacks, the CDK is still in development and they are working on a method for regional deployments. So in this instance the AppStream stack is deployed in Sydney and web server in Singapore. Once this is complete we get the VPC ID's from both deployments and run the remaining stack manually to create the VPC peering connections.

### Setting up a test

The above deployment also builds the web server in Singapore so all we have to do now is test.

[locust.io](https://locust.io) is a nice simple testing framework that user Python to generate a load test. I setup a simple test that could be run from my local machine to the public IP of the web server. Then I would repeat the same step from AppStream. The second part took a bit more time, I had to create an image and install Python but I was up and running quickly. Below are the results:

#### Direct Connect - Avg: 309 / Max: 930
[<img src="{{ site.baseurl }}/images/2019-06-03-AppStream/1direct.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-06-03-AppStream/1direct.png")

#### AppStream - Avg: 175 / Max: 354
[<img src="{{ site.baseurl }}/images/2019-06-03-AppStream/2appstream.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-06-03-AppStream/2appstream.png")

So we have ~43% improvement in average response time, but we have taken ~60% off the max response time, this is definitely going to be a better time for the end users.