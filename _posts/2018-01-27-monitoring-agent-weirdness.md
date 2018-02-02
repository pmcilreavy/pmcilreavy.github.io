---
layout: post
title: Microsoft Monitoring Agent Assembly Weirdness
excerpt: "Calling GetReferencedAssemblies() on an Assembly returns different results if Microsoft Monitoring Agent is installed."
date: 2018-01-27
categories: [c#, dotnet, operations manager]
comments: true
share: true
firehose: true
---

# The Problem
Whilst working on a C# WebApi project I ran into a strange assembly issue. At startup the application used reflection to enumerate all of the assemblies already loaded into the current AppDomain. Then for each of those assemblies it got the referenced assemblies and loaded them into the AppDomain if they weren't already. The pseudo code for that is shown below.

```cs
var loadedAssemblies = AppDomain.CurrentDomain.GetAssemblies();

foreach (var assembly in loadedAssemblies)
{
    var referencedAssemblies = assembly.GetReferencedAssemblies();

    foreach (var assemblyName in referencedAssemblies)
    {
        if (!IsAlreadyLoaded(assemblyName))
        {
            // if this assembly cannot be resolved an error will be thrown
            Assembly.Load(assemblyName); 
        }
    }
}
```

This process is recursively repeated until all assemblies are loaded, logging any failures on the way. One of the benefits this gives is to identify any deployment issues early (i.e. missing dlls).

After having been tested and deployed through various environments the problem arose when deployed to PROD where this crashed and burned with an error about a missing dll called *Microsoft.WindowsAzure.StorageClient*. This was particularly puzzling as the code did not use this dll and the solution was hosted on-premise, not in Azure.

# The Investigation
The dlls deployed to each environment were identical and were promoted and controlled via *Octopus Deploy* so I was relatively confident that it was something specific to the PROD environment itself that was causing the problem.

In order to try and establish where this transitive reference to *Microsoft.WindowsAzure.StorageClient* was coming from, I wrote a stand alone single page asp.net forms app which replicated the recursive assembly loading logic but also wrote out the results in a hierachical html table structure. This web page gave me the following output in *production*:

### *Production Output*
<img src="..\..\img\prod.PNG" alt="Production" title="Production">

This allowed me to see, if we follow the green arrows from left to right, that *System.ServiceModel.Web* has a reference to *System.Web.Extensions* which has a reference to *System.Data.Services.Client* which has (you still with me) a reference to *Microsoft.EnterpriseManagement.OperationsManager.Apm.DataCollecting.Producers.Azure.1.6* which has a reference to the offending *Microsoft.WindowsAzure.StorageClient* dll which was failing to load.

### *Test Output*
<img src="..\..\img\test.PNG" alt="Test" title="Test">

On *test* though, as you can see from the diagram above, this is not the case. The trail of breadcrumbs ends at *System.Data.Services.Client* and there are no subsequent references to *Microsoft.EnterpriseManagement.OperationsManager.Apm.DataCollecting.Producers.Azure.1.6* and so in TEST we didn't have a problem because the code didn't attempt to load the *Microsoft.WindowsAzure.StorageClient* dll.

I tried to find any differences between these servers and *OperationsManager* in the dll name was the initial clue. Sure enough I could see that the prod server had *Microsoft Monitoring Agent* installed which is part of Operations Manager.

I also ported my diagnostic app to a console application and did not observe the same problem, so it seems to be confined to IIS hosted code.

I'm speculating then that the mere act of installing *Microsoft Monitoring Agent* is somehow altering *System.Data.Services.Client* such that it has this additional reference to *Microsoft.EnterpriseManagement.OperationsManager.Apm.DataCollecting.Producers.Azure.1.6* that it would not otherwise have. How this only effects IIS code is still a bit of a mystery.

# The Solution
In the end I changed the assembly loading code to explicitly ignore the operations manager dlls and the problem went away. I still find it very puzzling that installing *Microsoft Monitoring Agent* would cause `GetReferencedAssemblies()` to return different result. I would be interested to hear any alternative theories.
