---
layout: post
title: 'Patching Sitecore Commerce Rules'
date: 2018-06-26 22:53:16.000000000 +11:00
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

Recently I noticed that the "Number of orders [compares to] [value]" promotion qualification wasn't working when comparing to 0, which turned out to be because it _always_ returns false when the order count is 0. Since this was clearly a bug that would eventually be fixed, I wanted to avoid adding a "new" rule as it would require some manual database patching when the fix did eventually come along.

After a little exploration, I found that it is indeed possible to replace the implementation of existing qualifications and benefits (conditions and actions), so I thought I'd share what I learned.

Sitecore Commerce uses `Sitecore.Framework.Rules` for conditions and actions, so the key is understanding how rule registration is handled in that library. Rules are configured by adding types and exclusions to the registry (`IReflectionDiscoveryConfig`), which is then later used by the `IEntityDiscoverer` to find available rules at runtime.

It turns out there are two implementation details that make it easy to swap out a rule:

1. Only rules _names_ (applied via attribute) are stored in the promotion entity, not the full type name.
2. Exclusions to the rule regitry are applied at _discovery_ rather being a _removal_ of existing configuration

With these two things in mind, all we need to do is apply the same rule name to our condition via the `EntityIdentifier` attribute:

```csharp
[EntityIdentifier("CurrentCustomerOrdersCountCondition")]
public class ZeroSupportingCurrentCustomerOrdersCountCondition : ICustomerCondition, ICondition, IMappableRuleEntity
{
    // Implementation
}
```

And then exclude the original type during configuration:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.Sitecore().Rules(rules =>
        rules.Registry(registry => registry
            .RegisterAssembly(Assembly.GetExecutingAssembly())
            .ExcludeType<CurrentCustomerOrdersCountCondition>() // We're replacing this rule
        )
    );
}
```

And that's it. Since the display text is looked up using the same `EntityIdentifier` attribute, it will even appear correctly in the Commerce Business Tools without any other changes.