---
layout: post
title: 'Diagnosing missing package assemblies in Helix solutions'
---

There's a issue I've seen come up again and again when publishing Helix solutions: the assemblies from a NuGet package (usually Unicorn) referenced by a Feature/Foundation module are not included when the Project is published, causing errors during deployment or at runtime. For example:

> error CS0246: The type or namespace name 'Unicorn' could not be found (are you missing a using directive or an assembly reference?)

To understand why this happens, we need to look at how assembly resolution works during the build process. For the project being built, the `ResolveAssemblyReferences` target is used to collect all of the assembly, project, and nuget references. So far, so good.

The issue of missing assemblies comes into play when dealing with _downstream_ references, like the references of referenced projects. For these, the _output assemblies_ (not projects) are analysed for their references which are then all gathered together to be included in the building-project's output.

This works as expected for libraries like Glass, where the assemblies names and types are used directly by the project's code and so necessitates an assembly reference in that project's output. Packages like Unicorn cause a problem because in most scenarios the Unicorn assemblies will be required by config files but not actually referenced in code. This precludes them from being included as assembly references and are therefore ignored when it comes to publishing the upstream project.

Here's a crude visualisation of this process:

<img src="/assets/referenced-assembly-publishing.png" style="max-width: 600px" />

There are a number of ways this problem can be addressed:

1. Reference the package directly from the project that you are publishing
2. Have a project reference types from the package's assembly (make sure they're meaningful so they don't get optimized away)

[Helix Publishing Pipeline](https://github.com/richardszalay/helix-publishing-pipeline) actually [automatically applies](https://github.com/richardszalay/helix-publishing-pipeline/blob/master/src/targets/Helix.Publishing.Plugins/CollectReferencesFromHelixModules.targets) a third technique: extend the build process and invoke `ResolveAssemblyReferences` on all referenced projects so that their "soft" references can be included. Below is a more general purpose port of the implementation:

```xml
<!-- Add this to your pubxml or wpp.targets to have all references from referenced projects -->
<Project>
  <PropertyGroup>

    <ResolveAssemblyReferencesDependsOn>
      $(ResolveAssemblyReferencesDependsOn);
      CollectReferencesFromProjectReferences;
    </ResolveAssemblyReferencesDependsOn>

  </PropertyGroup>

  <Target Name="CollectReferencesFromProjectReferences">

    <MSBuild BuildInParallel="$(BuildInParallel)" Projects="@(ProjectReference)" RemoveProperties="DeployOnBuild;PublishProfile" Targets="ResolveAssemblyReferences">
      <Output ItemName="_ReferencesFromProjectReferences" TaskParameter="TargetOutputs"/>
    </MSBuild>

    <ItemGroup>
      <ReferencesFromProjectReferences Include="@(_ReferencesFromProjectReferences)" Condition="'%(_ReferencesFromProjectReferences.CopyLocal)'=='true'"/>

      <Reference Include="%(ReferencesFromProjectReferences.FusionName)">
        <ResolvedFrom>%(ReferencesFromProjectReferences.ResolvedFrom)</ResolvedFrom>
        <HintPath>%(ReferencesFromProjectReferences.FullPath)</HintPath>
        <Redist>%(ReferencesFromProjectReferences.Redist)</Redist>
        <CopyLocal>true</CopyLocal>
      </Reference>
    </ItemGroup>
    
  </Target>

</Project>
```
