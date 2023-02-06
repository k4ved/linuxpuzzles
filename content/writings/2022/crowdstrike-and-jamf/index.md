---
author: Andy James
date: '2022-03-28'
summary: |
  Crowdstrike and Jamf are a puzzle to work with together. It can be easy, but getting to a working solution will take time.
  This blog is geared towards fixing that.
tags:
  - crowdstrike
  - jamf
  - security
title: Crowdstrike and Jamf Pro, Together at Last
---

As an IT company, securing our users devices was a must to do. When we did our audit on all the platforms we wanted to use we landed on Crowdstrike Falcon. All signed up, access granted, off to install on all our systems using Jamf Pro.

Thats where the fun began. At the time we started with Crowdstrike they did not have any real documentation for using Jamf Pro, and their support is very clear that they do not support Jamf Pro. They did provide a configuration file, but it was complicated to install, required you to re-sign it, and then didn’t work. A lot of banging my head against my desk to figure out whats going on with their recommended policy.

Jamf’s support also doesn’t support crowdstrike in return, but was helpful to provide some steps that helped get closer to an install, but they also didn’t work well and depended on the end-users clicking the Crowdstrike Falcon pop-up to get going. This isn’t ideal as not all users are in-front of their computer when pushed, if they did not click in time, the install would fail.

With all that said, Crowdstrike has now put together some documentation that, with a caveat that I’ll talk about at the end, mostly solves the problem. Here are the instructions you need.

## How to install Crowdstrike with Jamf Pro

### Create a new configuration policy for all devices

Edit the sections as follows:

**Privacy Preferences Policy Control**

```
App Access
Identifier: com.crowdstrike.falcon.Agent
Identifier Type: Bundle ID
Code Requirement: identifier "com.crowdstrike.falcon.Agent" and anchor apple generic and certificate 1[field.1.2.840.113635.100.6.2.6] /* exists / and certificate leaf[field.1.2.840.113635.100.6.1.13] / exists */ and certificate leaf[subject.OU] = "X9E956P446"
App or Service: SystemPolicyAllFiles -> Allow
```
```
App Access
Identifier: com.crowdstrike.falcon.App
Identifier Type: Bundle ID
Code Requirement: identifier "com.crowdstrike.falcon.Agent" and anchor apple generic and certificate 1[field.1.2.840.113635.100.6.2.6] /* exists / and certificate leaf[field.1.2.840.113635.100.6.1.13] / exists */ and certificate leaf[subject.OU] = "X9E956P446"
App or Service: SystemPolicyAllFiles -> Allow
```
**Application & Custom Settings**

In the upload section, let’s setup plist. This will save you from running a script to add the license later

*Preference Domain:* com.crowdstrike.falcon

Property List:
```
<dict>
  <key>ccid</key>
    <string>YOUR CCID</string>
</dict>
```
**System Extensions**
```
Allow users to approve system extensions: Checked
Display Name: com.crowdstrike.falcon.Agent
System Extension Type: Allowed System Extensions
Team Identifier: X9E956P446
Allowed System Extensions: com.crowdstrike.falcon.Agent
```
**Content Filter**
```
Filter Name: Falcon
Identifier: com.crowdstrike.falcon.App
Organization: CrowdStrike, Inc.
Filter Order: Inspector
Socket Filter:
Socket Filter Bundle Identifier: com.crowdstrike.falcon.Agent
Socket Filter Designated Requirement: identifier "com.crowdstrike.falcon.Agent" and anchor apple generic and certificate 1[field.1.2.840.113635.100.6.2.6] and certificate leaf[field.1.2.840.113635.100.6.1.13] and certificate leaf[subject.OU] = "X9E956P446"
```
### Create a new configuration policy for all Mac OS X Catalina

**Approved Kernel Extensions**
```
Allow users to approve kernel extensions: Checked
Display Name: Crowdstrike, Inc
TeamID: X9E956P446
```
### Scoping
When you scope these two policies, all computers in your organization should get the all computers policy. Using a smart group, you should make a group to narrow down to just Catalina and scope that group to the Catalina policy also. Both policies should be installed on Catalina, but only one on every other OS. If you’re running older than Catalina, you should upgrade them first.

### Caveat while building the install policy
Installing is easy, upload the image, create a policy to install that application, and done. No bash scripts for licenses required, installs like every other application, with one exception.

The Falcon installer doesn’t handle being installed “again”. IE, if you push out new policies or changes to your installer policy, it will simply show all your devices failing. When you dig into the error on Google you will find lots of results, but no real answers. Heres the short version of whats really happening;

When the installer exits out, it doesn’t send an exit code 0. This is because the installer technically did fail, even though it simply just exited because Falcon is already installed. Jamf can’t tell what the installer should have done, so it just sees the exit code is not successful, and therefore the install is a failure.

My recommended approach is to change your policy to install on computers where Falcon is not already installed using smart groups. This makes sure that it’s the first time install. Falcon takes care of updating itself, so no future monitoring in Jamf is required. The smart group to create is as follows:

Create a new smart group, name it what you want, and use the following criteria
```
Application Title is not Falcon.app
```
