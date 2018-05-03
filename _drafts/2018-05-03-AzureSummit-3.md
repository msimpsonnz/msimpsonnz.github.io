---
layout: post
title: Azure Summit 2018 - Part 3 - Going live and taking it global
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

```
    services.AddApiVersioning(options =>
    {
        options.ReportApiVersions = true;
        options.AssumeDefaultVersionWhenUnspecified = true;
        options.DefaultApiVersion = new ApiVersion(majorVersion: 1, minorVersion: 0);
    });
```

Then I just add the new version of the controller and decorate it

```
    [ApiVersion("2.0")]
    [Route("api/shop")]
    public class ShopControllerV2 : Controller
    {
        private static HttpClient httpClient = new HttpClient();
        private readonly IConfiguration _configuration;
        private IMemoryCache _cache;
        private IEventBus _eventBus;

        public ShopControllerV2(IConfiguration configuration, IMemoryCache cache, IEventBus eventBus)
        {
            _configuration = configuration;
            _cache = cache;
            _eventBus = eventBus;
        }

        [HttpGet]
        public IActionResult GetAllProducts()
        {
            //Action here
        }
    }
```

Great, so now I can test my new service by adding `/api/shop?api-version=2.0` to my test. If I don't specify any `api-version` then the old code executes as normal. This is perfect, I can release my update safe in the knowledge I have not changed any of the old code and just letting MVC routing do its magic.

