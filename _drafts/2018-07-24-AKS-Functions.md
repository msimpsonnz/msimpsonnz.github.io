---
layout: post
title: Azure Kubernetes Service and Azure Functions
---

# Orchestration with the power of Functions (K8s^F)

I have been a long time fan of both orchestrators and Functions as a Service (or Serverless if you like).

### Orchestration
K8s has become the defacto orchestrator but I have spent a few years working with Microsoft's Service Fabric, which is their orchestration platform for microservices and runs much of the underlying Azure services at scale.

So an orchestrator is a must for most larger scales microservices deployments, providing placement, service discovery, high availability, maintenance....the list goes on.

Microsoft has a number of orchestration offerings but are focused on Azure Kubernetes Service (AKS) which is their managed K8s offering in the cloud.

### FaaS
Functions as a Service (FaaS) was originally conceived by AWS in it Lambda service which was previewed almost 4 years ago (Nov 2014). This was a relative breakthrough at the time, just give them your code and it would execute and be billed on a per second basis.

Microsoft released Azure Functions in preview back in 2016 and there has been a massive growth in this area of the platform. It's had its challenges and currently undergoing a revision in the runtime but has a great open source, local debug and extensibility story.

[OpenFaas](https://docs.openfaas.com/) is another OSS project which has gained a lot of traction and they were actually the first to combine FaaS with K8s providing a simple what to self host your own platform.


### Orchestration of FaaS
So fast forward to today and these two worlds are getting even closer....for Microsoft at least.

Just last week they announced support for running the Azure Functions runtime in a container on Kubernetes, right [here](https://github.com/Azure/azure-functions-core-tools#getting-started-on-kubernetes)

I think this is great, we can really see the vision for these products starting to merge. As a developer I can write once, debug locally, ship and let ops team deploy this for me on their chosen platform.
* If you already have some capacity in the K8s cluster then running these functions can use that spare capacity
* For burst scale out from AKS, I could connect to Azure Container Instance (ACI) which would be quicker that scaling AKS nodes
* For something that needs massive scale, I could deploy the same code natively on Azure

# Project Ringo

[Dan](https://twitter.com/DanielLarsenNZ) and I have been working on a [Spotify bot](https://github.com/Ringobot/ringo) for a while now. We are using this a continuing project where we can try out different technology, some of which is a duplicate of what is there, but mainly so we have a common domain that we understand and work on regularly. We are going to add more features but some of work I have been doing in the last few months has been around different options to host the backend APIs.
We moved some of the functionality out of the bot as the project grew from an initial idea, I thought it would be good to run this in Azure Functions and also experimented with Durable Functions.

So when I saw the announcement about Functions on K8s, I thought this would be a good place to try out moving an existing project to K8s.

## Adding Dockerfile

Turns out the Azure Functions CLI has support for this, so I just go to my existing Functions directory
`func init --docker`

I get a Dockerfile and go to build.....failed

As I said before I have been using Durable Functions in this project and something in there is not compatible with the Docker extension. I've logged an [issue](https://github.com/Azure/azure-functions-core-tools/issues/598) and will see if it gets picked up or will need to debug some more.

I wasn't done yet, so I decided to make a K8s version of my Functions project, that was simple and added my existing code to the new project. I ended up with a new Dockerfile but did run into some issues building the image and having to reference an external project dependency which contains my core models and services.

```
FROM microsoft/dotnet:2.1-sdk AS installer-env

COPY ./Ringo.Common/ /src/Ringo.Common
COPY ./Ringo.Functions.K8s/ /src/dotnet-function-app
RUN cd /src/dotnet-function-app && \
    mkdir -p /home/site/wwwroot && \
    dotnet publish *.csproj --output /home/site/wwwroot

FROM microsoft/azure-functions-dotnet-core2.0:2.0
ENV AzureWebJobsScriptRoot=/home/site/wwwroot

COPY --from=installer-env ["/home/site/wwwroot", "/home/site/wwwroot"]
```

There is a handy CLI function that lets you build the image, copy it to a registry and then deploy to Kubernetes
`func deploy --platform kubernetes --name ringo-artist --registry mjsdemoreg.azurecr.io --config ./artist.yaml`

Awesome!!!

I had to create a custom config as I needed to inject some env as secrets using a Config Map in K8s for the API keys the code needs, but once I had that figured I was away and working.

I was going to extend this solution to run this on ACI, but at this stage it doesn't support using HTTP triggers!