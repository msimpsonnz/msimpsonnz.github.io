---
layout: post
title: Improving application response times with Amazon AppStream
summary: In this post we look at using Amazon AppStream to provide better response times to remote users for sensitive internal apps
---

### Streaming isn't just for movies

With [Amazon AppStream](https://aws.amazon.com/appstream2/) we have a fully managed service that hosts desktop applications and "streams" them to the client using a browser.

The scenario we were looking at was providing remote workers on a different continent access to internal applications. We looked at a few options, caching and reverse proxy, however, there are potentially 100's of apps and its not a one size fits all, some of these are full desktop apps too.

So if we provision a AppStream resource in Sydney we can hop on the AWS network backbone to Singapore, where the apps are hosted.

I scaffolded out this for testing and used the CDK again to generate the templates, the source can be found [here](https://github.com/msimpsonnz/aws-misc/tree/master/app-stream/cdk).

The above contains three stacks, the CDK is still in development and they are working on a method for regional deployments. So in this instance the AppStream stack is deployed in Sydney and web server in Singapore. Once this is complete we get the VPC ID's from both deployments and run the remaining stack manually to create the VPC peering connections.

