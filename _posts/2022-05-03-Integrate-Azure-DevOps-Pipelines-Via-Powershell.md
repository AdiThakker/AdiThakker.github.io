---
layout:     post
title:      Integrate Azure DevOps Pipelines via Powershell
date:       2022-05-03
summary:    This post explores how to automate Azure DevOps pipelines via its Powershell module
categories: Azure DevOps, Powershell
---

If you read the [last]({{site.url}}/Integrate-Azure-DevOps-Pipelines-Via-Code) post, we saw how you can integrate Azure DevOps via its [.NET Client Libraries](https://docs.microsoft.com/en-us/azure/devops/integrate/concepts/dotnet-client-libraries?view=azure-devops) to automate task import (from Task Groups) into our CI/CD pipelines.

You must have noticed, that I mentioned another alternative [VSTeam Powershell](https://github.com/MethodsAndPractices/vsteam) as well, so in this post we will see how to automate atleast one of the task import automation via it.

If you read through the VSTeam's link, the first thing we have to do is download that module. So lets get started...

Firing up pwershell, I first updated it to the latest version and then installed the module as shown below:

![image]({{site.url}}/images/devops-ps-1.png)

![image]({{site.url}}/images/devops-ps-2.png)

With that done, I validated my connection to Azure DevOps via [PAT](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows): 

![image]({{site.url}}/images/devops-ps-3.png)

Now, I was ready to script this automation, which I have shown below with comments to explain main steps:

***NOTE:  i heavily leveraged [source](https://github.com/MethodsAndPractices/vsteam/tree/trunk/Source/Public) and this [issue](https://github.com/MethodsAndPractices/vsteam/issues/339) to find the cmdlets and experimented to get the below script working:*** 

***Also this script is bare minimum and does not perform any validation checks, which you might have to implement on your end*** 

~~~powershell

# Get the CI Build Definition
$buildDefinition = Get-VSTeamBuildDefinition -Id 4   -Raw # Use the Id here since its the only Build in the project

# Get the Build Task Group by its name 
$buildTask = Get-VSTeamTaskGroup -Name "Update Tags"

# retrieve the steps of the build
$steps = $buildDefinition.process.phases[0].steps

# define task to import
$metaTagsTask = [PSCustomObject]@{
    displayName = $buildTask.Name
    task = $buildTask.tasks[0].task
	inputs = $buildTask.tasks[0].inputs
	id = $buildTask.tasks[0].task.id
    enabled    = "True"
}

# update Definitions Tasks
$buildDefinition.process.phases[0].steps = $steps + $metaTagsTask

# add Comment
Add-Member -InputObject $buildDefinition -NotePropertyName "comment" -NotePropertyValue "Imported Update Tags" -Force

# Update Definition
$buildJson = $buildDefinition | ConvertTo-Json -Depth 100
Update-VSTeamBuildDefinition -ProjectName AzureFunctionDeployment -Id 4 -BuildDefinition $buildJson

~~~

So there you see folks... depending on your preference, you can leverage this option as well. For my use case we ended up leveraging the .NET Client libraries

***NOTE: I'll leave the updating of CD pipeline as an  exercise for you to explore 😉 and keep this post short.***


