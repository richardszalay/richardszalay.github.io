---
layout: post
title: 'Creating a Build-Once iOS Deployment Pipeline: Part 2 - Designing the Deployment Pipeline'
date: 2016-08-14 22:19:00.000000000 +10:00
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

This is part 2 of my [Creating a Build-Once iOS Deployment Pipeline series]({% post_url 2016-08-14-ios-deploy-pipeline-1-introduction %}). 

In [part 1]({% post_url 2016-08-14-ios-deploy-pipeline-1-introduction %}), we looked at the tools and processes for building and distributing iOS apps. Let's see what that looks like as a deployment pipeline that needs to be tested in HockeyApp before being released via the App Store.

The app we'll base the rest of this series on is a basic single screen app that displays a value from the "DemoEnvironmentValue" key of the infoDictionary (Info.plist). We'll be using that as "environment specific configuration", but it could just as easily be an API base URL, or even a SQLite database used to seed data.

## Guiding tenants

The resulting 

### Security

Since freelance developers may be involved from time to time, secrets (passwords, private keys, etc) should be kept out of the application code repository. This is basic application of the principal of least privilege.

Also as all our builds, for various clients, run on the same build infrastructure, I'd like to avoid writing to (or relying on) the default keychain.

### Portability

Servers die, different projects have different dependencies, developers change (or get new machines), and documentation gets out of date. Having a build process that self-describes (and even self-obtains) its dependencies will only save headaches down the line.

### Maintainability

Separating runtime configuration from application code allows redeployment without a rebuild.

Having a single source of truth (aka Don't Repeat Yourself) for important assets means less confusion in the future, especially if the multiple copies get out of sync. 

## The Deployment Pipeline

With all of that in mind, here's the high level architecture of the pipeline that I went with:

<img src="{{ site.baseurl }}/assets/ios-build.png" alt="The proposed deployment pipeline" aria-describedby="ios-build-diagram-description" />

<div id="ios-build-diagram-description" style="display: none">
    <p>Three repositories: application code, signing identities, and configuration.</p> 

    <p>
        Four phases: 
        <ul>
            <li>Test: depends on the application code repository and generates a "test results" artifact</li>
            <li>Build: depends on the application code repository and generates an "application archive" artifact</li>
            <li>QA Deploy: depends on the signing identity and configuration repositories, as well as the "application archive" artifact. Deploys to HockeyApp.</li>
            <li>App Store Deploy: depends on the signing identity and configuration repositories, as well as the "application archive" artifact. Deploys to iTunes.</li>
        </ul>
    </p>
</div>

In the diagram above, you can see the build/test phase is very separate from the deploy phase. This means that:

* There's no access to the signing identity and configuration repositories in the build/test phase
* The code repository is not needed for the deploy phase
* There's no connection to HockeyApp / iTunes until the deploy phases

### Repositories

We use git, but any VCS would work.

**Application Code Repository** – Contains xcproj, application code, and sufficient configuration to run on a simulator or development device.

**Signing Identity Repository** – Maintained by fastlane's "match". Contains certificates, private keys, and provisioning profiles related to the team. Updated via manually executed fastlane "lanes" (tasks). We'll be setting up a task to keep this updated, but if you'd prefer to migrate your existing Signing Identities and Provisioning Profiles [Michał Laskowski has a great guide over on Macoscope](http://macoscope.com/blog/simplify-your-life-with-fastlane-match/).

**Configuration Repository** – Contains deployment scripts (fastlane) and the configuration specific to each "environment" that the app runs in, such as QA/HockeyApp or the App Store.

### Pipeline Phases

**Test Phase** - build and run unit tests in the simulator and outputs the results in junit format. 

**Build Phase** - build an IPA that is unsigned, since the signing identity will be chosen at deploy time. 

**QA Deploy Phase** - using the IPA artifact, change our plist value to "QA", sign it using an AdHoc identity+ profile, and deploy to HockeyApp for testing.

**App Store Deploy Phase** - using the IPA artifact, change our plist value to "AppStore", sign the app using the app store certificate. Deploy to HockeyApp (for crash analytics), and release a new build to iTunes.

### Secrets

The deploy phase will need an Apple Developer account, to verify signing identities and deploy to iTunes. It will also need a hockeyapp api key, and access to the certificates repository. We'll be leaving these secrets to the build server to supply.

In [Part 3]({% post_url 2016-08-14-ios-deploy-pipeline-3-setup-and-build-test-phase %}) we'll configure our build server and setup the build and test phases in fastlane.
