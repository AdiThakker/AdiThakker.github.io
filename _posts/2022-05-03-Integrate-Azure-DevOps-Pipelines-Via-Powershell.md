---
layout:     post
title:      Integrate Azure DevOps Pipelines via Powershell
date:       2022-05-03
summary:    This post explores how to automate Azure DevOps pipelines via its Powershell module
categories: Azure DevOps, Powershell
---

If you read the [last]({{site.url}}/Integrate-Azure-DevOps-Pipelines-Via-Code) post, we saw how you can integrate Azure DevOps via its [.NET Client Libraries](https://docs.microsoft.com/en-us/azure/devops/integrate/concepts/dotnet-client-libraries?view=azure-devops) to automate task import (from Task Groups) into our CI/CD pipelines.

You must have noticed, that I mentioned another alternative [VSTeam Powershell](https://github.com/MethodsAndPractices/vsteam) as well, in this post we will see how to automate atleast one of the task import automation via it.

If you read through the VSTeam's link, the first thing we have to do is download that module. So lets get started...

When i first started powershell, i updated it to the latest version and installed the module as shown below:

![image]({{site.url}}/images/devops-ps-1.png)

![image]({{site.url}}/images/devops-ps-2.png)

After that, I validated my connection to Azure DevOps via [PAT]() as shonw below: 

![image]({{site.url}}/images/devops-ps-3.png)

Now, I was ready to script this automation, which I have shown below with comments to explain main steps:

***NOTE:  i heavily leveraged [source](https://github.com/MethodsAndPractices/vsteam/tree/trunk/Source/Public) to find the cmdlets and experimented to get the below script working:*** 

~~~powershell

# Get the CI Build Definition
$buildDefinition = Get-VSTeamBuildDefinition -Id 4

# Get the Build Task Group
$buildTask = Get-VSTeamTaskGroup -Name "Update Tags"

# Import Build Task into the Build Definition and update it
$steps = $buildDefinition.process.phases[0].steps

# Add new Task
$newTasks = @{}
$iteration = 1
foreach($step in $steps)
{
	$newTasks.Add($step.displayName, $step)
}
$newTasks.Add($buildTask.name, $buildTask)

# update Definitions Tasks
$buildDefinition.process.phases[0].steps = $newTasks

# Update Definition
$buildJson = $buildDefinition | ConvertTo-Json -Depth 100
Update-VSTeamBuildDefinition -ProjectName AzureFunctionDeployment -Id 4 -BuildDefinition $buildJson

~~~

So there you see folks... depending on your preference, you can leverage this option as well. For my use case we ended up leveraging the .NET Client libraries via that option

***NOTE: I'll leave the updating of CD pipeline as an  exercise for you to explore 😉 and keep this post short.***


