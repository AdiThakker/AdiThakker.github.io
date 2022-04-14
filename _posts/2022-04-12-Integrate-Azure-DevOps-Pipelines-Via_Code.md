---
layout:     post
title:      Integrate Azure DevOps Pipelines via Code
date:       2022-04-14
summary:    This post explores how to automate Azure DevOps pipelines via its API
categories: Azure DevOps, .NET, PowerShell
---

If you have read the [last]({{site.url}}/Update-Azure-Resource-Tags-Via-DevOps) post, you saw that we built custom [Task Group](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/task-groups?msclkid=70d17771a95611ec986e92ff7f24881f&view=azure-devops) for creating reusable tasks. This was a good start since these tasks could be imported into our CI/CD pipelines. However we soon realized that manually updating several of our CI / CD was very tedious task 😉. So, the thought was hey, can we automate this?

A quick research resulted in couple options:

- [.NET Client Libraries](https://docs.microsoft.com/en-us/azure/devops/integrate/concepts/dotnet-client-libraries?view=azure-devops) available via NuGet to script Azure DevOps automation.

- [VSTeam Powershell](https://github.com/MethodsAndPractices/vsteam) modules that let you integrate with Azure DevOps.


***NOTE: This blog post explores the first option. There is also the REST api as well. You can refer that [here](https://docs.microsoft.com/en-us/azure/devops/integrate/rest-api-overview?view=azure-devops). The second option of VS Team Powershell we'll explore in another post.***


The ***simplified and exploratory*** version is available [here](https://github.com/AdiThakker/AzureDevOps.Integration) and is influenced by this [reference](https://github.com/microsoft/azure-devops-dotnet-samples), so with that in place, lets dive in.

Alright, reading through the docs... tells us that we need the following nuget libs to get started:

~~~xml
<PackageReference Include="Microsoft.TeamFoundation.DistributedTask.Common.Contracts" Version="16.170.0" />
<PackageReference Include="Microsoft.TeamFoundation.DistributedTask.WebApi" Version="16.170.0" />
<PackageReference Include="Microsoft.TeamFoundationServer.Client" Version="16.170.0" />
<PackageReference Include="Microsoft.VisualStudio.Services.Client" Version="16.170.0" />
<PackageReference Include="Microsoft.VisualStudio.Services.InteractiveClient" Version="16.170.0" />
<PackageReference Include="Microsoft.VisualStudio.Services.Release.Client" Version="16.170.0" />
~~~

Next, our [Program.cs](https://github.com/AdiThakker/AzureDevOps.Integration/blob/main/AzureDevOps.Integration/Program.cs) is straight forward and is shown below:

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

We first connect to Azure DevOps and retrieve the configured Repository after which we update the build and release definitions.The fluent syntax method calls are coded in [DevOpsExtensions](https://github.com/AdiThakker/AzureDevOps.Integration/blob/main/AzureDevOps.Integration/DevOpsExtensions.cs) class which is shown below:

~~~csharp
static string token = string.Empty;
static string url = string.Empty;
static string project = string.Empty;
static string repo = string.Empty;


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

There are [several](https://docs.microsoft.com/en-us/azure/devops/integrate/rest-api-overview?view=azure-devops#create-the-request) ways to Authenticate with DevOps and depending on your use case, you can use the appropriate one accordingly. In the above snippet, we are leveraging [PAT](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows) for our example. We wrap the instance of [VssConnection](https://docs.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2013/dn245455(v=vs.120) and return that in our custom [DevOpsContext](https://github.com/AdiThakker/AzureDevOps.Integration/blob/main/AzureDevOps.Integration/Models/DevOpsContext.cs) object.

Once we have the connection set, we get the configured repository via [GitHttpClient](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.sourcecontrol.webapi.githttpclient?view=azure-devops-dotnet) instance. ***NOTE: The .Net Client Libraries API use specific typed clients for accessing relevant parts of DevOps. This will be apparent in the following snippets.***

~~~csharp
public static DevOpsContext GetConfiguredRepository(this DevOpsContext context)
{
    using var gitClient = context.Connection.GetClient<GitHttpClient>();

    var repository = gitClient.GetRepositoryAsync(project, repo).Result;
    var properties = new Dictionary<string, object>();

    return new DevOpsContext(context.Connection, properties);
}
~~~

The next step is to retrieve and update the build definition with the relevant build Task from our Task group as shown below:

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

    // Get tag Group
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
In the ***UpdateLatestBuildDefinitionsWithTagTask*** method you can see that Task Groups are accessed via [TaskAgentHttpClient](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.distributedtask.webapi.taskagenthttpclient?view=azure-devops-dotnet) class and we filter to find the relevant build task from that task group. Then we create [BuildDefinitionStep](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.build.webapi.builddefinitionstep?view=azure-devops-dotnet) instance and set its inputs from our [TaskGroup](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.distributedtask.webapi.taskgroup?view=azure-devops-dotnet) instance. We then use [BuildHttpClient](https://docs.microsoft.com/en-us/dotnet/api/microsoft.teamfoundation.build.webapi.buildhttpclient?view=azure-devops-dotnet) to update our build definition. 

Following is the snapshot of the build after the update:

![image]({{site.url}}/images/devops-1.png)

Updating the release definition follows a similar approach. 

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

- snapshot



So...that's it folks, I am sure there are numerous ways to accomplish this. If you know of any, feel free to leave your comments!!!














