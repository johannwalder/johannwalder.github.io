---
layout: post
title:  "NuGet package fails to install"
date:   2017-10-07
categories: 
tags: configuration dotnet
---

From time to time when you have solved an issue you might think afterwards how a problem was so easy to fix but "difficult" to spot at first.
Well, this is one of those odd cases.

I only needed to add one NuGet package to a couple of projects and at the beginning everything seemed to be going forward without any issues - until I hit one project 
where the same NuGet package couldn't be installed.

<!--more-->
  
Instead of a successful installation I was greeted with the following error message:

```
An error occurred while applying transformation to 'App.config' in project No element in the source document matches '/configuration/*[1]'
```

### Background

As you probably have already guessed from the error message above the NuGet package performs some changes to the application's configuration file
during its installation.

### The odd spot

But when checking the configuration file everything seemed to be correct.

To further narrow down the problem I have created a new project and tried to only deploy this particular NuGet package.
But surprisingly the installation succeeded there.

So, back to the project with the issue:
As the Visual Studio solution consisted of multiple projects (a second one like the one with the issue here) I have performed
the same steps for the second project too. And the package installed there successfully as well.

Now I was wondering what else could be different and compared those two projects.

### Solution

After checking those projects, I have noticed that both are using a linked configuration file from a common project to share same configuration file
across multiple similar projects.

This shouldn't have been any problem as the other project worked fine. 

But comparing the folders revealed the issue. 

When the project with the issue has been changed to use the linked configuration file the project changes have been committed - almost correctly -
but an important piece has been forgotten - the removal of the project's own app.config file from the file system.

And this particular app.config file had the following content:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration />
```

However, I couldn't see this file content from within Visual Studio as it was no longer part of the project itself. 
When I looked at the app.config file within Visual Studio I saw the linked app.config file from the common project and its content was correctly formatted.

Back then the app.config file has been excluded from the project itself but the file hasn't been removed from the repository and this caused the issue described here.

The moment I deleted the app.config file from the file system under the particular project the package could be installed successfully.


