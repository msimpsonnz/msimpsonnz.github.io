---
layout: post
title: Azure Front Door for geographic availability
summary: A quick walk though of getting Azure Front Door up and running to provide a global scale upload service
---

[Azure Front Door] is a new service that provides
> Scalable and secure entry point for fast delivery of your global applications

SSL offload, global load balancing, WAF, DDOS protection and more! As we are seeing an explosion in distributed applications, it's a great option to be able to put this all together under a single namespace and provide a global solution.

I was putting together a [Blazor demo] that needed Azure Functions for the API layer and storage for static hosting and file uploads. This was a good opportunity to test things out.

Pro tip: first you need the updated CLI bits
```
az extension add --name front-door
```

Use this [deployment script] the create the service, backend pools and rules

Awesome so now I have three services sitting behind Front Door to provide availability, better performance and simple entry point for the application components.

I can cache some of my API calls at the edge now which is great, but I really wanted to see what impact this would have on the performance of uploads.

So I built a simple .NET Core app that would upload a random file to blob using a SAS token, [this app] doesn't use the Azure Blob SDK, as I found out that Front Door is removing some of the required headers so the uploads fail, more info [here](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-http-headers-protocol).

I built a Ubuntu VM in Azure Central US and got the .NET Core app running, I also created an Azure CDN Standard profile and connected this to the actual storage accounts to test three different scenarios:
1. Upload direct to Azure Storage
2. Upload via CDN to Azure Storage
3. Upload via Front Door to Azure Storage

The server was in Central US and the storage account was in Australia East, although I am still "within" Azure it was still a good test with distance in between.

[<img src="{{ site.baseurl }}/images/2019-02-11-Azure-Front-Door/upload-test.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-02-11-Azure-Front-Door/upload-test.png")

* Direct = 14.05
* CDN = 8.13 (73% improvement)
* Front Door = 1.96 (717% improvement over Direct and 415% over CDN)

WOW! Results really speak for themselves!

[Azure Front Door]: https://azure.microsoft.com/en-us/services/frontdoor/
[Blazor demo]: https://github.com/msimpsonnz/training-portal
[deployment script]: https://github.com/msimpsonnz/training-portal/blob/master/deploy/deploy.sh
[this app]: https://github.com/msimpsonnz/az-blob-uploader