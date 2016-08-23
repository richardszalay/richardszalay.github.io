---
layout: post
title: 'Creating a Build-Once iOS Deployment Pipeline: Part 1 - Introduction'
date: 2016-08-23 08:15:00.000000000 +10:00
type: post
status: draft
categories: []
tags: []
meta:
  _edit_last: '6536909'
author:
  login: richardszalay
  email: richard@richardszalay.com
  display_name: Richard Szalay
  first_name: ''
  last_name: ''
---

Let's get this out of the way: I'm not an iOS developer. In fact, I'm not a mobile developer at all. Nevertheless, I found myself needing to optimise our iOS build, test, and release process for speed and consistency. This series is the resulting output.

The series will no doubt be wordy so, for your convenience, here's a tl;dr guide:

* Part 1 - Introduction, and concept/tools primer
* [Part 2 - Deploy pipeline design]({% post_url 2016-08-23-ios-deploy-pipeline-2-pipeline-architecture %})
* [Part 3 - Build server setup and build/test phases]({% post_url 2016-08-23-ios-deploy-pipeline-3-setup-and-build-test-phase %})
* [Part 4 - Deploy phase]({% post_url 2016-08-23-ios-deploy-pipeline-4-deploy-phase %})

Finally, the series was mostly written on my phone while holding my 7-month-old, so apologies in advance for any spelling mistakes.

## Deployment Pipelines

Deployment pipelines are an automated process through which code goes from development (here, Xcode) to production (the App Store). They are a major tenant of Continuous Delivery, which I strongly encourage you learn more about by reading [the Humble/Farley book of the same name](https://www.amazon.com/Continuous-Delivery-Deployment-Automation-Addison-Wesley/dp/0321601912).

As a logical extension of Continuous Integration, Continuous Delivery calls for the entire pipeline to occur on a non-development machine in order to keep the configuration known and reproducible.

Continuous Delivery also strongly recommends that application binaries are only built once and deployed to each target configuration/environment. There's a number of good reasons for this:

* Consistency - the application that was tested in one environment is guaranteed to be the same when it deployed into production, even if the build configuration (read: Xcode version). Once you’ve been doing this long enough, the idea of "building for production" is terrifying.
* Performance - build processes can be slow, and the Swift compiler, to put it kindly, is no speed demon. Paying that cost only once per "version", with a much shorter release time, let's you iterate faster. If the app is also white labelled, the benefits compound even sooner. 

## Why this series?

If the above sounds like a good idea, it would make sense that there would be existing tools and documentation to make this happen but unfortunately that's not the case. A significant amount of amazing work has been done around automation of iOS builds, but there's effectively nothing on having a single-build pipeline.

This series hopes to provide information on how to configure a pipeline that follows the tenants of a Continuous Delivery-style deployment pipeline.

## What's included in this series?

This series intends to describe the tools and archictecture of a deployment pipeline that builds only once and applies configuration at deploy-time, which is useful for both white labelled applications and those with numerous target API environments or unique client "keys".

What this series won't be covering is recommendations on how to write application code, unit tests, or on dependency management using Cocoapods/Carthage. There's plenty of information around about those topics, written by people far more qualified to write about them than I.

## Concept and Tools Primer

### The Apple Development Ecosystem

It's important to have a reasonable understanding of the world of iOS application signing and distribution before we go any further. I strongly recommend you read ["Maintaining Identifiers, Devices, and Profiles"](https://developer.apple.com/library/ios/documentation/IDEs/Conceptual/AppDistributionGuide/MaintainingProfiles/MaintainingProfiles.html#//apple_ref/doc/uid/TP40012582-CH30-SW1) section of the iOS App Distribution Guide in the Apple developer documentation, but here's a basic rundown:

First up, Apple has two management areas: **The Developer Portal** allows you to manage everything to do with code signing, and **iTunes Connect** allows you to upload (signed) builds, attach metadata like screenshots and pricing, and submit for review. You can be registered against more than one Team, and be able to manage each teams' apps.

> **Gotcha #1 (of many):** Apple Developer teams and iTunes Teams have different IDs when it comes to automation. The former's ID is plastered everywhere and uses numbers and letters, the latter is numeric and is not easy to find. More on this later.

Apple supports three primary method's of distributing an app: the **App Store**, where it can be downloaded by the world and make you a millionaire;  **AdHoc**, via a private distribution channel to a set number of registered devices; and for **Development** on your own personal devices. (there's actually also Enterprise, which is somewhere between AdHoc and a private App Store). 

A **Signing Identity** is a way of guaranteeing that your application was actually made by you, and is required for all forms of distribution. It consists of a **Certificate** and **Private Key**, which are somewhat analagous to the wax seals of yore, whereby the recognizable symbol is your public key and the stamp that presses the wax is the private key. These were used to prove that a letter came from a King (or at least was approved by him), and the concept is somewhat similar here. 

> **Gotcha #2:** Private keys are only available for download when the certificate is first created. Keep then safe!

There are two types of Signing Identity that relate to this series: Development, which can only be used for debugging on your own devices, and Distribution, which are used for both AdHoc and App Store distribution.

> **Gotcha #3:** A team can only have 3 distribution certificates (identities) active at any one time.

An **App (Bundle) Identifier** is a unique ID for an application, usually a "reverse domain" like com.richardszalay.appname. It's made universally unique by prepending your (portal) team ID.

A **Provisioning Profile** provides a way of approving "how" a signed app is used, and are distributed as part of the app. They are linked to both a signing identity and an app bundle identifier, though they support wildcards for the latter. Their usage for Development and App Store distributions is straight forward, but for AdHoc they contain the list of Unique Device Identifiers (UDID) of devices' that can install the app. For other devices, the app will simply refuse to install. 

> **Gotcha #4:** Obtaining a UDID from a device is quite an ordeal, and generally requires connecting the device to iTunes. Some beta testing services, like  HockeyApp, manage it via an app with special permissions.

### The XCode Build Process 

Contrasted to the ecosystem, the build tools are relatively straight forward and all ship either with XCode or as CLI Tools.

**xcode-select** allows you to change the default ("system") location of Xcode for command line usage. Using this tool requires sudo access, but setting the DEVELOPER_DIR environment variable is a non-Audi alternative that has the same effect.

**xcodebuild** builds, and optionally signs, an xcproj/xcworkspace and archives it into an xcarchive package directory.

**xcrun** invokes commands for a particular sdk version, and is used in this flow for exporting an xcarchive as an IPA (iPhone Application Archive), which is a single binary file that can be distributed via the App Store or AdHoc. It's actually just a zip file, but it's structure is important.

### Fastlane

[Fastlane](https://fastlane.tools/) attempts to simplify the above tools and processes by unifying them under a set of ruby-friendly actions, adding automation to previously manual web portals and smoothing out the rough edges of the command line tool experience.

Fastlane has been around for some time and has its own share of quirks and seemingly conflicting advice, so I'll outline a few of its main tools here:

**Gym** builds xcarchives and ipas, wrapping the build and signing processes of xcodebuild and xcrun, and works around the shortcomings of those tools by adding back features that are currently exclusive to the Xcode IDE, like support for Swift and Apple Watch.

**Scan** builds and runs unit tests on the simulator or a physical device.

**Cert** accesses and validates signing identities in the Developer Portal and stores them in your local keychain, creating a new one if required. 

**Sigh** functions similarly Cert but for provisioning profiles. It also includes a tool for resigning an existing ipa with a different signing identity / provisioning profile.

**Match** is the newer, and recommend, alternative to both Cert and Sigh. Instead of using the Developer Portal as the source of truth for signing identitues and provisioning profiles, it uses a git repository. The thinking behind the process is outlined at https://codesigning.guide/. Sigh and Cert can both still be useful for advanced scenarios, but most of the time you won't need them.

**Deliver** uploads a new app version into iTunes. It can either use local metadata and screenshots to automatically submit for review, or you can add that manually yourself.

There's also numerous other tools and plugins to do everything from talking screenshots to integration with services line HockeyApp (for beta testing) and Sentry (for error reporting).

In [Part 2]({% post_url 2016-08-23-ios-deploy-pipeline-2-pipeline-architecture %}) we'll design our pipeline around these technologies.
