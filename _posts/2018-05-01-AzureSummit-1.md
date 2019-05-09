---
layout: post
title: Azure Summit 2018 - Part 1 - Establish a performance baseline
summary: Part 1 of my presentation for Azure Summit 2018
---

## Azure Summit 2018

I've just finished my session at the [Azure Summit](https://www.microsoft.com/en-nz/apac/azuresummit/) in Wellington, wrapping up the event after the inaugural Auckland event a couple of weeks back. I had a great time sharing some performance engineering tips and tricks that my team at Microsoft have used in a number of customer engagements over the past few months.

My session title was "Accelerating App Innovation with DevOps" but I was told I could have some artistic license over the content to tailor this for New Zealand.

So I really set out to tell a story about "how to do more with what you have.....or don't break the bank getting there"

## Bob's Awesome Appliances

We didn't have a good single example so I created "Bob's Awesome Appliances"! Bob has moved from running a business in his garage on Excel to trying to conquer the world with Single Page Apps, APIs and databases running in the cloud.

The problem Bob was now having is that during peak sales on the website he was getting poor user experience and application performance. The frontend devs are blaming the backend devs and vice versa, he was losing money and every fix was a point solution.

How do we really fix this? Not just the performance issues, but how do we move forward, still allowing innovation and features,  but without the blame game when things go wrong or slow down. All software has bugs, all software fails.....eventually.

**Application Insights and VSTS to the rescue**

We often get given a problem with someone else’s interpretation or assumptions of the root cause, but with little evidence to back this up other than a potentially bias view. Establishing benchmarks and continuous testing allow us to understand how our solution will perform under load and whether or not that last commit killed out user experience!

[Application Insights](https://azure.microsoft.com/en-us/services/application-insights/) and [VSTS](https://www.visualstudio.com/team-services/) can really give you a great understanding of application performance and benchmarks.

Application Insights has three main areas:
* Client Side Metrics
    * The client side is JavaScript based function that runs within the clients browser, this looks at page rates, response times, and failures.
* Server Side Metrics
    * Server metrics run on the server and collect server perf. stats, page loads, HTTP error and tracks dependencies.
* Custom Events
    * Custom event can be run on client or server and allow us to infuse our domain specific logic on event handlers, for example, button click events in the browser on the clients side or custom logic extraction from method calls on the service side code.

There is so much more to this product like application [dependency tracking](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-asp-net-dependencies), [workbooks](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-usage-workbooks) and all the [client usage metrics](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-usage-overview) currently in preview (as of May 18) but I might cover that in a follow up, otherwise I will never get this done!

## Application overview

In the case of [my app](https://github.com/msimpsonnz/misc-microsoft/tree/master/nzsummit18/Retail), this is a .NET Core Web Application using the [SPA hosting templates](https://docs.microsoft.com/en-us/aspnet/core/spa/?view=aspnetcore-2.1), I used VueJS to create Bob's Awesome frontend application and .NET Core WebAPI to host the BFF(backend for frontend). I like this pattern as all the code is in one place and to add App Insights, I just right click in Visual Studio and add the SDK. As the SPA template uses the MVC pattern, it drops the App Insights JS snip I need right in the shared _Layout.cshtml which means all the renders have telemetry straight away with no other code changes required.

I then built some super simple functions, GET to retrieve top 10 products from an Azure SQL DB and POST to insert a newly purchased item. I did using [Dapper](https://github.com/StackExchange/Dapper), yes I could have used EF Core but this was built for speed and to prove a point, plus Dapper is awesome!! I was going for minimal setup here so I used 2 x B1 App Service and a B1 SQL DB (which has 5 DTUs)

Done! Not quite, push to Azure Web App and start hitting it with some traffic.

## Cloud based load testing

There is a "Performance" tab in the Web App blade of the portal which I can create a VSTS Cloud based load test from! I need to establish a baseline of the upper limits of what my app can handle for GETs and POSTs. So I built a couple of simple web tests in Visual Studio and used the VSTS Cloud base Load Testing to smash the API with 500 constant users for 1 minutes.

***VSTS GET Result***

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-BASE-GET.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-BASE-GET.png)

***VSTS POST Result***

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-BASE-POST.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-BASE-POST.png)

***SQL DTUs***

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-BASE-SQL.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-BASE-SQL.png)

***Cost Estimate***

[<img src="{{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-BASE-COST.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-05-01-AzureSummit/VSTS-BASE-SQL.png)

Ok, so not great results but this is not about how good the code it, it getting some facts about the system.

In [part 2]({{ site.baseurl }}/AzureSummit-2/) we will talk about patterns to improve performance.