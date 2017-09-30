---
layout: post
title:  "Activating SharePoint Search Topology fails"
date:   2015-07-10
categories: 
tags: sharepoint
---
{: .post-image}
![search service application]({{ site.contenturl }}20151007-search-service-application.png)

This was another of those kind of days when everything has been done as before but all repeated steps this time - no matter what has been tried - just didn't work.

After a SharePoint 2013 Search Service application got corrupted we first tried to fix it but at the end we needed to re-deploy a new SharePoint 2013 Search Service application.

Therefore we have removed the existing Search Service and run a PowerShell script to provision the service again. But even though the script has always run successfully before it just didn't want to succeed this time.

<!--more-->

There were several errors reported to the event logs and the script failed when executing the following line:

```powershell
$searchTopology.Activate()
```

The command above resulted in the following error:

{% highlight powershell-error %}
Exception calling "Activate" with "0" argument(s): "Topology does not contain any components of type Microsoft.Office.Server.Search.Administration.Topology.AdminComponent"
At line:1 char:1
+ $SearchTopology.Activate()
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [], MethodInvocationException
    + FullyQualifiedErrorId : InvalidTopologyException
{% endhighlight %}

We have followed many different troubleshooting steps including but not limited to checking permissions on all related Search Service databases and folders.

But after some further analysis the script contained the following line:

```powershell
$err = null
```

which prevented some errors to be displayed.

Removing this line and running the script again it stopped this time at the following line which adds a new index component:

```powershell
New-SPEnterpriseSearchIndexComponent -SearchServiceInstance $ssiAppSvr -SearchTopology $searchTopology -RootDirectory $indexLocation -IndexPartition 0
```

And now it presented us with the following error:

```powershell
New-SPEnterpriseSearchIndexComponent : Cannot bind parameter 'RootDirectory' to the target. Exception setting "RootDirectory": "New index location must be empty"
```

Now actually the Index location should have been setup with a custom location and after noticing this error checking the current index location with the following command:

```powershell
$ssa = Get-SPEnterpriseSearchServiceApplication 
$ssa.AdminComponent.IndexLocation
```

revealed that the location hasn't been set to the custom location but at the time when the activation of the search topology occurred it was set to the default SharePoint location.

Now based on the error before checking the custom index location showed that it included some subfolders from the previous installation.

After removing these subfolders and running the script again the index component could be successfully created using the custom location and when the search topology activation occurred it could use the custom index location and the Search Service application provisioned successfully.