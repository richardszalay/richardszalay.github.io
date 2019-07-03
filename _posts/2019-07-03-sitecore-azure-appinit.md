---
layout: post
title: "Sitecore App Service Warm-Up Demystified"
date: 2019-07-03 16:35:16.000000000 +11:00
type: post
published: true
status: publish
tags:
- sitecore
- azure
- performance
author:
  login: richardszalay
  email: richard@richardszalay.com
  display_name: Richard Szalay
  first_name: ''
  last_name: ''
---

It's well known the [Application Initialization](https://docs.microsoft.com/en-us/iis/configuration/system.webserver/applicationinitialization/) IIS module can assist in "warming up" an application. The problem this feature has always faced (for me, at least) is that its integration with Azure is not particularly well documented, and mostly limited to occasional forum posts that indicate deeper integration with the App Service load balancer without providing the detail.

That is, until now. Microsoft was kind enough to direct me to [App Service Warm-Up Demystified](https://michaelcandido.com/app-service-warm-up-demystified/) by Michael Candido, a Principal Software Engineering Manager on the App Service team. This blog post goes into an incredible amount of detail about how the feature works in App Service. I'm not going to try to reproduce Michael's blog post here, and I do consider it required reading if using this feature, but I'd like to summarise the feature's behaviour in areas that directly affect Sitecore.

# tl;dr

To reduce/remove any "cold" start delays to end users:

* Use deployment slots for blue/green deployments
* Use Application Initialization to warm up a specific resources for scale and maintenance events
* Use Advanced Application Restart to restart the application

Things to be aware of when using Application Initialization:

* Make sure your warmup calls can actually reach the target Sitecore "site" using specific hostName and/or URL rewrite rules
* Prevent caching of unintended absolute URLs (eg. "http://localhost") by using relative URLs or setting the `targetHostname` and `scheme` properties on the target site
* Make sure your application can warm up (including app init URLs) in under 10 minutes

# Overview

Generally speaking, the Application Initialization module simply requests a number of URLs sequentially when the application pool starts. It does so in the background, without interfering with traditional requests.

Azure App Service, however, provides additional support for this feature by preventing instances from receiving traffic until after all of the initialization URLs have been loaded sequentially.

This prevention applies in the following scenarios:

* Adding new instances via scale _out_ (either manually or autoscale)
* Swapping deployments slots
* Instances being taken in and out for maintenance
* Advanced Application Restart (in "Diagnose and Solve Problems")

It does __not__ prevent requests from being sent to an instance in the following scenarios:

* Deployments (use deployment slots)
* Scaling _up_ instances (there's a [feature request for this](https://feedback.azure.com/forums/169385-web-apps/suggestions/33580975-add-application-initialization-support-for-scale-u))
* Restarting your application via the "Restart" option (use Advanced Application Restart instead)
* Applications not running in Azure App Service (on-prem, Azure IaaS, etc)

# Sitecore integration

There are a number of things to be aware of when using Application Initialization with Sitecore:

1. _Any_ response from an initialization URL is sufficient, even 302 (Redirect) or 500 (Error)
2. All app initialization requests are sent using "localhost" unless configured otherwise
3. All app initialization requests are sent using HTTP (ie. not HTTPS)
4. The instance will be torn down if the application doesn't restart in 10 minute (30 in some circumstances)

If not planned for, these nuances can result in instances receiving traffic too early because all of the requests resulted in not-found errors or redirects without actually warming anything up.

Let's look at these issues and how you can work around them.

## Localhost

Assuming your target site isn't configured to match `localhost`, the request will likely result in a 404 (or a 302 to the Not Found page). There are a number of things to consider when fixing that:

1. Whether to target the site via the host header or override the site via `?sc_site`
2. Whether to configure the target site on each initialization URL, or 'globally' via URL Rewrite

My personal preference is to rewrite `?sc_site`, since it's portable between environments, using either Rewrite rule if there's only one target site, or on each warmup URL if there are multiple sites being.

However, since your requirements may necessitate any combination of these options, I'll provide configuration samples for each.

### Using `<applicationInitialization>`

```xml
<system.webServer>
  <applicationInitialization doAppInitAfterRestart="true">
    <!-- Via site name -->
    <add initializationPage="/?sc_site=veryexcellent" />

    <!-- Via host name -->
    <add initializationPage="/" hostName="veryexcellentsite.com" />
  </applicationInitialization>
</system.webServer>
```

### Using `<rules>`

**NOTE:** Modifying the host using a rewrite rule requires that you add `HTTP_HOST` to `system.webServer/rewrite/allowedServerVariables` via an [applicationHost.xdt transform](https://github.com/projectkudu/kudu/wiki/Xdt-transform-samples)

Remember that rules are processed in order, so this will need to be first.

```xml
<rules>
  <!-- Via site name -->
  <rule name="Change warmup site" stopProcessing="true">
    <match url="(.*)" />
    <conditions>
      <add input="{APP_WARMING_UP}" pattern="1" />
    </conditions>
    <action type="Rewrite" url="{R:0}?sc_site=veryexcellent" appendQueryString="true" />
  </rule>

  <!-- Via host name -->
  <rule name="Change warmup host" stopProcessing="true">
    <match url=".*" />
    <conditions>
      <add input="{APP_WARMING_UP}" pattern="1" />
    </conditions>
    <serverVariables>
      <set name="HTTP_HOST" value="veryexcellentsite.com" />
    </serverVariables>
  </rule>
</rules>
```


## HTTP

HTTPS can't be rewritten via a rule, but if you have an HTTP->HTTPS redirect you'll need to inject an earlier rule that allows the warmup request to go through unsecured (the 302 won't be followed, and will be treated as a success response).

NOTE: If you're using a rule to apply the site/host, you can simply set `stopProcessing="true"` on that and this rule won't be required.

```xml
<rule name="No HTTP->HTTPS" stopProcessing="true">
  <match url=".*" />
    <conditions>
      <add input="{APP_WARMING_UP}" pattern="1" />
    </conditions>
  <action type="None" />
</rule>
```

## Cached Renderings with Absolute Links

If you have any cached renderings that use absolute links and you use `AlwaysIncludeServerUrl` (either in those renderings, or globally), make sure you set the following in your site configuration so that your links aren't cached as "http://localhost/*":

* `targetHostname`
* `scheme` (if using https)

## Time limit

Finally, the application has 10 minutes to complete warmup before the VM is shut down. This might be as long as 30 minutes, but as it's not reliable it's best to aim for 10 minutes.

I won't cover startup optimisation as it's a large topic on its own, but here's two easy things that can help:

* Ensure the site is configured to use Roslyn ([Microsoft.CodeDom.Providers.DotNetCompilerPlatform](https://www.nuget.org/packages/Microsoft.CodeDom.Providers.DotNetCompilerPlatform)) as the cshtml compiler rather than the default. **I've seen startup time drop by 3-4 minutes doing this**
* Precompile views (for example, using [RazorGenerator.MsBuild](https://github.com/RazorGenerator/RazorGenerator/wiki/Using-RazorGenerator.MsBuild)) and [registering the assembly with Sitecore](https://kamsar.net/index.php/2016/09/Precompiled-Views-with-Sitecore-8-2/). Should should drop another 30-120 seconds from the startup time.