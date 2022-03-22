---
layout:     post
title:      Update Azure Resource Tags via DevOps
date:       2022-03-21
summary:    This post explores how to update Azure Function Tags via Azure DevOps leveraging PowerShell.
categories: Azure DevOps, PowerShell 
---

In a recent development at work, we were looking for a way to expose certain metadata about our resources into the Azure Portal. This came out of that fact that we were trying to troubleshoot an issue with our functions, only to find out that one of it's dependencies was not updated to the correct version.

So the thought came, what if we surface up or display that dependency version on that resource's portal page, we can validate the correct deployment is in place before we dive any further into code troubleshooting and that can then potentialy reduce our troubleshooting efforts in future.

So for this reason, we decided to export that dependency version and any other related metadata to the portal using [Azure Resource Tags](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources?tabs=json)   and this post explores that further.

Now, like any other continuous integration, we were leveragining Azure CI/CD pipelines for deploying our Azure Functions, however ***we were still using the [Classic](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started/pipelines-get-started?msclkid=16444b80a95311ec88cb9ec51b8851cd&view=azure-devops#define-pipelines-using-the-classic-interface) model instead of the [YAML](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started/pipelines-get-started?msclkid=16444b80a95311ec88cb9ec51b8851cd&view=azure-devops#define-pipelines-using-yaml-syntax)***

Alright, so my first step was to start with the some built-in task, that could extract and output data which could be processed by the deployment pipeline and for that I leveraged the [PowerShell](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/powershell?msclkid=d2b76f8fa95311ec95652e109ece21bc&view=azure-devops) task as shown below:

![image]({{site.url}}/images/powershell-1.png)

After importing the task, I configured the inline script as shown below: 

~~~powershell
# Load the passed in csproj file
$xml = [xml](Get-Content -Path $(csprojfile))

# Get Metadata
$functionVersion = $xml.Project.ItemGroup.PackageReference | Where-Object {$_.Include -eq "Microsoft.NET.Sdk.Functions"} | Select Version
$buildData = "$(Build.BuildNumber) - $(Build.DefinitionName) - $(Build.Repository.Name) - $(Build.SourceBranch) - $(Build.SourceVersion)"

# Write Metadata to file
Add-Content -Path $(Build.ArtifactStagingDirectory)\Tags.txt -Value "Common Lib. Version : $functionVersion"
Add-Content -Path $(Build.ArtifactStagingDirectory)\Tags.txt -Value "BuildDetails : $buildData"
~~~

The above code is self-explanatory, we first extract out the Functions SDK version from the .csproj file and output that along with other [build](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?msclkid=9afcf81aaa2611ecaf9ecd989a5756c9&view=azure-devops&tabs=yaml#build-variables-devops-services) information to a file.

***NOTE: In order to reuse this task in many pipelines, I have converted it to a [Task Group](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/task-groups?msclkid=70d17771a95611ec986e92ff7f24881f&view=azure-devops) and parmeterized the csproj file. If you use YAML, then the same could be done via [Templates](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops)***


Following is the snapshot of using that task in a CI pipeline:

![image]({{site.url}}/images/powershell-2.png)

Running the above pipeline and looking at the artifacts generated we can verify that tags are exported as expected:

![image]({{site.url}}/images/powershell-3.png)

![image]({{site.url}}/images/powershell-4.png)

Alright, so with CI in place, next step was to update the CD pipeline to get these tags and create / append that to the Azure Function resource. This is where I used [Azure Powershell](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-powershell?msclkid=f88d6795a95711eca5a21efc64e87c28&view=azure-devops) task with inline script as shown below:

![image]({{site.url}}/images/powershell-8.png)

***NOTE: Like previously this task is also exported as a Task Group for reusability and the appropriate variables are parameterized***

~~~powershell
# Get Metadata tags from file
$newTags = (Get-Content -Path $(tagsfile))

# Get existing tags from the resource
$tags = (Get-AzResource -ResourceGroupName $(resourceGroupName) -ResourceName $(resourceName)).Tags

# Append new tags 
foreach($tag in $newTags) 
{
        $key = $tag.Split(":")[0]
        $value = $tag.Split(":")[1]
         if (! $tags.ContainsKey($key))
         { 
               $tags.Add($key,$value)
         }
         else
         {
               $tags[$key] = $value
         }
}

# Set Tags
Set-AzResource -ResourceGroupName $(resourceGroupName) -ResourceName $(resourceName) -ResourceType Microsoft.Web/sites -Tag $tags -Force
~~~

You can see in the above snippet that we first get the contents of Tags.txt file and then get the existing tags on our function App, only to then append / update and set them accordingly using the [Az module](https://docs.microsoft.com/en-us/powershell/azure/new-azureps-module-az?view=azps-7.3.2#:~:text=Overview%20The%20Az%20PowerShell%20module%20is%20a%20set,example%20in%20the%20context%20of%20a%20CI%2FCD%20pipeline.?msclkid=27180780aa2911ec90e6091fe5a9d4e4). 

Like the last task, this one is also exported to a Task group for reusability and the values like tagsfile, resourceGroupName, etc are parameterized accordingly.

Following is the snapshot of using this task in a deployment pipeline:

![image]({{site.url}}/images/powershell-5.png)

![image]({{site.url}}/images/powershell-6.png)

And finally running the deployment pipeline, you can see our tags are displayed on Function resource portal page accordingly as shown below:

![image]({{site.url}}/images/powershell-7.png)


So...That's it, I am sure there are other ways to accomplish this. Feel free to leave your thoughts as well!!!














