---
layout: post
title: "Upgrading Buggy Bits to .NET Core - Part 1: The Code"
date: 2017-09-05 21:55:12.000000000 +11:00
redirect_from: "/2017/09/25/buggy-bits-core-part-1/"
type: post
published: true
status: publish
tags:
- debugging
- dotnetcore
author:
  login: richardszalay
  email: richard@richardszalay.com
  display_name: Richard Szalay
  first_name: ''
  last_name: ''
---

It's been almost 10 years since [I first raved](https://blog.richardszalay.com/2008/02/18/advanced-debugging-lab-series/) about Tess Ferrandez' [Debugging Labs for .NET](https://blogs.msdn.microsoft.com/tess/2008/02/04/net-debugging-demos-information-and-setup-instructions/). Since then, have saved me considerable pain on at least 5 occasions and I still strongly encourage any .NET developer spend the time to go through them.

However, the labs were written for .NET 2.0 and a lot has changed since then. .NET Core has also just hit 2.0 and it poses a number of problems for those trying to follow Tess' labs: it doesn't support WebForms, and it can run on platforms other than Windows.

This series attempts to update the labs to be applicable to .NET Core so that they can continue to benefit developers everywhere as .NET Core becomes the norm. (I've spoken to Tess and she's cool with it, in case you're worried.)

For this first part I've upgraded the sample code. Since .NET Core doesn't support WebForms, the project has been upgraded to MVC.  I'm not going to focus on the upgrade process, but the new codebase follows a pattern that should make the code easy to find using the original lab series:

* Each page is now its own Controller
* The Page_Load method has been replaced with the default Index action
* Any Button_Click postback code is now an HttpPost action
* View models have been created as a vehicle to get the data to the view

I've otherwise avoided changing the original code, with the only exception being a few formatted message strings that have been updated to use [interpolated strings](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/interpolated-strings).

You can find the code on GitHub at [richardszalay/buggy-bits-core](https://github.com/richardszalay/buggy-bits-core), including instructions on how to get it running (spoilers: it's "dotnet run").

In the next part, we'll look at how the various debugging tools have changed on Windows.