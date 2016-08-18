---
layout: post
title: 'Creating a Build-Once iOS Deployment Pipeline: Part 3 - Setup and Build/Test Phases'
date: 2016-08-14 22:20:00.000000000 +10:00
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

This is part 3 of my [Creating a Build-Once iOS Deployment Pipeline series]({% post_url 2016-08-14-ios-deploy-pipeline-1-introduction %}).
 
In [part 2]({% post_url 2016-08-14-ios-deploy-pipeline-2-the-road-ahead %}) we looked what our deployment looked like at a high level, as well as what we were trying to achieve. And now we finally start to put things together. For those that made it through the first two posts, I thank you. To those that skipped them, I don't blame you. Let's get to it. 
 
The build and test phases both require the application code repository and are actually relatively simple so we'll include them both in this part. 
 
## Setup 
 
### Build Server 

#### LaunchAgents vs LaunchDaemons

If you are configuring a build agent, you'll probably want to configure it as either a LaunchAgent or LanuchDaemon. If you need to access the 
Simulator for your automated tests, LaunchAgent is your only choice since LaunchDaemons can't access the UI. Since LaunchAgents only run when the user is logged in,
you'll probably want to configure the user to auto-login in case the server is rebooted. 

Relevant Apple documentation:

* [Creating Launch Daemons and Agents](https://developer.apple.com/library/mac/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)
* [Set your Mac to automatically log in during startup](https://support.apple.com/en-us/HT201476)

 
#### XCode Versions

Whether you have numerous applications in varying states of legacy or a new app that you'd like to test against newer tooling on a branch, you'll eventually want to make 
use of multiple versions of XCode. Apple provides downloads for every version of XCode and the Command Line Tools at the [Downloads for Apple Developers](https://developer.apple.com/download/more/) on the Developer Portal.

You can set the `DEVELOPER_DIR` directory before your build and it will use the appropriate XCode/Tooling installation.

#### Dependency Management

Both Ruby Gems and Homebrow both install packages under `/usr/local`, which your build user won't necessarily be able to write to.

TODO

For Gems, you can set the `GEM_HOME` variable to something like `~/.gems` works well. Setting this at the start of your deployment script would be ideal, just don't forget to append it to `$PATH`. 

As long as you are using a Gemfile and the `bundle exec fastlane` syntax, your Gem dependencies will remain isolated to your build. 

Homebrew dependencies, like ImageMagick (required by fastlane's badge), aren't so easy to isolate. I would still recommend using a bundler like [Brew Bundle](https://github.com/Homebrew/homebrew-bundle), but how 
you deal with being able to write to `/usr/local` is up to you. Your choice is essentially between `chown`ing the `/usr/local` directory or configuring Brew to install to a profile-local directory. 
I went with the former, but you can read [this Stack Exchange question](http://apple.stackexchange.com/questions/1393/are-my-permissions-for-usr-local-correct) and decide for yourself. 

### Fastlane 
 
I'm not going to walk through how to setup fastlane, since this is not a fastlane tutorial. We don’t need any of the default lanes, so you can delete them if you don’t need then. We also won't need any if the other config files (Matchfile, Deliverfile) though, again, you're free to keep them if they work for you. You could even skip initializing fastlane and start writing your "lanes" (tasks) yourself. 
 
One recommendation I will make is to use a Gemfile and the "bundle exec" syntax for both fastlane and any other build tools you might be using (ie. Cocoapods). These tools often break compatibility and a Gemfile allows you to lock the version local to your app, rather than relying on the globally installed version.

You may need to add some variables to your `.bashrc` to set the locale to en_US.UTF-8 to avoid any "invalid byte sequence" errors. See [fastlane/#488](https://github.com/fastlane/fastlane/issues/488) for more details.
 
Also keep in mind that, for all fastlane actions, almost all of their options can be specified as arguments, environment variables, or in a separate configuration file. Don't be afraid to look at the source code and use GitHub's "search in repository" feature. Here's a direct link to all the actions we're using: 
 
* [Gym](https://github.com/fastlane/fastlane/tree/master/gym) ([options](https://github.com/fastlane/fastlane/blob/master/gym/lib/gym/options.rb)) 
* [Scan](https://github.com/fastlane/fastlane/tree/master/scan) ([options](https://github.com/fastlane/fastlane/blob/master/scan/lib/scan/options.rb)) 
* [Match](https://github.com/fastlane/fastlane/tree/master/match) ([options](https://github.com/fastlane/fastlane/blob/master/match/lib/match/options.rb)) 
* [Resign](https://github.com/fastlane/fastlane/blob/master/sigh/lib/sigh/resign.rb) ([options](https://github.com/fastlane/fastlane/blob/master/sigh/lib/sigh/resign.rb#L67)) 
* [Act](https://github.com/richardszalay/fastlane-plugin-act) (this one's mine) 
* [Hockey](https://github.com/fastlane/fastlane/blob/master/fastlane/lib/fastlane/actions/hockey.rb) ([options](https://github.com/fastlane/fastlane/blob/master/fastlane/lib/fastlane/actions/hockey.rb#L81)) 
* [Deliver](https://github.com/fastlane/fastlane/tree/master/deliver) ([options](https://github.com/fastlane/fastlane/blob/master/deliver/lib/deliver/options.rb)) 
 
## Implementation

### Build 
 
To build, we use Gym, which will build an xcarchive and export it as an IPA. The IPA should be unsigned, since we won't know which signing identity to use until deployment, and it also avoid having to access the signing identity repository during this phase. 
 
> **Gotcha #5:** Gym doesn't officially support a way of creating an unsigned IPA, and doesn't make it easy to do due to the way it's designed. Gym tests both builds and exports. If it's signing identity option is applied as a blank string, the build will succeed but the export will attempt to sign it with a blank signing identity, which falls. There's also no way to just build and not export. As a workaround, we use the xcargs option to supply additional command arguments to xcodebuild but not xcrun. See [fastlane/#5617](https://github.com/fastlane/fastlane/issues/5617) 
 
> **Gotcha #6:** Xcode 7 introduced a new API for exporting archives. The new system is triggered by using the `-exportArchive` parameter with a manifest specified by `-exportOptionsPlist `, and supports newer iOS features such as app thinning and bitcode. However, it doesn't support generating unsigned IPAs based on my research. Still, we'll look at how we can take advantage of those features in a future post. 
 
```ruby
desc "Build an unsigned IPA artifact" 
lane :build do 
  gym( 
    output_directory: "./fastlane/build", 
    scheme: "MyApp", 
    configuration: "Release", 
    use_legacy_build_api: true, 
    xcargs: "CODE_SIGNING_REQUIRED=NO CODE_SIGN_IDENTITY=''" 
  ) 
end
```

### Test 
 
To test we use Scan, selecting the simulator to run then in and the scheme to use when building the tests. 
 
> **Gotcha #7:** Scan always outputs test results using the selected format as a file extension, in our case report.junit. Some CI servers might but pick this up, so we'll rename the file to .xml. 
 
```ruby
desc "Run unit tests"
lane :test do
  begin
    scan(
      scheme: "MyApp",
      device: "iPhone 5 (9.1)",
      output_types: "junit"
    )
  ensure
    #rename it so CI can find it
    sh "mv test_output/report.junit test_output/report.xml"
  end
end
```

### Putting it all together

Here's a gist of the fully assembled Fastfile:

{% gist b0546b4ec49ae2be118a21fb883e32f7 .bashrc %}

{% gist b0546b4ec49ae2be118a21fb883e32f7 build.sh %}

{% gist b0546b4ec49ae2be118a21fb883e32f7 build-gemfile.rb %}

{% gist b0546b4ec49ae2be118a21fb883e32f7 build-fastfile.rb %}

#### Build artifacts 
 
We now have two sets of artifacts that can be captured: 
 
**fastlane/test_output/report.xml** - unit test results for your CI server. 
 
**fastlane/build/DeploymentPipeline.ipa** and 
**fastlane/build/DeploymentPipeline.app.dSYM.zip** - application archive and symbols archive. Well need to keep both for our deployment phase: the IPA contains our application and the dSYM.zip is useful for getting meaningful crash reports in services like HockeyApp, TestFlight, and Sentry. How you capture these artifacts will depend on your CI server, but most should support a glob-style pattern. 

In [Part 4]({% post_url 2016-08-14-ios-deploy-pipeline-4-deploy-phase %}) we'll setup our deployment fastfile in the configuration repository.