---
layout: post
title: Working with Azure Functions and Log Analytics
summary: Log Analytics is a great platform for getting insight out of applications and this explains how to get it working with multiple Azure Functions
---

### Critical Logging

It is critical to get your logging strategy defined and implemented, this can help in all aspects of the dev loop, from added features to troubleshooting bugs in production.

I really wish I had told myself this again and again during my latest adventure, but what turned from a proof of concept, to pilot to production in just under three months, proved again that the perception of "just writing some code" is always slower than taking some time to plan for the future.

The last thing on my mind during our 2 day pilot was structured logging, let alone a strategy for managing serverless applications across different deployment stacks. As the project headed for production we realised that we had to implement proper logging before we could go live.

But this was a learning opportunity, I'll save the structured logging for another time, I want to focus on what I learned about Application Insights and how this saved me from a half implemented logging strategy. 

### Application Insights

[Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) is an Application Performance Management product. It will monitor client side, server and background services. You can read about it all from the link above, I wanted to focus on using it to monitor Azure Functions as App Insights is the default monitoring solution for Azure Functions now.

Again I don't want to get off track too much, but we ended up with a number of Azure Function apps. The unit of deployment for Azure Functions is the "Function App" so we decided we would have separate apps for each part of our microservices architecture. Why is this important/relative? We decided to have separate Application Insights resources for each Function App, we could deploy another stack in another region and have a separate monitoring instance, again there were decisions made here and it is important to the queries below, just check your own requirements and trade offs here before doing the same.

### Kusto

App Insights has some great metrics and charts out the box, the [Live Stream](https://docs.microsoft.com/en-us/azure/azure-monitor/app/live-stream) is a great example. But if you want to get into some custom metrics queries, then [Kusto](https://docs.microsoft.com/en-us/azure/kusto/query/index) is the way to go, this is the query language used for Log Analytics which is the data store behind Application Insights, you can review the basics [here](https://docs.microsoft.com/en-us/azure/azure-monitor/app/analytics)

For our scenario we had a couple of Function Apps deployed, the first would take a blob upload and run it through Azure Media Services (AMS), the second would take the output from AMS and notify our application, some sample code is [here](https://github.com/msimpsonnz/MediaServices.Demo/tree/master/MediaServices.Demo.Function).

So that is great we have a serverless encoding pipeline, but from Application Insights we have a big hole in our timeline, we cannot track AMS as a dependency, we just get a notification when the job is complete. So my problem was stitching these two functions together to find out how long it was taking from event trigger to application notified.

I'd only half implemented structured logging at this stage, so I had a correlation id so I could track multiple events across their lifetime, but hadn't quite got this working with Application Insights Custom Metrics!

*WARNING: this is a hack, but demonstrates the awesome power of Kusto and shows no matter what you were thinking when you "tried" a logging strategy you can still get some data.*

*Don't be like Matt, fix your logging!!!*

#### mvexpand

So in the below code we
1. look for messages with specific text
2. get the timestamp
3. extract the correlation id which is appended to "output-"
4. make this a string
5. repeat 1-4 on a join dataset from a diffrent App Insights workspace called 'notification' - app('notification').traces
6. use mvexpand and the range command to bucket each execution into 1m grains
7. summarize this

```
(traces | where message contains "Status: Created new output" | project JobCreated=timestamp, concat=split(message, "Status: Created new output output-")[1] | project JobCreated, Id=split(concat, ",")[0] | extend Id = tostring(Id))
|
join
(app('notification').traces | where message contains "assetName = output-" | project JobFinished=timestamp, Id=split(message, "output-")[1] 
| mvexpand samples = range(bin(StartTime, 1m), StopTime , 1m)
| summarize Jobs=count(Id) by bin(todatetime(samples),1m)
| order by Jobs desc
```
This gives us a view of how many executions we had in a given ranged and was used to plan for peak application demand!



```
(traces | where message contains "Status: Created new output" | project JobCreated=timestamp, concat=split(message, "Status: Created new output output-")[1] | project JobCreated, Id=split(concat, ",")[0] | extend Id = tostring(Id))
|
join
(app('notifications').traces | where message contains "assetName = output-" | project JobFinished=timestamp, Id=split(message, "output-")[1] 
| extend Id = tostring(Id)) on Id
| project Id, JobCreated, JobFinished
| extend duration=totimespan(JobFinished-JobCreated)
| where JobCreated between((todatetime('2018-11-12T16:20:00Z') .. todatetime('2018-11-12T16:30:00Z')) 
```