---
layout: post
title: Trying to use Span<T> to save money...and falling short
summary: In this post we dive into a new(ish) feature in .NET Core Span<T>, which could allow us to reduce our serverless application footprint but didn't quite work out
---

### Learn through doing... failing and trying again

For me the path to enlightenment is usually through diving in and trying to build something. I was skeptical about posting this as ultimately it turned out to be a fail, but if this post helps just helps one person then I'll call it a win.

### ~~Spin~~ Span to win!

I had been curious to try out the new Span features of .NET Core since they were released with 2.1, if you need a refresh then [this post] should help.

I had done some reading and kept hearing about how these features were improving performance of the underlying BCL. Then I read about how .NET Core 3.0 would be replacing Newtonsoft with a new set of JSON APIs and part of the rational was around being able to use Span<T> for high performance throughput, detailed info on that can be found in this [GitHub issue].

Great, I could use some high performance and lower memory usage to shave some $'s of my serverless bill. The major cloud providers charge based on GB/s for compute time, so the less of these I use the cheaper the bill.

I even had a function to refactor, we had a notification that needed to be persisted for auditing purposes. The notification needed to be parsed to extract some relevant data for later reporting. The current function was using `Substring` to extract a file name from a path. I could replace this with Span and Slice and save on some allocations. Steve Gordon actually did a [great post] about this with an example of how to get allocations down to 0 and benchmarks this using BenchmarkDotNet!

### .ToString or not .ToString

So I built a little [demo] to reproduce the sample environment and benchmark. All was going well and the initial cut I was able to get to zero allocations, but then I tried to create the output as an object!

String is immutable! That is all!

```
    public Event CreateEventWithSpan(EventGridEvent eventGridEvent)
    {
        ReadOnlySpan<char> subject = eventGridEvent.Subject;
        var lastSpaceIndex = subject.LastIndexOf('/');
        return new Event()
        {
            Id = Guid.NewGuid().ToString(),
            BlobName = subject.Slice(lastSpaceIndex + 1).ToString(),
            EventData = eventGridEvent.Data
        };
    }
```

So in order to get my parsed data back out into an object that I could persist in the database I have to call .ToString() on the Span... which allocates memory... which defeats the point and disproves my hypothesis!

So I did a fair bit of digging on this and really came to the conclusion that I was doing it wrong, a better use case would be operating over a stream. But it is great to see more of the underlying APIs using this and I can see this will have a big improvement for the JSON API's in .NET 3.0.

I learnt heaps doing this and that is what I am taking away.

[this post]: https://docs.microsoft.com/en-us/dotnet/standard/memory-and-spans
[GitHub issue]: https://github.com/dotnet/announcements/issues/90
[great post]: https://www.stevejgordon.co.uk/an-introduction-to-optimising-code-using-span-t
[demo]: https://github.com/msimpsonnz/misc/tree/master/src/Function.Demo.Span
