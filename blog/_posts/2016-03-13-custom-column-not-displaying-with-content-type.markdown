---
layout: post
title:  "Custom column not displaying with content type"
date:   2016-03-13 12:00:00
categories: 
---
{: .post-image}
![invalid field name]({{ site.contenturl }}20160313-invalid-field-name.png)

A while back a SharePoint user has reported that a custom site column isn't visible when adding documents to a library. The document library uses a custom content type and therefore before starting to look into this issue I was thinking the column would just need to be associated to the content type as it has been the root cause in a few previous cases.

<!--more-->

But when further looking at the document library settings everything seemed to be set up correctly as the site column "Test column" was connected with the content type "Test Content Type" as shown in the image below:

![content type used columns]({{ site.contenturl }}20160313-content-type-used-columns.png)

Then I was considering if the site column itself would have just been hidden from the form fields but this wasn't true either. Furthermore, I have also checked the GUID of the field and it matched the field definition from the Elements.xml file.

So, troubleshooting continued and I checked the content type itself:

![content type columns]({{ site.contenturl }}20160313-content-type-columns.png)

### [](#header-3)Issue

At first everything appeared to be accurate but when I clicked on the "Test column" link I was greeted with the **"Invalid field name. {15db2ed4-0b62-40af-aabd-c99ea363a782}"** error and when I also clicked on "Add from existing site or list columns" I realized that the "Test column" hasn't been correctly connected to the content type as it was still available from the list to choose from:

![list content type add columns]({{ site.contenturl }}20160313-list-content-type-add-columns.png)

As a result, I have just added it and the column appeared correctly when adding or editing document properties. It hasn't either created a duplicate column but as it seems it linked the column with the content type correctly.

However, as this was a temporary fix for the current document library I needed to look deeper into this issue to find a permanent solution.

There are two SharePoint projects involved with the creation of the document library and adding the custom column: the core project where the library is created during site provisioning and the custom project where customer specific columns will be added.

Consequently, my first guess was to check the custom project for any hints - after all the issue concerns the custom column. 

First I checked how the column is added to the custom library during site provisioning. It is added by a feature receiver that runs when a new library has been added to the site. The custom column is obtained from the site columns and then the
[SPFieldCollection.AddFieldAsXml](https://msdn.microsoft.com/en-us/library/office/ms472543.aspx){:target="_blank"} method is used to add it to the library itself. 
This method has a parameter [SPAddFieldOptions](https://msdn.microsoft.com/en-us/library/office/microsoft.sharepoint.spaddfieldoptions.aspx){:target="_blank"} which can be used to set different options, like adding the column to all list content types with "AddToAllContentTypes" which has been used in this particular code as well.
Further analyzing and testing this particular code part hasn't revealed any issue and therefore it was time to go back to the "drawing board".

### [](#header-4)Hint:

But before doing so I would like to put our attention to one particular issue with the SPAddFieldOptions that has been causing issues before as well. The problem back then was that the display name and internal name of the site columns has been set to different values, like:

```xml
<Field ID="15db2ed4-0b62-40af-aabd-c99ea363a782"
Name="TestColumnInternalName"
StaticName="TestColumnInternalName"
DisplayName="TestColumnDisplayName"> <!-- for simplicity of this blog post but usually set via resource file -->
...
</Field>
```

When using the SPAddFieldOptions with "AddToAllContentTypes" option only the "Name" value is ignored and in the list both values are set to "TestColumnDisplayName". This issue is very easy to prevent by adding the option "AddFieldInternalNameHint".


Now back to the original problem with the "Invalid field name" and because analyzing the custom project code hasn't revealed any issue I started to have a closer look at the core project.

At this point, I checked how the custom library is provisioned and ended up looking at the definition of the custom content type. And initially it hasn't looked any different to any of the other content types and at the beginning I couldn't spot the issue. But after looking closer at the definition it got obvious that the OOTB columns have been added twice: once as a field reference and second as a field itself:

```xml
<contenttype id="0x0101002460FA4C5D5B4330A7A2BB2AD04AC561" name="$Resources:ResourceFile,contenttype_testcontenttype_name;" group="$Resources:ResourceFile,contenttype_group_name;" description="" inherits="TRUE" version="0">
    <fields>
        <field id="{8553196d-ec8d-4564-9861-3dbe931050c8}" name="FileLeafRef" sourceid="http://schemas.microsoft.com/sharepoint/v3" staticname="FileLeafRef" group="_Hidden" showinfiledlg="FALSE" showinversionhistory="FALSE" type="File" displayname="$Resources:core,Name;" authoringinfo="$Resources:core,for_use_in_forms;" list="Docs" fieldref="ID" showfield="LeafName" joincolname="DoclibRowId" joinrowordinal="0" jointype="INNER" required="TRUE"></field>
        <field id="{fa564e0f-0c70-4ab9-b863-0177e6ddd247}" name="Title" sourceid="http://schemas.microsoft.com/sharepoint/v3" staticname="Title" group="_Hidden" type="Text" displayname="$Resources:core,Title;" required="TRUE" frombasetype="TRUE"></field>
    </fields>
    <fieldrefs>
        <!-- FileLeafRef -->
        <fieldref id="{8553196d-ec8d-4564-9861-3dbe931050c8}" name="FileLeafRef" required="TRUE"></fieldref>
        <!-- Title -->
        <fieldref id="{fa564e0f-0c70-4ab9-b863-0177e6ddd247}" name="Title" required="FALSE" showinnewform="FALSE" showineditform="TRUE"></fieldref>
    </fieldrefs>
</contenttype>
```

### [](#header-3)Solution

After removing the duplicated fields from the custom content type and deploying the updated solution package everything started to work again for newly created document libraries:

```xml
<contenttype id="0x0101002460FA4C5D5B4330A7A2BB2AD04AC561" name="$Resources:ResourceFile,contenttype_testcontenttype_name;" group="$Resources:ResourceFile,contenttype_group_name;" description="" inherits="TRUE" version="0">
    <fieldrefs>
        <!-- FileLeafRef -->
        <fieldref id="{8553196d-ec8d-4564-9861-3dbe931050c8}" name="FileLeafRef" required="TRUE"></fieldref>
        <!-- Title -->
        <fieldref id="{fa564e0f-0c70-4ab9-b863-0177e6ddd247}" name="Title" required="FALSE" showinnewform="FALSE" showineditform="TRUE"></fieldref>
    </fieldrefs>
</contenttype>
```

For all existing libraries I have wrote a small PowerShell script to identify those libraries where this issue existed and luckily it hasn't been spread to too many libraries which could be fixed easily after we got aware of the root cause.

Another SharePoint task and again a bit wiser.

**Disclaimer**: In this post I have modified everything from field GUID, over field and list names to design of the site to protect the customer's identity.