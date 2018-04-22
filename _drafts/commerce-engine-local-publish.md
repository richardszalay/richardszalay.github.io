---
layout: post
title: commerce-engine-local-publish
---

A standard Commerce Engine installation involves four individual instances: Ops, Authoring, Shop, and Minions.

While these environments don't strictly need to be separate nodes on a development machine, in reality it keeps the environment JSON consistent between development, UAT, and test installations.

In this post, I walk through a number of tips I use to simplify the deployment process. The techniques are mainly focused on development environments, but they might be applicable to test and production environments too.

## Sharing Assemblies

Since the assemblies don't differ between the four environments, we can optimise local publishing by having the Authoring, Shop, and Minion environments all use the assemblies from the Ops directory. They'll still need config.json and Global.json files, so these will need to be copied manually.

To configure the other environments to share assemblies, simply change the `processPath` in their Web.config files:

```xml
<aspNetCore processPath="..\CommerceEngine\Sitecore.Commerce.Engine.exe" arguments="" forwardWindowsAuthToken="false" stdoutLogEnabled="false" requestTimeout="00:10:00" stdoutLogFile=".\logs\stdout" />
```

Now the local publish profile can be configured to publish to the Ops web root and the rest of the environments will be updated at the same time.

The major issue with this approach is that it's difficult to determine which `Sitecore.Commerce.Engine.exe` process is which, since they all have the same executable path. In practice, though I've found that there only tends to be 2 running at any one time (Minions and Authoring) and you can disable Minions if you only run them on demand.

## Dealing with file locking

ASP.NET Core does not support shadow copying assemblies, which means they are locked while the process is runnning. If you use the assembly sharing technique above, you'll need to terminate _all four_ processes before you'll be able to deploy the updated assemblies.

To terminate the processes, we can temporarily create an `app_offline.htm` file in the root of each of the environments for the duration of the publish. Place this at the bottom of your local publish profile (`pubxml`):

```xml
<!-- Takes the engine offline so we can replace its assemblies -->
<ItemGroup>
  <CommerceEngineNodes   Include="C:\inetpub\wwwroot\CommerceAuthoring_Sc9" />
  <CommerceEngineNodes   Include="C:\inetpub\wwwroot\CommerceMinions_Sc9" />
  <CommerceEngineNodes   Include="C:\inetpub\wwwroot\CommerceOps_Sc9" />
  <CommerceEngineNodes   Include="C:\inetpub\wwwroot\CommerceShops_Sc9" />
</ItemGroup>

<Target Name="BeforePublish">
    <WriteLinesToFile Lines="" File="%(CommerceEngineNodes.FullPath)\app_offline.htm" />
</Target>

<Target Name="AfterPublish">
    <Delete Files="@(CommerceEngineNodes -> '%(FullPath)\app_offline.htm')" />
</Target>
```

## F5 Debugging

I find that having the Visual Studio project configured to "run" as the Minions role is the most useful, since the other environments will likely be debugged by attaching to their running process.

## Bootstrapping

Bootstrapping via Postman can get a little tedious, so I've development a simple PowerShell module ([gist](https://gist.github.com/richardszalay/6f03d8898bf524d4ea26ba99333e80e6)) to automate it:

```PowerShell
Import-Module .\Invoke-SitecoreCommerceBootstrap.psm1
Invoke-SitecoreCommerceBootstrap
```

The parameters default to standard development ports and passwords, but you can pass them in if they are different.

The module also has the benefit of handling CSRF being enabled, which the Postman configuration does not.