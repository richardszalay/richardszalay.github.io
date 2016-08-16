---
layout: post
title: 'Creating a Build-Once iOS Deployment Pipeline: Part 4 - Deploy Phase'
date: 2016-08-14 22:21:00.000000000 +10:00
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

This is part 4 of my [Creating a Build-Once iOS Deployment Pipeline series]({% post_url 2016-08-14-ios-deploy-pipeline-1-introduction %}).

In [part 3]({% post_url 2016-08-14-ios-deploy-pipeline-3-setup-and-build-test-phase %}) we configured our build server agent and added build and test phases. Leaving signing out of out build and test phases kept them simple, but it leaves a lot of things to be solved during deployment.

At a high level, here's what our deployment lane will look like:

1. Configure the IPA by modifying it's Info.plist file.
1. Call Match to import a valid signing identity and provisioning profile from our certificate repository
1. Call Resign to resign the repository using the imported signing identity and profile
1. Deploy the IPA to HockeyApp either to be tested (QA) or to improve crash diagnostics.
1. Deploy to AppStore (exclusive to our "AppStore" environment)

## Approach

### Credentials and Secrets

In vast contrast to the build lane, deployment has a substantial number of actions that require some kind of credential or secret. Most can use the keychain, through some support alternative methods of authentication, like SSH or environment variables.

|Action|Usage|Keychain|Alternatives|
|:-----|:----|:-------|:-----------|
|Match|Certificate Repository (Git) authentication|(list-keychains)|SSH/RSA|
|Match|Password for encrypting the private keys stored in the repository|(list-keychains)|ENV["MATCH_PASSWORD"]|
|Match|Importing signing identities|`ENV['MATCH_KEYCHAIN_NAME']`|N/A|
|Sigh|Accessing signing identities|(list-keychains)|N/A|
|HockeyApp|HockeyApp API token|N/A|`ENV['FL_HOCKEY_API_TOKEN']`|
|Deliver|iTunes password for deployment|(default)|`ENV['DELIVER_PASSWORD']`|

A few things to note

Match supports an option for a specific keychain, but Sigh does not. There's also no way to avoid Keychains altogether, as even actions that accept a custom keychain tend to rely on downstream functionality that does not.
Actions that use the "default keychain" actually use the first item from keychain search list, that is the result of "security list-keychains" as opposed to "security default-keychain", so there's no need to change the default/login keychains in order to customise them.

I've highlighted which of the options we went with, but I'll explain the justification as we go.

### Configuration

I won't be making any strong recommendations on how to store your environment specific configuration, since it depends on how complex it is and how many environments there are.

In our scenario, we had a fair number of environments (4-5 white labelled app versions, 3 deployment targets each), so we stored each profile in a YAML file in our configuration repository that our deployment lane parsed. Inside, we had one section for Info.plist values, one for GoogleServices-Info.plist, and one for environment variables which specified miscellaneous fastlane configuration (like HockeyApp team ids, etc). We then had a batch file that wrapped execution of fastlane, and could invoke it numerous times based on matching profiles. Overkill for most apps, but a maintainable approach for us.

Regardless, it's important to take the time to think about configuration for your requirements, whether it be hardcoded in the fastfile or loaded from a central database at deploy time.

### Versioning

Versioning, like configuration, is largely dependent on your internal tools and processes. How semantic is your versioning scheme? What kind of auto-incrementing does your build/release server support?

The only recommendations I would make, though you are free to ignore, are:

Avoid incrementing the build number in your actual build scripts. It both requires that your CI server commits the changes (and this has write access to your repository) and will get out of sync as various developers run the build locally. If your CI server has an auto incrementing number, pass that in and patch it in during the build phase (using `act`, if you like). If you swap build servers, commit a hardcoded "seed" value that you add to the CI build number.

Avoid committing your semantic (ie. release) version to your application repository. Because it's semantic, the change in version won't be known until you decide to release, at which time you don’t want to have to start the build pipeline again. Either make use of your CI server's release versioning, or commit the value to your configuration repository.

## Implementation

Both release types need to change our environment-specific plist value, resign the app, and deploy to HockeyApp. The App Store deployment will then also deploy to the App Store using Deliver.

### Configuration

Before we resign the IPA, we want to make changes to it. IPAs are only zip files, but running through all of the unzip/plistbuddy/zip shell commands introduces noise to your deploy script and makes the intent less clear.

To avoid this, I've created a fastlane action called "act", which takes the hassle out of modifying list files inside IPAs. It also supports replacing the icon with an .iconset, but we won't be doing that here to keep this simple.

To install it, run:

```bash
bundle exec fastlane add_plugin act
bundle exec fastlane install_plugins
```

Then you can configure an ipa in your fastfile, use:

```ruby
act(
  ipa: path_to_ipa,
  plist_values: {
    ":DemoEnvironmentValue" => "QA"
  }
)
```

You may need to call `act` a number of times, as some configuration is stored in different plist files (eg. GoogleServices-Info.plist)

With our ipa configured, we need to sign it so that we can release it.

### Match

Match has a number of hurdles to overcome, since it needs to connect to your certificates repository, the developer portal, and a local keychain.

For your certificates repository, I'd recommend using ssh as it avoids having to introduce another keychain. Regardless of where you want to store your private key, in your configuration repository or on the build server, you should also make sure your known\_hosts has your git server in it to avoid the deploying locking up waiting for user input. If you need it, you can override the private key and known\_hosts location by setting this environment variable:

```bash
export GIT_SSH_COMMAND="ssh -i path/to/id_rsa_git"
```

Next, match is going to validate the certificate against the developer portal. These can, and probably should, be set using the `DELIVER_USERNAME` and `DELIVER_PASSWORD` environment variables, since that can likely be configured securely on your CI.

Finally, Match will import the signing identity into the first keychain in the search list. Attempting to keep that keychain around would only introducing "source of truth" issues, so I've written a Gem named spare_keys to help. Install it either global to your Gemfile and then use it like this:

```ruby
SpareKeys.temp_keychain(true) { |keychain_path|
  # our temp kecyhain, stored at keychain_path,
  # will be used by match/resign within this block.
  # Everything will be restored afterwards.
)
```

With this in place, our call to Match looks like this:

```ruby
SpareKeys.temp_keychain(true) { |keychain_path|
  match(
    type: "adhoc" # or "appstore",
    git_url: "git@server.com:path/to/certificates-repository.git",
    app_identifier: "com.richardszalay.deploydemo",
    keychain_name: keychain_path,
    readonly: true
  )
}
```

> **Gotcha #8:** You can ignore this error in your logs: "There are no local code signing identities found".  We just created a fresh keychain, so its obviously empty.

I personally run match here with the read only flag, which prevents it from attempting to create new signing identities and provisioning profiles. Instead, I create a second lane called update_certs that is run mentally when required. Whether you should do this too depends entirely on how ok you are with your deployment processes making side effects of this kind.



> **Gotcha #9:** Match will create a provisioning if one doesn’t exist in the repository, and will do so by calling Sigh with the force flag. This will automatically add all devices to it. If your AdHoc release service  can't be solely relied upon to restrict access between owners of registered devices, you might need to handle this yourself (or file an issue with fastlane).

### Resign

Now that we have a valid signing identity and provisioning profile, the call to resign *should* be pretty basic. Unfortunately resign accepts provisioning profile options using a hash that's not compatible with the environment variables set by match. To work around this, I use this utility function:

```ruby
# Creates a hash, suitable for resign's provisioning_profile option, from
# the environment variable set by match
def match_result_provisioning_profile(app_identifier, match_type)
  sigh_profile_key = "sigh_#{app_identifier}_#{match_type}"

  return {
      app_identifier => File.join(FastlaneCore::ProvisioningProfile.profiles_path, "#{ENV[sigh_profile_key]}.mobileprovision")
  }
end
```

With that last quibble out of the way, we can call resign within our temp keychain scope:

```ruby
SpareKeys.temp_keychain(true) { |keychain_path|
  match_type = "adhoc" # or "appstore"
  app_identifier = "com.richardszalay.deploydemo"

  match(
    type: match_type,
    git_url: "git@server.com:path/to/certificates-repository.git",
    app_identifier: app_identifier,
    keychain_name: keychain_path,
    readonly: true
  )

  resign(
    ipa: "fastlane/build/DeploymentPipeline.ipa",
    provisioning_profile: match_result_provisioning_profile(app_identifier, match_type)
  )
}
```

And there we have it: a signed IPA, configured to a specific environment, in a fraction of the time it would have taken to rebuild it. You'll need to offset the time it took to read this wordy blog series, of course, but I'm sure you'll become net positive eventually.

### HockeyApp

From here on out, it's stock fastlane all the way.

We'll only be deploying to HockeyApp for our QA release to keep things simple, but since it provides crash analytics (assuming you've configured the SDK) you may want to upload a release as part of your appstore release, too. If you do, make sure they are declared as different apps in HockeyApp.

We're not going to auto release here, as we don't have the release notes in our configuration repository, so the release will need to be completed via the HockeyApp dashboard. However, if your workflow supports it, the HockeyApp fastlane action supports a tone of options for you to choose from.

Hocleyapp()

### AppStore

Finally will be flying to the app Store

Gotcha if you remember from part one, iTunes have different team ids from the developer portal. If the account you are using to authenticate with has access to multiple iTunes teams, you'll have to specify which via deliver's xxxx option

### Putting it all together

Here is the fully assembled Fastfile, though I've taken the liberty of refactoring it to avoid deduplication between release types. I've also included bash scripts to  

```bash
#!/bin/bash
# release-qa.sh

# https://github.com/fastlane/fastlane/issues/227
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8
```


```ruby
require 'spare_keys'

default_platform :ios

platform :ios do

  CERTIFICATES_REPOSITORY = "git@server.com:path/to/certificates-repository.git"

  QA_APP_IDENTIFIER = "com.richardszalay.demo.qa"
  APPSTORE_APP_IDENTIFIER = "com.richardszalay.demo"

  lane :update_certs do |options|

    match(
      type: "adhoc",
      git_url: CERTIFICATES_REPOSITORY,
      app_identifier: QA_APP_IDENTIFIER,
      readonly: false
    )

    match(
      type: "appstore",
      git_url: CERTIFICATES_REPOSITORY,
      app_identifier: APPSTORE_APP_IDENTIFIER,
      readonly: false
    )

  end

  desc "Releases a build to HockeyApp. Requires that FL_HOCKEY_API_TOKEN is set."
  lane :release_hockey do |options|
    release(
      ipa: options[:ipa],
      app_identifier: QA_APP_IDENTIFIER,
      match_type: "adhoc",
      demo_environment_value: "QA!"
    )
  end

  desc "Releases a build to iTunes. Requires that FL_HOCKEY_API_TOKEN, DELIVER_USERNAME, and DELIVER_PASSWORD are set."
  lane :release_appstore do |options|
    release(
      ipa: options[:ipa],
      app_identifier: APPSTORE_APP_IDENTIFIER,
      match_type: "appstore",
      demo_environment_value: "AppStore!"
    )
  end

  private_lane :release do |options|
    act(
      ipa: options[:ipa],
      plist_values: {
        ":DemoEnvironmentValue" => options[:demo_environment_value]
      }
    )

    SpareKeys.temp_keychain(true) { |temp_keychain_path|
      match(
        type: options[:match_type],
        git_url: CERTIFICATES_REPOSITORY,
        app_identifier: options[:app_identifier],
        keychain_name: temp_keychain_path,
        readonly: true
      )

      resign(
        ipa: options[:ipa],
        provisioning_profile: match_result_provisioning_profile(options[:app_identifier], options[:match_type])
      )
    }

    # We always release to hockey for the symbolised crash logs
    hockey(
      ipa: options[:ipa]
    )

    deliver(
      ipa: options[:ipa],

      skip_screenshots: true,
      skip_metadata: true,
      submit_for_review: false,
      automatic_release: false
    ) if options[:match_type] == "app_store"
  end

  def match_result_provisioning_profile(app_identifier, match_type)
    sigh_profile_key = "sigh_#{app_identifier}_#{match_type}"

    return {
      app_identifier => File.join(FastlaneCore::ProvisioningProfile.profiles_path, "#{ENV[sigh_profile_key]}.mobileprovision")
    }
  end
end
```

## Closing Thoughts