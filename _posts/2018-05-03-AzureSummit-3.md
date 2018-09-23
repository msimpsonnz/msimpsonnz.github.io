---
layout: post
title: Azure Summit 2018 - Part 3 - Going live and taking it global
summary: Part 3 of my presentation for Azure Summit 2018
---

## Part 3

This is a three part post, so if you skipped [part 1]({{ site.baseurl }}/AzureSummit-1/) you might want to check that out first. Or are you still looking for [part 2]({{ site.baseurl }}/AzureSummit-2/)?

However, a quick recap, we are working with Bob. Bob has a under performing web application, we need to improve performance, so we took some benchmarks using VSTS and have established a baseline. We then identified the issues applied some design patterns and ended up with 4x performance bump!!

## Going live with V2

So this performance bump is all well and good but how do we get these changes into production without risking app stability?

This isn't a breaking change but we are adding a new dependency so we should bump the version. With these kind of API changes we don't want to have to rev the client for every single `PATCH` fix so we are not doing full [SemVer](https://docs.microsoft.com/en-us/dotnet/core/versions/#semantic-versioning) but we do have a versioning scheme for `MAJOR` and `MINOR` changes. I'd like to think we can deprecate the old code fairly soon after this which I why I would opt to push Major and go to version 2.0.

So that is the theory, in practice we can us another handy .NET Core library that can do all the versioning for us `Microsoft.AspNetCore.Mvc.Versioning`

Awesome, we now have a library that supports versioning using URL path, query string or headers. We can also infer a default version if one is not provided, which gives you and your customer the flexibility to use a specific version if they need too and you can keep moving forward without waiting for everyone to catch up.

So I add the library and add this to `Startup.cs`

<script src="https://gist.github.com/msimpsonnz/c3168ee510cd938b72240548c81d9945.js"></script>

Then I just add the new version of the controller and decorate it

<script src="https://gist.github.com/msimpsonnz/ec1bb5fb560aa1fad7862aaf8bd20056.js"></script>

Great, so now I can test my new service by adding `/api/shop?api-version=2.0` to my test. If I don't specify any `api-version` then the old code executes as normal. This is perfect, I can release my update safe in the knowledge I have not changed any of the old code and just letting MVC routing do its magic.

## Feature Flags ##

Version control is great and we are pushing forward with that now, but what if we are doing a minor enhancement or we want to enable a piece of functionality for just one customer. [Feature Toggles](https://martinfowler.com/articles/feature-toggles.html) (aka Feature Flags) can be really useful here.

We can write our code to check if certain features are *"toggled"* or *"flagged"* and act accordingly. For example, a new feature enhancement could be made and deployed and the calling method could use a configuration value to determine where to execute it or not.

I have enabled a feature on the API to inject failures for every *"n"* request. This *"feature"* is enabled through environment variables and my controller method checks the state of the feature, if this is enabled then failures are going to occur. This is a good technique for getting in chaos engineering and deliberately introducing failures to test how the system reacts.

<script src="https://gist.github.com/msimpsonnz/08b10461dc1f0200b503883f1b9b1458.js"></script>

## Going global ##

Awesome, so we have fixed our perf issues, added versioning and features. Now it's time for world domination!

Bob wants to expand into the US, but this needs to be pragmatic and we don't want to spend a fortune while we are just starting out. What we do want is to reuse as much as what we have and what if we could improve the availability of the current solution whilst we are at it!

Azure SQL DB has an awesome [geo-replication](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-geo-replication-overview) feature which is available on all tiers!

Nice, so we can use our existing database and replicate from Australia to USA, giving us instant DR! Then we take the existing code for the Web App, Service Bus and Function App, deploy these in a new resource group in the US.

Then it's just a case of updating application settings on the US side to point to the SQL replica and the new Service Bus Queue.

But what about writes? I can't use the SQL replica as it's *"read-only"* replica. No problem, I simply point the Function App at the SQL master. Sure my writes are now going around the globe but that's fine too, Service Bus is persistent and I can even create a dead letter queue if there are any issues, these messages are stored and can be processed later.

I then layer on [Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview) which can do geo based routing, so clients in Southeast Asia go to Australia region deployment, whereas North and South American users get to the US.

This is the final result

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/HL-ARCH-FINAL.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/HL-ARCH-FINAL.png)

And this is what it ended up costing

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/FINAL-COST.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/FINAL-COST.png)

## That's a wrap ##

So we started out benchmarking performance, identified root cause, implemented a new fix using versioning, took a look a feature flags and finished it all off by deploying in another geo to provide DR and improved UX!

That's it! I hope you enjoyed this mini series and found it useful. Any constructive feedback is welcome and please hit me up on Twitter or LinkedIn.