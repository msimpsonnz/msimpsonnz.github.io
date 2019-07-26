---
layout: post
title: Functions testing Functions
summary: This time we work through building an API load test using Azure Durable Functions and Application Insights
---

### TL;DR
Don't be like me, try and use [loader.io], [JMeter], [Blazemeter], [locust.io], [Vegeta] ect ect. I needed something quick and effective and didn't have much experience with some of these tools, or they couldn't quiet meet my needs.
Check out the [GitHub repo] for source.

### The problem
We were working on a new project using Azure Functions and Cosmos DB, this was going to be a high throughput system and we wanted to ensure this would perform under load.

However, this was using custom authentication system, with all of the APIs being authorized using JavaScript Web Tokens (JWT).

So what we needed was a load test that could go and get a token and use that in subsequent requests.

### Right tool for the job
Our requirements were pretty simple, we had a request to get a token, then a subsequent request to get some data using that token and sample file of user data to use for each request.

Now I know I could have striped out the auth, used one account and a hundred other things to make this easier, but we wanted to make this look more like the user load we were expecting.

We wanted to use 100's and then 100's of thousands of accounts to create some load on the database, test the indexes and ensure we were getting performance across the board.

I have used [loader.io] from SendGrid before and this is a great tool for quickly putting some load against an API backend. They have an awesome free tier and it is super quick to get going. I love this tool and if you haven't seen it then definitely worth checking out.

However, at the time of writing (March 2019) "Loader.io supports HTTP   basic access authentication" see [creating a test]. Ok this wasn't going to work for me...next!

I then thought of Azure DevOps but remembered that they recently announced the end of life for their [cloud based load testing service].

Right, I use [Postman] all the time to test the APIs so can I leverage this?

Maybe...???

I can use the awesome pre and post scripting to input data and chain the authentication requests. Awesome, but I need to run this at scale... ok great I can use [Newman] which is the cli runner for PostMan.

A few hours later...

Nope! 
I ran into this [Newman issue] where it doesn't yet have the features to support updating environment variables to chain requests...next!

What else haven't I tried?

Ok, so I've used [JMeter] before and maybe past experience kept me from wanting to use this again but none the less I had a look at [Blazemeter] ... the free tier wasn't going to cut it...next!

I spent a bit more time looking at other options which gave me more flexibility like [locust.io] and [Vegeta], but I would still need to wire up some way of running compute to actually produce the load.

This got me thinking back to my [previous post] using Durable Functions to bulk load data into Cosmos. I could use the same pattern to spin up Azure Functions to ping my other Azure Functions running the API...inception!

### Durability for the win

Check out the [GitHub repo] for the source, it's pretty rough, I ended up using [Bogus] to create fake user data which I loaded into Cosmos via another Function. Then use the JSON output which is batched up by Durable Functions and spawns an activity to run the steps, all using .NET Core and HttpClient library.

Below is the main orchestrator function, which take a JSON array as the input, batches this up and then runs an activity function for each request.

```csharp
[FunctionName("HttpLoader")]
public static async Task<List<string>> RunOrchestrator(
[OrchestrationTrigger] DurableOrchestrationContext context)
{
    var input = context.GetInput<List<User>>();

    var outputs = new List<string>();

    var batches = await context.CallActivityAsync<List<List<User>>>("HttpLoader_BatchReq", input);

    var parallelTasks = new List<Task<string>>();

    foreach (var batch in batches)
    {
        foreach (var user in batch)
        {
                Task<string> task = context.CallActivityAsync<string>("HttpLoader_Load", user);
                parallelTasks.Add(task);
        }

    }

    IEnumerable<string> results = await Task.WhenAll(parallelTasks);
    return results.ToList();
}
```

### How did we do?

So that is great but I needed some way to visualize the results! Application Insights enabled, no code changed and I get this!

[<img src="{{ site.baseurl }}/images/2019-03-22-Functions-testing-Functions/logs.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-03-22-Functions-testing-Functions/logs.png")

And the metrics by request count per second

[<img src="{{ site.baseurl }}/images/2019-03-22-Functions-testing-Functions/counts.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-03-22-Functions-testing-Functions/counts.png")

The actual numbers were
* Total Duration: 123.023 (sec)
* Total Request Count: 13,105
* Average RPS: ~106
* Peak RPS: 224

Not bad as a baseline, now its time to run this as part of the Blue/Green deployment before we cut over!

[loader.io]: https://loader.io/
[JMeter]: https://jmeter.apache.org/
[Blazemeter]: https://www.blazemeter.com/
[locust.io]: https://locust.io/
[Vegeta]: https://github.com/tsenart/vegeta/
[creating a test]: https://support.loader.io/article/15-creating-a-test/
[cloud based load testing service]: https://devblogs.microsoft.com/devops/cloud-based-load-testing-service-eol/
[Postman]: https://www.getpostman.com/
[Newman]: https://github.com/postmanlabs/newman
[Newman issue]: https://github.com/postmanlabs/newman/issues/1679
[previous post]: https://msimpson.co.nz/Cosmos-Durable-Functions/
[GitHub repo]: https://github.com/msimpsonnz/misc/tree/master/src/Function.Demo.Perf
[Bogus]: https://github.com/bchavez/Bogus