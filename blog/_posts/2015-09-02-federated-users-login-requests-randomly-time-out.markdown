---
layout: post
title:  "Federated users login requests randomly time out"
date:   2015-09-02 12:00:00
categories: 
---
{: .post-image}
![win proxy setttings]({{ site.contenturl }}20150902-win-proxy-settings.png)

When you have been working with SharePoint you might have came across the need to allow CRL (certificate revocation list) checks from SharePoint servers. It's nothing new and also exists with standard ASP.NET applications but with SharePoint it's not always as obvious as you will find out shortly.

<!--more-->

There have already been written many great articles about some different options for [CRL checking with SharePoint](https://blogs.msdn.com/b/chaun/archive/2014/05/01/best-practices-for-crl-checking-on-sharepoint-servers.aspx){:target="_blank"} before and my preferred method is always to somehow allow the servers to get the certificate revocation list live from the real address.

### [](#header-3)Issue

In this particular case, federated users randomly faced issues during the login. And the issue couldn't either be isolated to any specific user group.
During the login process the user needed to wait longer than usual until at the end the user was presented with a timeout error page. After a second request the user could access the same site again and because of this random behavior and the success on second try we didn't have a closer look at it first.

### [](#header-3)Solution

The SharePoint servers couldn't access the internet directly as the network has been configured to force the usage of a proxy. And to allow CRL checks for SharePoint web applications we have added the following lines to the corresponding web.config files. The following entry has been added:

```xml
<defaultProxy useDefaultCredentials="true">
    <proxy usesystemdefault="false" proxyaddress="proxyserver" bypassonlocal="true" /> 
</defaultProxy>
```

But this change above has already been applied before and therefore not the cause of the problem reported.

Finally, when further analyzing this issue we noticed that the Event Viewer reported errors similar to:

```xml
A certificate validation operation took 17328.8231 milliseconds and has exceeded the execution time threshold.  If this continues to occur, it may represent a configuration issue.  Please see [https://go.microsoft.com/fwlink/?LinkId=246987](https://go.microsoft.com/fwlink/?LinkId=246987){:target="_blank"} for more details.
```

And the event log has also revealed the second part which was pointing to the user account which triggered the event entry and it was the user assigned to the STS web application "SecurityTokenServiceApplication":

![sts iis site]({{ site.contenturl }}20150902-sts-iis-site.png)

This SharePoint web application also has its own web.config file and in a default installation it is located in the following folder:

```plaintext
C:\Program Files\Common Files\Microsoft Shared\Web Server Extensions\15\WebServices\SecurityToken\
```

Now remembering the proxy settings configured for the custom SharePoint web applications before an easy solution would have just been to perform the same settings to this particular web.config file as well. But with best practices in mind we don't change any SharePoint out-of-the-box files which might get overridden by any future SharePoint update.

Therefore we were thinking if it's possible to set the proxy settings across Windows which landed us to the following command:

```shell_session
netsh winhttp set proxy "<proxy-address>" "<addresses-excluded-from-proxy>"
```

If you want to know that any proxy has been configured you can run the following command:

```shell_session
netsh winhttp show proxy
```

### [](#header-3)Good practice

You might further want to restrict access to only specific CRL addresses (i.e. [https://crl.microsoft.com](https://crl.microsoft.com){:target="_blank"}) and not allow any other internet traffic from your SharePoint servers.

And to be able to analyze other issues with CRLs you can also [enable CAPI2 logging](https://blogs.msdn.microsoft.com/benjaminperkins/2013/09/30/enable-capi2-event-logging-to-troubleshoot-pki-and-ssl-certificate-issues/){:target="_blank"} temporary which will also give more details about failed CRL requests.

As always the steps described here have fixed our particular case but it should always be validated for correctness in a different environment.