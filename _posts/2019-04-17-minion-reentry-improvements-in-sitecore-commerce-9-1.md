---
layout: post
title: 'Reentrancy improvements to Minions in Sitecore Commerce 9.1'
date: 2019-04-17 12:37:16.000000000 +11:00
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

**tl;dr** In XC91, you'll no longer need to worry about how long your Minion takes to run. Just override `Minion.Execute` rather than `Minion.Run` and you're good to go.

Back in March of last year, [I responded to StackExchange question](https://sitecore.stackexchange.com/a/11004/1173) on the limitations of Sitecore Commerce's Minion in terms of reentrancy. Put more plainly, in XC9.0 Minions used a "fire and forget" approach for starting each 'invocation' which resulted in Minions running twice in parallel when the previous invocation took longer to execute than the configured `WakeupInterval`.

In my answer I described two workarounds to the problem, both having a subtle impact on the behavior of Minions:

1. Detect reentrant calls using `Interlocked.CompareExchange` and skip them, which may result in an "invocation" being skipped entirely
2. Delay the WakeupInterval pause until after the invocation is complete, which moves the pause from the "start" of the invocation to the end

The recently released Commerce 9.1 aims to tackle this issue of reentrancy and the implementation actually takes a little from both of the approaches above, as well as handling some additional scenarios. Let's start with an overview of how Minions are started in XC9.0:

```
StartEnvironmentMinionsBlock
  Minion.Start()
    loop forever
      Minion.Run (fire and forget) <-- RunMinion calls this
      WakeupInterval (blocking)
```

And here's a visualisation to illustrate the problem this causes:

<figure style="max-width: 600px">
<img src="/assets/minions-reentrancy-xc90.png" />
<figcaption style="font-style: italic; text-align: center">Minion execution flow in Sitecore Commerce 9.0</figcaption>
</figure>

As you can see, XC 9.0 starts it's WakeupInterval delay from the moment Minion.Run is invoked. This is actually fine... for some configurations. The problem arises when Run takes longer than WakeupInterval, and results in Run being executed twice in parallel. If a Minion was not written with parallel execution in mind, it can cause some serious problems.

I'm happy to report that XC 9.1 fixes all of the flaws in the original design. Starting with the issue described above, Minions now apply the `WakeupInterval` delay _after_ the main method (now called `Execute`, with `Run` being marked as "obsolete") completes asynchronously:

<figure style="max-width: 600px">
<img src="/assets/minions-reentrancy-xc91.png" />
<figcaption style="font-style: italic; text-align: center">Minion execution flow in Sitecore Commerce 9.1</figcaption>
</figure>

As you can see, it's now impossible for `Execute` to be invoked twice in parallel.

The changes don't stop there, though. In XC90 it will also possible to start a minion via the `RunMinion` API while it was already running, resulting in it running twice in parallel. The new model not only prevents the same Minion from running twice, but will also skip an invocation if there is another Minion running that includes any of the values from its `MinionPolicy.Entities` array. It also doesn't matter _how_ the Minion is started: calling the `RunMinion` API will still skip execution if the Minion is currently executing on its schedule.

Here's a deeper look into the new flow:

```
StartEnvironmentMinionsBlock
  Minion.StartAsync() - called by StartEnvironmentMinionsBlock
    Warn if no WakeupInterval set
    loop forever
      await Minion.Process <-- RunMinion calls this
          Skip if policy is marked as running
          Skip if policy with any intersecting `Entities` value is running
          Mark as running
          await Minion.Execute
          Mark as not running
      await WakeupInterval
```

To summarise the behaviour changes:

1. Override Minion.Execute instead of Minion.Start
2. As before, handling exceptions is the responsibility of Minion.Execute
3. WakeupInterval now starts after Execute has completed asynchronously
4. Execute will be skipped if the minion is already running (either on a timer or via API)

