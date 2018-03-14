---
layout: post
title: 'Improving Sitecore Commerce exception logging with Demystifier'
type: post
status: draft
tags:
- sitecore
- experience-commerce-9
meta:
  _edit_last: '6536909'
author:
  login: richardszalay
  email: richard@richardszalay.com
  display_name: Richard Szalay
  first_name: ''
  last_name: ''
---

Sitecore Experience Commerce's "Engine" microservice makes heavy use of asynchronous programming in everything from API controllers and commands to pipelines and data access. While this makes the most out of your server resources by minimising thread contention, "with great power comes terrible stack traces".

<pre><code>System.Exception: Error processing block: Core.block.FindEntityInMemoryCache ---> System.NullReferenceException: Object reference not set to an instance of an object.
   at Sitecore.Commerce.Core.Caching.EntityMemoryCachingPolicy.GetCachePolicy(CommerceContext commerceContext, Type arg)
   at Sitecore.Commerce.Core.FindEntityInMemoryCacheBlock.&lt;Run>d__2.MoveNext()
--- End of stack trace from previous location where exception was thrown ---
   at System.Runtime.ExceptionServices.ExceptionDispatchInfo.Throw()
   at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   at Sitecore.Framework.Pipelines.ReflectionPipelineBlockRunner.&lt;InvokeBlock>d__2.MoveNext()
   --- End of inner exception stack trace ---
   at Sitecore.Framework.Pipelines.ReflectionPipelineBlockRunner.&lt;InvokeBlock>d__2.MoveNext()
--- End of stack trace from previous location where exception was thrown ---
   at System.Runtime.ExceptionServices.ExceptionDispatchInfo.Throw()
   at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
   at Sitecore.Framework.Pipelines.BasePipelineBlockRunner.&lt;Run>d__3`1.MoveNext()</code></pre>

The essence of the exception is shrouded by implementation details of the async/await process, making it very difficult to undersatnd.

Ben Adams decided to tackle this problem head on a few months back and created [Demystifier](https://github.com/benaadams/Ben.Demystifier), which transforms known stack frame patterns into something that resembles the original code. The concept has been so popular that it has been [merged into .NET Core 2.1](https://www.ageofascent.com/2018/01/26/stack-trace-for-exceptions-in-dotnet-core-2.1/) and will likely make it into a future release of the .NET Framework.

For the time being, though, beautiful stack trackes require a little work. Thankfully, Nicholas Blumardt has developed [Serilog integration for Demystifier](https://github.com/nblumhardt/serilog-enrichers-demystify) that reduces the integration process to a few steps.

First, we need to add the `Serilog.Enrichers.Demystify` package to our `Sitecore.Commerce.Engine` project:

{% highlight powershell %}Install-Package Serilog.Enrichers.Demystify -IncludePrerelease{% endhighlight %}

Next we need to update the `LoggerConfiguration` (included in the Startup constructor in the default Commerce SDK) to use the enricher:

{% highlight csharp %}
Log.Logger = new LoggerConfiguration()
  .ReadFrom.Configuration(this.Configuration)
  .Enrich.FromLogContext()
  .Enrich.WithDemystifiedStackTraces() // Add this line
  .Enrich.With(new ScLogEnricher())
{% endhighlight %}

That's it! The next time you see an exception in your log, it will look something like this:

<pre><code>System.Exception: Error processing block: Core.block.FindEntityInMemoryCache ---> System.NullReferenceException: Object reference not set to an instance of an object.
   at EntityMemoryCachingPolicy Sitecore.Commerce.Core.Caching.EntityMemoryCachingPolicy.GetCachePolicy(CommerceContext commerceContext, Type arg)
   at async Task&lt;FindEntityArgument> Sitecore.Commerce.Core.FindEntityInMemoryCacheBlock.Run(FindEntityArgument arg, CommercePipelineExecutionContext context)
   at async Task&lt;object> Sitecore.Framework.Pipelines.ReflectionPipelineBlockRunner.InvokeBlock(IPipelineBlock block, IPipelineExecutionContext context, object current)
   --- End of inner exception stack trace ---
   at async Task&lt;object> Sitecore.Framework.Pipelines.ReflectionPipelineBlockRunner.InvokeBlock(IPipelineBlock block, IPipelineExecutionContext context, object current)
   at async Task<TOutput> Sitecore.Framework.Pipelines.BasePipelineBlockRunner.Run<TOutput>(string name, object input, IPipelineExecutionContext context)</pre></code>


