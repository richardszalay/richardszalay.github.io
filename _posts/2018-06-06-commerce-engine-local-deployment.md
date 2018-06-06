---
layout: post
title: 'Continuous local deployments for Sitecore Commerce Engine'
date: 2018-06-06 22:20:16.000000000 +11:00
type: post
published: true
status: publish
tags:
- sitecore
- experience-commerce-9
author:
  login: richardszalay
  email: richard@richardszalay.com
  display_name: Richard Szalay
  first_name: ''
  last_name: ''
---

Commerce Engine's default configuration poses some interesting problems for deployment, especially for local development workflow:

1. It has four roles (Ops, Shop, Authoring, and Minions) that each run as their own site/application. Each has their own "startup" configuration, but otherwise load runtime configuration from the database (put their by "Bootstrapping" the Ops role)
2. ASP.NET Core doesn't support assmebly shadow copies, so the assembly files are locked while the roles are running.

In this post I'll run through the tricks I use to simplify local development workflow without causing problems for your downstream CI/CD that can come from a custom "file copy" approach in local development.

By the end of this post we'll have continuous local deployment triggered on build without affecting build performance, as well as providing an easy way to debug the minion role.

## Publish profile

To start off, we'll create a publish profile using the `FileSystem` publish method. The target will be the ops role, for reasons that will be explained in the next section, but is otherwise default configuration:

```xml
<!-- Local.pubxml -->
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <WebPublishMethod>FileSystem</WebPublishMethod>
    <LastUsedBuildConfiguration>Release</LastUsedBuildConfiguration>
    <LastUsedPlatform>Any CPU</LastUsedPlatform>
    <PublishFramework>net452</PublishFramework>
    <UsePowerShell>True</UsePowerShell>
    <publishUrl>C:\inetpub\wwwroot\CommerceOps_Sc9</publishUrl>
    <DeleteExistingFiles>False</DeleteExistingFiles>
    <EnableMSDeployAppOffline>true</EnableMSDeployAppOffline>
  </PropertyGroup>
</Project>
```

## Sharing assemblies

The first challenge is the four role directories. Publishing four times _will_ increase publishing time in a noticable way, especilly if the aim is to do it on each build.

While it may be tempting to consolidate everything into a single role for development, this can actually complicate your downstream deployment process because your environment policies won't correlate. Worse, it means that your minion role can't be isolated for debugging because it's part of the authoring role.

The approach I have taken is to configure the four roles to all use the assemblies from the Ops role's directory, which is done by changing `aspNetCore/@processPath` in Web.config to a relative path:

```xml
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
    </handlers>
    <modules runAllManagedModulesForAllRequests="false">
      <remove name="WebDAVModule" />
    </modules>
    <aspNetCore processPath="..\CommerceOps_Sc9\Sitecore.Commerce.Engine.exe" arguments="" forwardWindowsAuthToken="false" stdoutLogEnabled="false" requestTimeout="00:10:00" stdoutLogFile=".\logs\stdout" />
  </system.webServer>
</configuration>
```

This approach has two caveats to be aware of: 

Each of the role directories still need a `bootstrap\Global.json` and `config.json`. I currently consider these an "initial setup task", but I may eventually try to deploy/configure them using custom targets in the publish profile.

Correlating `Sitecore.Commerce.Engine.exe` processes to their role becomes more difficult. In reality, though, it will usually only be the authoring role that requires being attached to so there will only be one process to choose from.

## Unlocking Assemblies

Continuous deployment is pointless if it can't overwrite any files, and ASP.NET Core doesn't support shadow copies. Luckily, IIS has a trick up it's sleeve: the `app_offline.html` file. It's existence shuts down the application pool and rejects all requests (with a 503 status) until the file is deleted.

In our scenario, we'll need four separate app_offline files (one for each site), but that's easily achievable by hooking into the "Before" and "After" events of the publish. The code below should be placed in `Properties\PublishProfiles\Local.wpp.targets` (or directly in `Local.pubxml` if you prefer):

```xml
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <!-- Takes the engine offline so we can override it's assemblies -->
  <ItemGroup>
    <CommerceEngineNodes Include="C:\inetpub\wwwroot\CommerceAuthoring_Sc9" />
    <CommerceEngineNodes Include="C:\inetpub\wwwroot\CommerceMinions_Sc9" />
    <CommerceEngineNodes Include="C:\inetpub\wwwroot\CommerceShops_Sc9" />
  </ItemGroup>
  
  <Target Name="BeforePublish">
    <WriteLinesToFile Lines="" File="%(CommerceEngineNodes.FullPath)\app_offline.htm" />
  </Target>

  <Target Name="AfterPublish">
    <Delete Files="@(CommerceEngineNodes -> '%(FullPath)\app_offline.htm')" />
  </Target>
</Project>
```

(The Ops role is the target of the deployment, so the use of `EnableMSDeployAppOffline` means we don't need to create an `app_offline.html` file there explicitly)

## Configure continuous deployment

Now that we have the means to easily update our four applications, the next logical step is to automatically deploy the files whenever we build.

By placing the following in your engine csproj (or `Directory.Build.props` if using VS2017), the project will be published to your local instances whenever you build in "Debug" from within Visual Studio. The impact on build time is negligable because unlike right click :: Publish it doesn't cause a rebuild, so it's win-win!

```xml
<Project>
  <!-- Automatically publish to Properties/PublishProfiles/Local.pubxml on build -->
  <PropertyGroup Condition="'$(Configuration)' == 'Debug' and '$(BuildingInsideVisualStudio)' == 'true'">
    <DeployOnBuild>True</DeployOnBuild>
    <PublishProfile>Local</PublishProfile>
  </PropertyGroup>
</Project>
```

This trick only works for "sdk-style" projects, like those used ASP.NET Core, but there's a similar approach for legacy web projects is outlined in [helix-publishing-pipeline](https://github.com/richardszalay/helix-publishing-pipeline)

## Debugging

Commerce Engine's architecture lends itself to test-driven development, and I highly recommend doing it if you're not already. Having said that, there's always going to be the need for interactive debugging to confirm any assumptions made about what data will be available at any given time.

For the version of config.json and Global.json that you commit into source control (ie. part of your Engine project source), I recommend setting them up to run as your Minion environment. I find myself needing to "F5" into the Minion role far more often than any other role. In fact, I tend to disable the Minion app pool in development unless there's a reason for it to run in the background.

For authoring, I currenly resort to checking the Application event log for the entry that correlates the IIS AppPool with the process id. Awkward, but it works.

## Bonus: Bootstrapping

Bootstrapping as part of continuous _local_ deployments is unlikely to be worth the time it takes to fire up the app pool, but if you're looking to easily bootstrap without using going through Postman I've created a PowerShell script that defaults all it's parameters to local development defaults. Check it out here: [https://gist.github.com/richardszalay/6f03d8898bf524d4ea26ba99333e80e6](https://gist.github.com/richardszalay/6f03d8898bf524d4ea26ba99333e80e6)