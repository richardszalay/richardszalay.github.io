---
layout: post
title: "Upgrading Buggy Bits to .NET Core - Part 2: Windows Tooling"
date: 2017-09-08 20:20:16.000000000 +11:00
type: post
published: true
status: publish
tags:
- sitecore
author:
  login: richardszalay
  email: richard@richardszalay.com
  display_name: Richard Szalay
  first_name: ''
  last_name: ''
---

This series attempts to update Tess Ferrandez' Debugging Labs to be applicable to .NET Core so that they can continue to benefit developers everywhere as .NET Core becomes the norm.

In [Part 1](/2017/09/05/buggy-bits-core-part-1/), we upgraded the codebase to ASP.NET Core. In this part, we'll take a look at the different diagnostic tools and approaches used by the labs and how they've been changed in Windows 10.

## Load Testing

tinyget is still available via the [IIS 6.0 (!) Resource Kit Tools](https://support.microsoft.com/en-us/help/840671/the-iis-6-0-resource-kit-tools), but if you're looking for a more modern Windows CLI for HTTP load testing you might be out of luck. 

There are a number of popular cross platform HTTP loading tools available like [JMeter](http://jmeter.apache.org/), [boom](https://github.com/rakyll/boom), and [vegeta](https://github.com/tsenart/vegeta), but these require additional runtimes like Java and Go to be installed.

Luckily, the Windows Subsystem for Linux means we are no longer limited by tooling availability for Windows. If you have Bash for Windows installed, you can installing a tool like [wrk](https://github.com/wg/wrk) using `sudo apt install wrk`.

wrk in particular has a similar syntax to tinyget, so aligns well with the original labs:

```
wrk --threads 30 --duration 10 http://localhost:12345
```

## Monitoring Memory

Monitoring process memory hasn't changed terribly, but in addition to using Performance Monitor you can also view memory (private working set is your best choice) in Task Manager by right clicking on the headers of the Details tab and selcting "Select columns".

For detailed information on the GC, the situation is slightly tricker. .NET Core [does not currently support performance counters](https://github.com/dotnet/corefx/issues/3906), so there's no way to _passively_ read GC generation sizes from outside the process. The only choice you have is to attach a debugger (see below) and use SOS's `!HeapStat` command.

## User-mode Memory Dumps

The various approaches to creating memory dumps now have [official documentation](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/choosing-the-best-tool), though [ADPlus](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/adplus) is still recommended for the majority of usecases. ADPlus is largely unchanged in terms of CLI options, despite now being an executable rather than a vbs script, and still ships with the [Debugging Tools for Windows](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/index). Just keep in mind that the `-iis` argument should be replaced with `-pn dotnet.exe` or `-p <PID>` since .NET Core runs its own web server.

For crash dumps, there is now also the option of having Windows Error Reporting [generate them automatically](https://msdn.microsoft.com/en-us/library/windows/desktop/bb787181(v=vs.85).aspx).

Task Manager now has support for generating user dumps, which can be useful for hang scenarios. Simply right click any process in the "Processes" or "Details" tab and select "Create dump file" 

## Debugger

WinDBG is still the built-for-purpose application for this type of debugging, and it still ships with the [Debugging Tools for Windows](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/index). However, there's also a [complete refresh of WinDBG](https://blogs.msdn.microsoft.com/windbg/2017/08/28/new-windbg-available-in-preview/) in development that you can [currently preview via the Windows Store](https://www.microsoft.com/en-us/store/p/windbg/9pgjgd53tn86). It contains a tonne of UI improvements, including a Local/Watch window and quick access to things like threads and the callstack.

Visual Studio .NET also has reasonable native debugging support and can still load SOS, so you might want to try that if you'd prefer to stay in your warm IDE blanky.

Regardless of your choice of debugging interface, you'll still need to load the SOS module that ships with .NET Core. Once you've opened a user dump, you can load SOS by typing:

```
.loadby sos coreclr
```

(Meaning "load the 'sos' module from the same directory as the 'coreclr' module")

Fun fact: SOS stands for Son of Strike and is named after Strike, the original debugging tools used internally by Microsoft while developing .NET which at that stage was still codenamed "Lightning".

In the next part, we'll see how the tools and techniques translate to debugging .NET Core on Linux.