---
layout: post
title:  "Using Application Insights for centralized logging"
date:   2017-10-08
categories: 
tags: azure logging applicationinsights
---

{: .post-image}
![application insights search example]({{ site.contenturl }}20171008-application-insights-search-example.png)

We have been using Azure for a while now and for one of our projects we have been using log4net as our logging framework.
But as the product expands across multiple Azure resources (including different Azure app services) I wanted to find
a way to centralize log entries.

<!--more-->

There are many different options available and one very convinient for an Azure environment is to use [Application Insights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-overview){:target="_blank"}.

[Application Insights API](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-api-custom-events-metrics){:target="_blank"} can be used to send custom events and metrics from all kind of devices, 
however, in our case log4net has already been used for many parts of the application. 

Fortuntly, there is an [Application Insights Log4Net Appender](https://www.nuget.org/packages/Microsoft.ApplicationInsights.Log4NetAppender){:target="_blank"} as a NuGet package
that allowed us to keep log4net logging in place by just adding a new appender to our log4net configuration which sends existing log messages to Application Insights.

There are very good blog posts describing how to set up Application Insights with log4net and therefore I won't go into the details here but as a summary:
- install Log4Net NuGet package
- add "log4net.Config.XmlConfigurator.Configure();" to the code
- install Microsoft.ApplicationInsights.Log4NetAppender NuGet package
- set InstrumentationKey
- configure log4net appender for Application Insights

One easy mechanism I want to show you is how it's possible to send every components messages to the same Application Insights instance but still be able to recognize from which component
the log entry has been created.
Basically, first we need to use the same InstrumentationKey for all componentns and second we just needed to use the log4net pattern layout.

When Application Insights is set up it will create the following pattern layout:

```xml
<layout type="log4net.Layout.PatternLayout">
    <conversionPattern value="%message%newline" />
</layout>
```

And we can just modify the pattern a little bit and add an identifier for our component:
```xml
<layout type="log4net.Layout.PatternLayout">
    <conversionPattern value="COMPONENT1:%message%newline" />
</layout>
```

There are other ways too but this was the easiest one to get started with.

After modifying the pattern we can see something like the image below within Application Insights blade in Azure portal:

![sample log message]({{ site.contenturl }}20171008-sample-log-message.png)

### Making the "InstrumentationKey" easily configurable

Application Insights uses the "InstrumentationKey" within the ApplicationInsights.config file to send messages to the correct Application Insights instance.
But this isn't optimal when using different environments (i.e. dev, test, production) and all environments should use their own Application Insights instance.

Therefore we have chosen to set the InstrumentationKey from the appSettings as shown below:

```csharp
public void Configuration(IAppBuilder app)
{
    // changed the code for simplicity by excluding check if the appSettings key exists
    TelemetryConfiguration.Active.InstrumentationKey = ConfigurationManager.AppSettings["ApplicationInsightsInstrumentationKey"];
    log4net.Config.XmlConfigurator.Configure();
    
    // ...
}
```

### [Issue: Azure Storage stopped working after enabling Application Insights](#issue-azure-store-stopped-working-after-enabling-application-insights)

The minute, the Application Insights NuGet packages have been installed and configured I noticed that Azure storage code we used to upload files wasn't any longer working.

Looking at the logs I noticed the following error message:

```
"The remote server returned an error: 403 Forbidden"
```

First I was thinking an access key has changed and I also looked at other Azure Storage related settings as at that time I didn't make the connection to Application Insights yet.
But not long after searching for similar issues I came across the following issue where the same problem was explained:
[403 Forbidden when connecting to Azure Storage with Application Insights Web Tracking HTTP Module](https://github.com/Azure/azure-sdk-for-net/issues/3460){:target="_blank"}

As described in the comments the issue has been solved in version 2.4.1. But at the time of writing this post here when using the option "Add Application Insights to Project" in Visual Studio it will still install version 2.4.0.

The solution is just to update the "Microsoft.ApplicationInsights.Web" NuGet package to at least 2.4.1 after the Application Insights installation has finished.

### [Issue: ApplicationInsights.config being ignored](#issue-applicationinsights-config-being-ignored)

Another issue occurred with ApplicationInsights.config file which was ignored when using Git as it has been added to .gitignore in a pull request quite some time ago:

[Ignore ApplicationInsights.config for Microsoft Azure](https://github.com/github/gitignore/pull/1815){:target="_blank"} 

Luckily, it has already been fixed but our Visual Studio .gitignore version hasn't been updated to include the fix for this PR above. 
After reverting the changes of this particular PR the ApplicationInsights.config file has been added to our Git repository successfully as well.

This post is not even touching the tip of the iceberg of Application Insights as its possibilities are huge, so the journey will continue... 

