---
layout: post
title: Microsoft Monitoring Agent Assembly Weirdness
excerpt: "GetReferencedAssemblies() returning different results if Microsoft Monitoring Agent is installed."
date: 2018-01-27
categories: [c#, dotnet, operations manager]
comments: true
share: true
---

# The Problem
I was working on a C# WebApi project which at startup used reflection to get all the assemblies in the current AppDomain. Then for each of those assemblies; get the referenced assemblies and load them into the AppDomain if they weren't already.

```cs
...

var referencedAssemblies = assembly.GetReferencedAssemblies();

...

foreach (var assemblyName in referencedAssemblies)
{
    Assembly.Load(assemblyName);
}

...
```

We recursively repeat this process until all assemblies are loaded, logging any failures on the way. One of the benefits this gives is to identify any deployment issues early (i.e. missing dlls).

This worked fine on dev machines and on TEST and UAT servers however on PROD this crashed and burned with an error about a missing dll called *Microsoft.WindowsAzure.StorageClient*. This was particularly puzzling as the code did not use this dll and the solution was hosted on-premise, not in Azure.

# The Investigation
The dlls deployed to each environment are identical and are promoted and controlled via *Octopus Deploy* so I was relatively confident that it must be something specific to the PROD environment itself that was causing the problem.

First thing was to try and establish where this transitive reference to *Microsoft.WindowsAzure.StorageClient* was coming from. I wrote a single page asp.net forms app which replicated the recursive assembly loading logic but also wrote out the results in a hierachical table structure. This web page gave me the following output in *production*:

### *Production Output*
<img src="..\..\img\prod.PNG" alt="Production" title="Production">

This allowed me to see, if we follow the green arrows from left to right, that *System.ServiceModel.Web* has a reference to *System.Web.Extensions* which has a reference to *System.Data.Services.Client* which has (you still with me) a reference to *Microsoft.EnterpriseManagement.OperationsManager.Apm.DataCollecting.Producers.Azure.1.6* which has a reference to the offending *Microsoft.WindowsAzure.StorageClient* dll which fails to load.

### *Test Output*
<img src="..\..\img\test.PNG" alt="Test" title="Test">

On *test* though, as you can see from the diagram above, this is not the case. The trail of breadcrumbs ends at *System.Data.Services.Client* and there are no subsequent references to *Microsoft.EnterpriseManagement.OperationsManager.Apm.DataCollecting.Producers.Azure.1.6*.

I looked on the prod server and could see *Microsoft Monitoring Agent* was installed which I'm assuming is part of Operations Manager.

I then ported my diagnostic app to a console application and did not get the same problem, so it seems to be confined to IIS hosted code.

I'm speculating then that the mere act of installing *Microsoft Monitoring Agent* is somehow altering *System.Data.Services.Client* such that it has this additional reference to *Microsoft.EnterpriseManagement.OperationsManager.Apm.DataCollecting.Producers.Azure.1.6* that it would not otherwise have.

# The Solution
In the end we changed our assembly loading code to explicitly ignore the operations manager dlls and the problem went away. I still find it very puzzling that installing *Microsoft Monitoring Agent* would cause this dll to be altered in this way. I would be interested to hear any alternative theories.
