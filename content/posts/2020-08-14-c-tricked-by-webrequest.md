---
url: c-tricked-by-webrequest-209014d99653
canonical_url: https://medium.com/@kevingosse/c-tricked-by-webrequest-209014d99653
title: 'Tricked by WebRequest'
subtitle: Even with years of experience, WebRequest managed to surprise me.
date: 2020-08-14
description: ""
tags:
- dotnet
author: Kevin Gosse
thumbnailImage: /images/c-tricked-by-webrequest-209014d99653-1.webp
---

The Datadog .NET tracer uses the HTTP protocol to communicate with the agent (usually installed on the same machine) and send traces every second. We originally used `HttpClient` to handle the communication. However, we ran into some dependencies errors on .NET Framework (it's a bit complicated and outside of the scope of the article, but it has something to do with us instrumenting the `System.Net.HttpClient` assembly while at the same time having a dependency on it), and so we decided to switch to `HttpWebRequest`. By doing so, we ran into a gotcha that some of you may already be aware of, but really blew my mind.

# The setup

For various reasons, such as testability, we use an abstraction layer to send the HTTP request. We manipulate a [`IApiRequestFactory`](https://github.com/DataDog/dd-trace-dotnet/blob/master/src/Datadog.Trace/Agent/IApiRequestFactory.cs) that returns instances of [`IApiRequest`](https://github.com/DataDog/dd-trace-dotnet/blob/master/src/Datadog.Trace/Agent/IApiRequest.cs), which encapsulate the HTTP call. The code of our default `IApiRequestFactory`, using `HttpWebRequest`, was as straightforward as it gets:

```csharp
internal class ApiWebRequestFactory : IApiRequestFactory
{
    public IApiRequest Create(Uri endpoint)
    {
        return new ApiWebRequest((HttpWebRequest)WebRequest.Create(endpoint));
    }
}
```

# What could possibly go wrong?

Well, thank you for asking. We shipped this code confidently, and eventually got [a bug report for a customer](https://github.com/DataDog/dd-trace-dotnet/issues/858). The tracer was logging the following exception:

```
System.InvalidCastException: Unable to cast object of type 'Services.CompositionHttpWebRequest' to type 'System.Net.HttpWebRequest'.
   at Datadog.Trace.Agent.ApiWebRequestFactory.Create(Uri endpoint)
   at Datadog.Trace.Agent.Api.<SendTracesAsync>d__9.MoveNext()
```

Where does that `Services.CompositionHttpWebRequest` comes from? It turns out it's possible to globally override the type of `WebRequest` used for any URL prefix! In this case, the customer was, for instrumentation purposes, doing something like:

```csharp
WebRequest.RegisterPrefix("http://", new CustomWebRequestFactory());
```

Which caused subsequent calls to `WebRequest.Create` to return the custom type of `WebRequest` when used with HTTP. And therefore our code would crash when innocently trying to cast it to `HttpWebRequest`.

How to fix that? Well that's when it struck me. I've always wondered about `WebRequest.CreateHttp`. I thought it was just a helper to avoid casting manually the `WebRequest` to `HttpWebRequest`, but it still seemed a bit odd to me. It turns out that it's also a way to guarantee that you'll get an `HttpWebRequest` even if somebody overrides the default factory.

Knowing that, the fix was a one-liner:

```csharp
internal class ApiWebRequestFactory : IApiRequestFactory
{
    public IApiRequest Create(Uri endpoint)
    {
        return new ApiWebRequest(WebRequest.CreateHttp(endpoint));
    }
}
```

# What about testing?

Writing a unit test was bit more challenging. I wanted my test to register a custom `WebRequest` factory, then call the `ApiWebRequestFactory` to make sure no exception was thrown. Unfortunately, I couldn't find any supported way to unregister the factory. That's a problem, because it would mean that the unit test could impact the next tests after executing. Interestingly, the source code for the `UnregisterPrefix` **does** exist [but is commented out!](https://referencesource.microsoft.com/#System/net/System/Net/WebRequest.cs,446)

{{<image classes="fancybox center" src="/images/c-tricked-by-webrequest-209014d99653-1.webp" title="I didn't think the BCL would troll me that hard" >}}

In the end, I decided to use reflection to manually unregister my custom factory at the end of the test. I did so by accessing the `WebRequest.PrefixList` static property to save the list of factories beforehand, then use the same property to restore the old value at the end of the test:

```csharp
/// <summary>
/// This test ensures that the ApiWebRequestFactory behaves correctly when
/// a different type of WebRequest is assigned to the http:// prefix
/// </summary>
[Fact]
public void OverrideHttpPrefix()
{
    // Couldn't find a way to "officially" unregister a prefix but that shouldn't stop us
    var prefixListProperty = typeof(WebRequest).GetProperty("PrefixList", BindingFlags.Static | BindingFlags.NonPublic);
    var oldPrefixList = prefixListProperty.GetValue(null);

    WebRequest.RegisterPrefix("http://", new CustomWebRequestCreator());

    // Make sure we properly hooked the WebRequest factory
    Assert.IsType<FakeWebRequest>(WebRequest.Create("http://localhost/"));

    try
    {
        var factory = new ApiWebRequestFactory();

        var request = factory.Create(new Uri("http://localhost"));

        Assert.NotNull(request);
    }
    finally
    {
        // Unregister the prefix
        prefixListProperty.SetValue(null, oldPrefixList);
    }

    // Make sure we properly restored the old WebRequest factory
    Assert.IsType<HttpWebRequest>(WebRequest.Create("http://localhost/"));
}

private class CustomWebRequestCreator : IWebRequestCreate
{
    public WebRequest Create(Uri uri)
    {
        return new FakeWebRequest();
    }
}

private class FakeWebRequest : WebRequest
{
}
```

It's funny how after all this years I can still get tricked by one of the most commonly used types of the .NET Framework.
