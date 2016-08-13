## Designing the Deployment Pipeline

In part 1, we looked at the tools and processes for building and distributing iOS apps. Let's see what that looks like as a deployment pipeline that needs to be tested in HockeyApp before being released via the App Store.

The app we'll base the rest of this series on is a basic single screen app that displays a value from the infoDictionary (Info.plist). We'll be using that as  "environment specific configuration", but it could just as easily be an API base URL, or even a SQLite database used to seed data.

### Guiding tenants

#### Security

Imagine that every developer is a freelancer, that your build agent is publically shared, and that your client's IT wants to review your security configuration.

**Principle of least privilege**. Avoid accessing signing identitues and Apple portals until the deployment phase. This allows the build/test phase to be run by any developer, regardless of access.

#### Portability

Imagine that your build server is going to be rebuilt from scratch everyday. Even if it's not, all servers will die eventually.

**Isolation**.  Assume the build agent is shared in a public arena, so don't install (or rely on) passwords/certificates on the build agent's default keychain. Doing so makes the build less portable and opens the door to other builds being able to read your secrets.

#### Maintainability

Imagine that all of your configuration and secrets (eg. ITunes password) change one a month. 

Separate deployment configuration from application code. This allows the deployment scripts and configuration to evolve independently from the application configuration code. With this design, a release can be redeployed when configuration changes, without having to wait for another application build.

**Don't Repeat Yourself**. Signing Identities and Provisioning Profiles are owned by the team, not the application, so store them separately to both application code and deployment configuration. If your pipeline signs with more than one team (eg. Company Team for dev/adhoc, Client Team for app store) then you'll want one repository for each team. Everything should only have one source of truth.

### The Deployment Pipeline

![]({{ site.baseurl }}/assets/8E332AF192B64AEEBA05EBDAF7192470.png)

The diagram above, you can see the build/test phase is very separate from the deploy phase. This means that:

* There's no access to the signing identity and configuration repositories in the build/test phase
* The code repository is not needed for the deploy phase
* There's no connection to HockeyApp / iTunes until the deploy phases

#### Repositories

**Application Code Repository** – Contains xcproj, application code, and sufficient configuration to run on a simulator or development device.

**Signing Identity Repository** – Maintained by fastlane's "match". Contains certificates, private keys, and provisioning profiles related to the team. Updated via manually executed fastlane "lanes" (tasks). We'll be setting up a task to keep this updated, but if you'd prefer to migrate your existing Signing Identities and Provisioning Profiles Michał Laskowski has a great guide over on Macoscope.

**Configuration Repository** – Contains deployment scripts (fastlane) and the configuration specific to each "environment" that the app runs in, such as QA/HockeyApp or the App Store.

#### Pipeline Phases

**Test Phase** - build and run unit tests in the simulator and outputs the results in junit format. 

**Build Phase** - build an IPA that is unsigned, since the signing identity will be chosen at deploy time. 

**QA Deploy Phase** - using the IPA artifact, change our plist value to "QA", sign it using an AdHoc identity+ profile, and deploy to HockeyApp for testing.

**App Store Deploy Phase** - using the IPA artifact, change our plist value to "AppStore", sign the app using the app store certificate. Deploy to HockeyApp (for crash analytics), and release a new build to iTunes.

#### Secrets

The deploy phase will need an Apple Developer account, to verify signing identities and deploy to iTunes. It will also need a hockeyapp api key, and access to the certificates repository.
