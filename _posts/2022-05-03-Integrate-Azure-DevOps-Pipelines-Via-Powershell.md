---
layout:     post
title:      Integrate Azure DevOps Pipelines via Powershell
date:       2022-05-03
summary:    This post explores how to automate Azure DevOps pipelines via its Powershell module
categories: Azure DevOps, Powershell
---
This post is going to be really short 😊. 

If you read the [last]({{site.url}}/Integrate-Azure-DevOps-Pipelines-Via-Code) post, we saw how you can integrate Azure DevOps via its [.NET Client Libraries](https://docs.microsoft.com/en-us/azure/devops/integrate/concepts/dotnet-client-libraries?view=azure-devops) to automate task import into our CI/CD pipelines.

Now since I mentioned another [VSTeam Powershell](https://github.com/MethodsAndPractices/vsteam) option  as well, in this post we will see how to automate atleast one of those earlier tasks using it.




If you read through the VSTeam's link, the first thing we have to do is download that module. So lets get started...

When i first started powershell, i updated it to the latest version and installed the module as shown below:

![image]({{site.url}}/images/devops-ps-1.png)

![image]({{site.url}}/images/devops-ps-2.png)

After validating the connection to Azure DevOps via [PAT](), i was ready to script it.

![image]({{site.url}}/images/devops-ps-3.png)

I have shown the script below with comments included to explain steps:

***NOTE:  i heavily leveraged [source](https://github.com/MethodsAndPractices/vsteam/tree/trunk/Source/Public) to find the cmdlets and experimented to get the below script working: 

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


***NOTE: I'll leave the updating of CD pipeline as an  exercise for you to explore, since in the actual implementation we ended up using the DevOps Client libraries.***


