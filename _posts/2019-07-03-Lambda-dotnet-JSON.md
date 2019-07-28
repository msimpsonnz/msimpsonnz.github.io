---
layout: post
title: Using the new .NET Core 3 (Preview) JSON Serialization with Lambda Custom Runtime
summary: Here we take a look at the new builtin serialization capabilities of .NET Core 3, using Lambda Custom Runtime we remove the Newtonsoft library and add a shim for the existing function handler.
tags: [aws, elasticloadbalancing, lambda, cloudwatch, dotnet]
---

### Built for speed

The .NET Team published an [article](https://github.com/dotnet/announcements/issues/90) a while back addressing the future of JSON in .NET Core 3.0, where they talk about removing the dependency on Json.NET and providing a high performance API.

[.NET Core 3.0 Preview 5](https://devblogs.microsoft.com/dotnet/announcing-net-core-3-0-preview-5/) was released back in May 2019 and I have been wanting to take a look at using this with Lambda.

The AWS SDK team provide an awesome set of libraries, tools and samples to help, this is the .NET repo [aws-lambda-dotnet](https://github.com/aws/aws-lambda-dotnet). This is everything from libraries to interact with other AWS services, tools to help deploy functions and a set of templates which scaffold out common scenarios.

Coincidentally the .NET Lambda Libraries also use Json.NET as the built in Serializer/Deserializer for handling request/response, my idea was to see if we can replace this with the new .NET Core 3 functionality.

At the time of writing (July 19) Lambda supports the current Long Term Support (LTS) releases of .NET, more details [here](https://dotnet.microsoft.com/platform/support/policy/dotnet-core), which today is .NET Core 2.1 (1.0 and 1.1 just ended support a few days ago in June).
.NET Core 3.0 and 3.1 (LTS) will be released later in 2019, but the SDK team have release a new library to support .NET Standard 2.0 compatible runtime, so we can now run .NET Core 2.2 and 3.0 (Preview)!

This is the [link](https://aws.amazon.com/blogs/developer/announcing-amazon-lambda-runtimesupport/) which gives more details and also has a full walk through on how to get up and running. I used this article to get my base Lambda functions built.


### Shim to win

I just wanted to get an MVP going to see if this idea would work, so I've only really taken the basic functionality of the Serialization library, it does plenty of other things when working with different event sources. So I thought if I can take a JSON payload and response via an Application Load Balancer that would be a good start.

The main function we are using is `Amazon.Lambda.Serialization.Json.JsonSerializer` which has the two methods we need. So I took a copy and converted this over this the new `System.Text.Json.Serialization.JsonSerializer`

```csharp
using System.IO;
using Amazon.Lambda.Core;

namespace Amazon.Lambda.Serialization.Json
{
    public class JsonSerializer : ILambdaSerializer
    {
        public JsonSerializer()
        {
        }

        public T Deserialize<T>(Stream requestStream)
        {
            var json = new StreamReader(requestStream).ReadToEnd();
            return System.Text.Json.Serialization.JsonSerializer.Parse<T>(json);
        }

        public void Serialize<T>(T response, Stream responseStream)
        {
            System.Text.Json.Serialization.JsonSerializer.WriteAsync<T>(response, responseStream);

        }
    }
}
```

I then removed the `Amazon.Lambda.Serialization.Json` package and added a reference to my custom shim with the same name. I had to duck type the interface as it used by `Amazon.Lambda.Core` and I didn't want to have to peal the onion back any more so settled on this. Here is my `.csproj` for the Lambda, targets .NET Core 3.0 and has the reference to the shim library.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <AWSProjectType>Lambda</AWSProjectType>
  </PropertyGroup>
  <ItemGroup>
    <Content Include="bootstrap">
      <CopyToOutputDirectory>Always</CopyToOutputDirectory>
    </Content>
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Amazon.Lambda.RuntimeSupport" Version="1.0.0" />
    <PackageReference Include="Amazon.Lambda.Core" Version="1.1.0" />
        <PackageReference Include="Amazon.Lambda.ApplicationLoadBalancerEvents" Version="1.0.0" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\Amazon.Lambda.Serialization.Json\Amazon.Lambda.Serialization.Json.csproj" />
  </ItemGroup>
</Project>
```

I ran into some issues with the initial implementation due to the JSON data structures having the incorrect case! Event responses are *strict* so I had to made the correct response otherwise ALB would return `502 Bad Gateway`.

To solve this quickly I ended up creating a custom class for the response and decorated the properties with the correct names.

```csharp
[JsonPropertyName("statusDescription")]
public string StatusDescription { get; set; }
```

Awesome! The ALB is serving requests and we can now benchmark our functions.

### Crickets

I've used [locust.io](https://locust.io) in the past so that was my choice again here. I built a super simple load test, that hit both ALB paths with 1000 users for 1 minute.

```python
locust -f ./net30-native/test.py --no-web -c 1000 -r 100 --host=$host --csv=net30-native --run-time 1m
```

I wanted to look at calls that were over 100ms as anything under gets rounded to 100. So I added an additional function that builds some random data into the response payload, otherwise the function is just returning a string and not actually doing much work.

Method | Name | # Reqs | # Fail | Median | Average | Min | Max | Requests/s 
--- | --- | --- | --- | --- | --- | --- | --- | ---
GET | /lambda/net30-native | 19019 | 0 | 1200 | 1235 | 240 | 3937 | 307.27
GET | /lambda/net30-newton | 18520 | 0 | 1200 | 1253 | 249 | 3999 | 305.23

So Net30.Native is the function using new Microsoft library and Net30.Netwon is the Json.NET library, but on the face of it there is not that much in it when I comes to requests.

So what about from a Lambda perspective? Will take the ALB out of the equation and just look at any improvements we are getting on the function duration. The important thing here is `Billed Duration` and for this we need CloudWatch Insights.

I used the following query:
```sql
filter @type = "REPORT" |
fields @requestId, @billedDuration |
stats avg(@billedDuration) as AVG,
min(@billedDuration) as Min,
max(@billedDuration) as Max,
count(min(@billedDuration)) as Count,
sum(@billedDuration) as Total,
count(@requestId) as Req
```

[<img src="{{ site.baseurl }}/images/2019-07-03-lambda-dotnet-json/native.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-07-03-lambda-dotnet-json/native.png")

[<img src="{{ site.baseurl }}/images/2019-07-03-lambda-dotnet-json/newton.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2019-07-03-lambda-dotnet-json/newton.png")

Name | Avg  | Min | Max | Under 100ms | Total Duration | Total Req
--- | --- | --- | --- | --- | --- | ---
Net30.Native | 162.6017 | 100 | 2600 | 12 | 3217400 | 19787
Net30.Newton | 265.0536 | 100 | 2600 | 13 | 5121100 | 19321

Look at that! 

5,121,100 vs 3,217,400
1,903,700 less!
Which works out at ~37% reduction in cost!