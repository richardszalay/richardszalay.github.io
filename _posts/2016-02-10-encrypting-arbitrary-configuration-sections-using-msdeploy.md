---
layout: post
title: Encrypting Arbitrary Configuration Sections using MSDeploy
date: 2016-02-10 14:49:45.000000000 +10:00
categories:
- knowledge-base
tags:
- msdeploy
- encryption
- configuration
published: true
status: publish
---

Web Deploy 3.5 added the EncryptWebConfig for encrypting connection strings in Web.config files. While great, the feature 
only works with connectionStrings and doesn't support configSource.

To workaround this, we can use the `runCommand` provider to invoke "aspnet_regiis.exe" directly:

{% highlight shell %}
msdeploy.exe -postSync:runCommand="c:\windows\Microsoft.NET\Framework64\v4.0.30319\aspnet_regiis.exe 
	-pd system.net/mailSettings/smtp 
	-site awesomesite.com 
	-app /"
{% endhighlight %}

This leaves us with a problem: non-administrator deploy users don't have sufficient permission to encrypt configuration sections. And we're 
all using non-administrator deploy users, right?

### Configuring non-administrator deployments

Assuming non-administrator deployments are already configured, the following changes will allow the user to encrypt configuration sections:

1. Create a delegation rule to allow the deploy user to invoke aspnet_regiis.exe
    1. In IIS Manager, go to "Management service Delegation"
    2. Create a new, blank rule with the following settings:
        * **Provider**: runCommand
        * **User**: CurrentUser
        * **RegularExpression**: aspnet_regiis.exe
2. Grant the deploy user read access to %windir%\system32\inetsrv\config (ignore the errors)
3. Add the deploy user to the "Replace a process level token" group policy (Group Policy :: Local Policies :: User Rights Assignment)
