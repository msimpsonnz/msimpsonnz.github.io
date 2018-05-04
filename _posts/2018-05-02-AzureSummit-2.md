---
layout: post
title: Azure Summit 2018 - Part 2 - Patterns and Practices
---

## Part 2

This is a three part post, so if you skipped [part 1]({{ site.baseurl }}/AzureSummit-1/) you might want to check that out first.

However, a quick recap, we are working with Bob. Bob has a under performing web application, we need to improve performance, so we took some benchmarks using VSTS and have established a baseline.

## Azure Architecture Center

Quick call out to the [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/), this is a great resource for cloud design patterns, best practices and common application design patterns. Fantastic resource and contains lots of generic advice and patterns which is not language or platform specific. We are going to use a few of these patterns 

## Insulating GETs from the data store with Cache-Aside

[Cache-Aside](https://docs.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
>"Load data on demand into a cache from a data store. This can improve performance and also helps to maintain consistency between data held in the cache and data in the underlying data store."

Cache-aside is a great way to support slow moving data and not just blindly hammer the backend data store with the same query millions of times a day. With this pattern I can potentially use a lower tier of Azure SQL DB as I don't need it to serve up the same query all the time for the product catalog as I can just cache it.

Ok cool, so [Redis](https://redis.io/) is a cache right? Yep! OK GO...STOP...WAIT

Redis is great and we love Redis!! However, in part 1 we talked about *"doing more with what you have"*

So my app is pretty simple and I am using Vue to do some heavy lifting on the client side and my .NET Core API is nice and lightweight, so I have some spare memory available on the Azure App Service! Awesome, lets put that memory to use!

**Disclaimer - check you have headroom to enable this and you cache the slow moving stuff, do your own research for your app**

Ok...ok...disclaimer read and homework done, what next? [In-memory Caching](https://docs.microsoft.com/en-us/aspnet/core/performance/caching/memory?view=aspnetcore-2.1)

Great so in my .NET Core WebAPI `Startup.cs` under `ConfigureServices`, I can add `services.AddMemoryCache();`

Then I can change my database call to use the cache

<script src="https://gist.github.com/msimpsonnz/f1d9f0118e1ff681ee05d85116301e0d.js"></script>

Define a cache key, then check for it's existence, create it it's not there and use it for 1 hour before refreshing.

Nice! Run the perf test and here are the results

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-CACHE-GET.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-CACHE-GET.png)

So 100% improvement in performance for **$0** additional cloud spend and just the development effort to write, test and deploy!

But there is more...look what it had done to the SQL database requests

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-CACHE-SQL.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-CACHE-SQL.png)

So as I said before, I don't need to turn the volume up on SQL to 11 and can use the same tier.

## Queue based load leveling ##

Great, so the cache has fixed the GETs requests but I can't cache POSTs in the same way. I need a better way of getting orders from the web API to the database. Architecture Center has the solution - [queue based load leveling](https://docs.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling)

>"Use a queue that acts as a buffer between a task and a service it invokes in order to smooth intermittent heavy loads that can cause the service to fail or the task to time out. This can help to minimize the impact of peaks in demand on availability and responsiveness for both the task and the service."

This is exactly what I want! So a queue service in Azure, let me think...
* Storage Queues - Yep
* Service Bus - Yep
* Event Hub - Sure
* What about Event Grid, can that help?
    * Maybe, but it's not technically a queue.

Storage Queues would work fine, but I might want to move to a [Pub/Sub](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-how-to-use-topics-subscriptions) model in the future so I figured I would start with Service Bus. Service Bus has a Basic tier and if I did want Pub/Sub then I could upgrade to Standard when I needed Topics and Subscriptions. 
Event Hubs would work but I didn't need the scale just yet. Event Grid is "events" and I wanted the transactional nature of Service Bus, great article [here](https://docs.microsoft.com/en-us/azure/event-grid/compare-messaging-services) comparing these services.

Great, so I have a Service Bus queue to take orders to from a POST, but I need something at the other end of the queue to process messages and get them into the database. If we were super cost sensitive it could be hosted in a [Web Job](https://docs.microsoft.com/en-us/azure/app-service/web-sites-create-web-jobs) running in the same App Service as the Web API.

I wanted to do more with .NET Core and [Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-versions) which is still in preview (May 18), so I fired up VS with the preview runtime installed and got cracking.

I was able to take some of the logic from the existing Web API code and I was still using Dapper as it has a .NET Standard library. Added a couple of Function [triggers and bindings](https://docs.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings) and I was off and running.

<script src="https://gist.github.com/msimpsonnz/16583c9d6b4815e61a4cc1774edf066b.js"></script>

However, as I have my Function app running in a [Consumption Plan](https://docs.microsoft.com/en-us/azure/azure-functions/functions-scale) if I just dump 000's of messages on the queue, Functions hosts will spool up to meet demand, smash my SQL DB and I will be in the same position as before. No drama, I can use the `host.json` config of the Function App to set `maxConcurrentCalls` to `1` so that the runtime will process a single message at a time and I can protect the backend data store.

So this is what we have ended up with

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/HL-ARCH.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/HL-ARCH.png)

So lets run the perf test again and see how we did.

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-QUEUE-POST.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-QUEUE-POST.png)

Awesome, so we have improved the write performance by 100% from where we started.

I was stoked but the RPS numbers didn't look that great, to start with I didn't bother with Async/Await or any "proper" application design. *"Done is better than perfect"*

So what would happen if I took a bit more time and looked at async and some [Domain Driven Design](https://martinfowler.com/tags/domain%20driven%20design.html) patterns. I ended up with a few abstractions for Service Bus with some interfaces and an event handler that I could just push messages onto using async.

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-QUEUE-ASYNC.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-QUEUE-ASYNC.png)

**BOOM!! Another 100% improvement from adding the queue, taking us to 4x the performance from when we started all of this!**

Ok..calm down...how much did this cost me?

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-QUEUE-COST.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-QUEUE-COST.png)

**DOUBLE BOOM!!**

Yes, that's right folks all of this could be yours, for the princely sum of an additional **$29 NZD** (May 2018 public price list)

Ok, all jokes aside, the development effort - code, testing and deployment pails in comparison to cost of the infrastructure, but I guess that was the point I was trying to make with all of this.

All good, onwards and upward to the final [part 3]({{ site.baseurl }}/AzureSummit-3/) where I will cover some techniques on how to get all of the above goodness into production without breaking anything...well nothing that can't be rolled back easily.