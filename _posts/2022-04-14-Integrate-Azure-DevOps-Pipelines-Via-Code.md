---
layout:     post
title:      Integrate Azure DevOps Pipelines via code
date:       2022-04-14
summary:    This post explores how to automate Azure DevOps pipelines via its API
categories: Azure DevOps, .NET
---

In the [last]({{site.url}}/Update-Azure-Resource-Tags-Via-DevOps) post, you saw that we built custom [Task Group](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/task-groups?msclkid=70d17771a95611ec986e92ff7f24881f&view=azure-devops) for creating reusable tasks. This gave us the ability to import them into our CI/CD pipelines. 

However, we soon realized that manually updating several of our CI / CD pipelines was very tedious task 😉 itself. 

So, the thought was hey, can we automate this? And a quick research resulted in following options:

- [.NET Client Libraries](https://docs.microsoft.com/en-us/azure/devops/integrate/concepts/dotnet-client-libraries?view=azure-devops) available via NuGet to script Azure DevOps automation.

- [VSTeam Powershell](https://github.com/MethodsAndPractices/vsteam) modules that let you integrate with Azure DevOps.


***NOTE: This blog post explores the first option. There is also a [REST](https://docs.microsoft.com/en-us/azure/devops/integrate/rest-api-overview?view=azure-devops) API available, which we are not going to cover.***


I have made all the code available [here](https://github.com/AdiThakker/AzureDevOps.Integration) and it is heavily influenced by this [reference](https://github.com/microsoft/azure-devops-dotnet-samples), so with that in place, lets dive in.

Reading through the docs tells us that we first need the following NuGet libraries to get started:

~~~xml
<PackageReference Include="Microsoft.TeamFoundation.DistributedTask.Common.Contracts" Version="16.170.0" />
<PackageReference Include="Microsoft.TeamFoundation.DistributedTask.WebApi" Version="16.170.0" />
<PackageReference Include="Microsoft.TeamFoundationServer.Client" Version="16.170.0" />
<PackageReference Include="Microsoft.VisualStudio.Services.Client" Version="16.170.0" />
<PackageReference Include="Microsoft.VisualStudio.Services.InteractiveClient" Version="16.170.0" />
<PackageReference Include="Microsoft.VisualStudio.Services.Release.Client" Version="16.170.0" />
~~~

Next, if you look at our [Program.cs](https://github.com/AdiThakker/AzureDevOps.Integration/blob/main/AzureDevOps.Integration/Program.cs), its very straight forward and is shown below:

~~~csharp
using AzureDevOps.Integration;
using Microsoft.Extensions.Configuration;

// See https://aka.ms/new-console-template for more information
Console.WriteLine("Hello, Azure DevOps!");

// Setup configuration builder
var builder  = new ConfigurationBuilder()
                    .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                    .AddUserSecrets<Program>();

// Connect to Azure DevOps
var devOps = builder
                .Build()
                .ConnectToAzureDevOps()
                .GetConfiguredRepository();

// update build definition
devOps
    .GetLatestBuildDefinition()
    .UpdateLatestBuildDefinitionsWithTagTask();

// update release definition
devOps
    .GetLatestReleaseDefinition()
    .UpdateLatestReleaseDefinitionsWithTagTask();

Console.WriteLine("Definitions Updated!");
Console.ReadKey();
~~~

In the above code, we first connect to Azure DevOps and retrieve the configured repository, after which we update the build and release definitions. The fluent syntax is enabled via [DevOpsExtensions](https://github.com/AdiThakker/AzureDevOps.Integration/blob/main/AzureDevOps.Integration/DevOpsExtensions.cs) class and following are some of its relevant snippets.


 ### Connecting to Azure DevOps

 There are [several](https://docs.microsoft.com/en-us/azure/devops/integrate/rest-api-overview?view=azure-devops#create-the-request) ways to Authenticate with Azure DevOps and depending on your use case, you can use the appropriate one. 
 
For our use case, we are leveraging [PAT](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows) via ***VssConnection*** wrapped in our custom [DevOpsContext](https://github.com/AdiThakker/AzureDevOps.Integration/blob/main/AzureDevOps.Integration/Models/DevOpsContext.cs) object.

~~~csharp
public static DevOpsContext ConnectToAzureDevOps(this IConfigurationRoot configuration)
{
    token = configuration.GetValue<string>("token");
    url = configuration.GetValue<string>("url");
    project = configuration.GetValue<string>("project");
    repo = configuration.GetValue<string>("repo");

    // return instance of DevOpsContext wrapping VssConnection using Personal Access Token
    return new DevOpsContext(new VssConnection(new Uri(url), new VssBasicCredential(string.Empty, token)));
}
~~~


### Retrieving Repository

Once we have the connection set, we get the configured repository via [GitHttpClient](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.sourcecontrol.webapi.githttpclient?view=azure-devops-dotnet) instance. 

~~~csharp
public static DevOpsContext GetConfiguredRepository(this DevOpsContext context)
{
    using var gitClient = context.Connection.GetClient<GitHttpClient>();

    var repository = gitClient.GetRepositoryAsync(project, repo).Result;
    var properties = new Dictionary<string, object>();

    return new DevOpsContext(context.Connection, properties);
}
~~~

***NOTE: The .Net Client Libraries API use specific typed HTTP clients for accessing relevant parts of Azure DevOps. This will be apparent in the various snippets through this post.***

### Updating the Build Definition

The next step is to retrieve and update the [BuildDefinition](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.build.webapi.builddefinition?view=azure-devops-dotnet).

~~~csharp
public static DevOpsContext GetLatestBuildDefinition(this DevOpsContext context)
{
    var buildClient = context.Connection.GetClient<BuildHttpClient>();
    var latestDefinition = buildClient.GetFullDefinitionsAsync(project: project).Result.FirstOrDefault();
    Dictionary<string, object> properties = context.Properties ?? new Dictionary<string, object>();
    properties.Add(buildDefinitionKey, latestDefinition ?? new BuildDefinition() { Name = "Adding Meta Tags Task" });

    return new DevOpsContext(context.Connection, properties);
}

public static BuildDefinition UpdateLatestBuildDefinitionsWithTagTask(this DevOpsContext context)
{
    var buildDefinition = context.Properties[buildDefinitionKey] as BuildDefinition;
    var process = buildDefinition?.Process as DesignerProcess;
    if (process == null)
        process = new DesignerProcess();

    var taskClient = context.Connection.GetClient<TaskAgentHttpClient>();
    var taskGroups = taskClient.GetTaskGroupsAsync(project).Result;

    // Get tag task from task group
    var taskGroup = taskGroups.First(group => group.Name == "Export-MetaTags");
    var tagTask = taskGroup.Tasks.First();

    // import to the build definition
    var phase = process.Phases.First();
    var buildDefinitionStep = new BuildDefinitionStep()
    {
        DisplayName = taskGroup.Name,
        AlwaysRun = tagTask.AlwaysRun,
        Condition = tagTask.Condition,
        ContinueOnError = tagTask.ContinueOnError,
        Enabled = tagTask.Enabled,
        Environment = tagTask.Environment,
        TimeoutInMinutes = tagTask.TimeoutInMinutes,
        TaskDefinition = new Microsoft.TeamFoundation.Build.WebApi.TaskDefinitionReference()
        {
            DefinitionType = "metaTask",
            Id = taskGroup.Id,
            VersionSpec = taskGroup.Version
        }
    };

    // set step inputs
    buildDefinitionStep.Inputs = new Dictionary<string, string>();
    taskGroup.Inputs.ToList().ForEach(input => buildDefinitionStep.Inputs.Add(input.Name, @"$(System.DefaultWorkingDirectory)\**\*.csproj"));
    phase.Steps.Add(buildDefinitionStep);

    // Update build definition
    buildDefinition.Comment = "Updated with Export tag task";
    try
    {
        var buildClient = context.Connection.GetClient<BuildHttpClient>();
        buildDefinition = buildClient.UpdateDefinitionAsync(buildDefinition).Result;
        var steps = buildDefinition.GetProcess<DesignerProcess>().Phases.First().Steps.ToList();
        return buildDefinition;
    }
    catch (AggregateException aex)
    {
        throw new Exception(aex.Flatten().ToString());
    }
}
~~~
In the above code, the intersting stuff is in the ***UpdateLatestBuildDefinitionsWithTagTask*** function, where you can see that our build task is retrieved from Task Groups via [TaskAgentHttpClient](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.distributedtask.webapi.taskagenthttpclient?view=azure-devops-dotnet). Then we create [BuildDefinitionStep](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.build.webapi.builddefinitionstep?view=azure-devops-dotnet) and set its inputs from our [TaskGroup](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.distributedtask.webapi.taskgroup?view=azure-devops-dotnet) instance. 

Finally, we use [BuildHttpClient](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.build.webapi.buildhttpclient?view=azure-devops-dotnet) to update our [BuildDefinition](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.build.webapi.builddefinition?view=azure-devops-dotnet). 

Following is the snapshot of the updated build:

![image]({{site.url}}/images/devops-1.png)

### Updating the Release Definition

Updating the release definition follows a similar approach.

We use ***ReleaseHttpClient*** class to retrieve and update the [ReleaseDefinition](https://docs.microsoft.com/en-us/javascript/api/azure-devops-extension-api/releasedefinition) by its [Environments](https://docs.microsoft.com/en-us/javascript/api/azure-devops-extension-api/releasedefinitionenvironment) property. Each [DeployPhase](https://docs.microsoft.com/en-us/javascript/api/azure-devops-extension-api/deployphase) has a collection [WorkflowTask](https://docs.microsoft.com/en-us/javascript/api/azure-devops-extension-api/workflowtask), where we append our tag task and finally update the [ReleaseDefinition](https://docs.microsoft.com/en-us/javascript/api/azure-devops-extension-api/releasedefinition). 

All of this is shown below:

~~~csharp
public static DevOpsContext GetLatestReleaseDefinition(this DevOpsContext context)
{
    var releaseClient = context.Connection.GetClient<ReleaseHttpClient>();
    var latestDefinition = releaseClient.GetReleaseDefinitionsAsync(project: project, "", ReleaseDefinitionExpands.Environments).Result.FirstOrDefault();
    Dictionary<string, object> properties = context.Properties ?? new Dictionary<string, object>();
    properties.Add(releaseDefinitionKey, latestDefinition ?? new ReleaseDefinition() { Name = "Adding Meta Tags Task" });
    return new DevOpsContext(context.Connection, properties);
}

public static ReleaseDefinition UpdateLatestReleaseDefinitionsWithTagTask(this DevOpsContext context)
{
    // Get release definition
    var releaseDefinition = context.Properties[releaseDefinitionKey] as ReleaseDefinition;

    // retrieve task from task group
    var taskClient = context.Connection.GetClient<TaskAgentHttpClient>();
    var taskGroups = taskClient.GetTaskGroupsAsync(project).Result;
    var taskGroup = taskGroups.First(group => group.Name == "Update Tags");
    var tagTask = taskGroup.Tasks.First();

    // Get latest release definition by env.
    var releaseClient = context.Connection.GetClient<ReleaseHttpClient>();
    var releaseDefinitionByEnv = releaseClient.GetReleaseDefinitionAsync(project, releaseDefinition.Environments.First().Id).Result;

    // Get definition steps
    var workflowTasks = releaseDefinitionByEnv.Environments.First().DeployPhases.First().WorkflowTasks;
    var worklflowTask = new WorkflowTask()
    {
        Name = taskGroup.Name,
        Version = taskGroup.Version,
        AlwaysRun = tagTask.AlwaysRun,
        Condition = tagTask.Condition,
        ContinueOnError = tagTask.ContinueOnError,
        Enabled = tagTask.Enabled,
        TimeoutInMinutes = tagTask.TimeoutInMinutes,
        TaskId = taskGroup.Id,
        DefinitionType = "metaTask",
            
    };
    worklflowTask.Inputs = new Dictionary<string, string>();
    taskGroup.Inputs.ToList().ForEach(input =>
    {
        var value = input.Name switch
        {
            "AzureSubscription" => "",
            "resourceGroupName" => "resource group name param",
            "resourceName" => "resource name param",
            "tagsfile" => @"$(System.DefaultWorkingDirectory)\**\Tags.txt",
            _ => ""
        };
        worklflowTask.Inputs.Add(input.Name, value);
    });

    // Update definition
    releaseDefinitionByEnv.Environments.First().DeployPhases.First().WorkflowTasks.Add(worklflowTask);
    releaseDefinitionByEnv.Comment = "Updated with Update tag task";

    try
    {
        releaseDefinitionByEnv = releaseClient.UpdateReleaseDefinitionAsync(releaseDefinitionByEnv, project).Result;
    }
    catch (AggregateException aex)
    {
        throw new Exception(aex.Flatten().ToString());
    }
    
    return releaseDefinitionByEnv;
}
~~~

Finally updating the CD Pipeline with the release definition results in the following:

![image]({{site.url}}/images/devops-2.png)

So there you see folks, the above coding exercise is still very exploratory in nature and documentation is very lean around it so this definitely needs to be hardened further which i'll continue doing on my end.

Meanwhile in the next post we will try to explore the PowerShell module around this as well. 

So, see you then!!! 














