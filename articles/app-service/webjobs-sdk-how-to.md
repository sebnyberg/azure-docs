---
title: How to use the WebJobs SDK - Azure
description: Learn more about how to write code for the WebJobs SDK. Create  event-driven background processing jobs that access data in Azure services and third-party services.
services: app-service\web, storage
documentationcenter: .net
author: ggailey777
manager: jeconnoc
editor: 

ms.service: app-service-web
ms.workload: web
ms.tgt_pltfrm: na
ms.devlang: dotnet
ms.topic: article
ms.date: 02/18/2019
ms.author: glenga
#Customer intent: As an App Services developer, I want use the WebJobs SDK to be able to execute event-driven code in Azure.
---

# How to use the Azure WebJobs SDK for event-driven background processing

This article provides guidance on how to work with the Azure WebJobs SDK. To get started with WebJobs right away, see [Get started with the Azure WebJobs SDK for event-driven background processing](webjobs-sdk-get-started.md). 

## WebJobs SDK versions

The following are key differences in version 3.x of the WebJobs SDK compared to version 2.x:

* Version 3.x adds support for .NET Core.
* In version 3.x, you must explicitly install the Storage binding extension required by the WebJobs SDK. In version 2.x, the Storage bindings were included in the SDK.
* Visual Studio tooling for .NET Core (3.x) projects differs from .NET Framework (2.x) projects. For more information, see [Develop and deploy WebJobs using Visual Studio - Azure App Service](webjobs-dotnet-deploy-vs.md).

When possible, examples are provides for both version 3.x and version 2.x.

> [!NOTE]
> [Azure Functions](../azure-functions/functions-overview.md) is built on the WebJobs SDK, and this article links to Azure Functions documentation for some topics. The following are differences between Functions and the WebJobs SDK:
> * Azure Functions version 2.x corresponds to WebJobs SDK version 3.x, and Azure Functions 1.x corresponds to WebJobs SDK 2.x. Source code repositories follow the WebJobs SDK numbering.
> * Sample code for Azure Functions C# class libraries is like WebJobs SDK code except you don't need a `FunctionName` attribute in a WebJobs SDK project.
> * Some binding types are only supported in Functions, such as HTTP, webhook, and Event Grid (which is based on HTTP).
>
> For more information, see [Compare the WebJobs SDK and Azure Functions](../azure-functions/functions-compare-logic-apps-ms-flow-webjobs.md#compare-functions-and-webjobs).

## WebJobs host

The host is a runtime container for functions.  It listens for triggers and calls functions. In version 3.x, the host is an implementation of `IHost`, and in version 2.x you use the `JobHost` object. You create a host instance in your code and write code to customize its behavior.

This is a key difference between using the WebJobs SDK directly and using it indirectly by using Azure Functions. In Azure Functions, the service controls the host, and you can't customize it by writing code. Azure Functions lets you customize host behavior through settings in the *host.json* file. Those settings are strings, not code, which limits the kinds of customizations you can do.

### Host connection strings

The WebJobs SDK looks for Azure Storage and Azure Service Bus connection strings in the *local.settings.json* file when you run locally, or in the WebJob's environment when you run in Azure. By default, a Storage connection string setting named `AzureWebJobsStorage` is required.  

Version 2.x of the SDK lets you use your own names for these connection strings, or store them elsewhere. You can set them in code, as shown here:

```cs
static void Main(string[] args)
{
    var _storageConn = ConfigurationManager
        .ConnectionStrings["MyStorageConnection"].ConnectionString;

    //// Dashboard logging is deprecated; use Application Insights.
    //var _dashboardConn = ConfigurationManager
    //    .ConnectionStrings["MyDashboardConnection"].ConnectionString;

    JobHostConfiguration config = new JobHostConfiguration();
    config.StorageConnectionString = _storageConn;
    //config.DashboardConnectionString = _dashboardConn;
    JobHost host = new JobHost(config);
    host.RunAndBlock();
}
```

Because it uses the default .NET Core configuration APIs, there is no API in version 3.x to change the connection string names.

### Host development settings

You can run the host in development mode to make local development more efficient. Here are some of the settings that are changed when running in development mode:

| Property | Development setting |
| ------------- | ------------- |
| `Tracing.ConsoleLevel` | `TraceLevel.Verbose` to maximize log output. |
| `Queues.MaxPollingInterval`  | A low value to ensure queue methods are triggered immediately.  |
| `Singleton.ListenerLockPeriod` | 15 seconds to aid in rapid iterative development. |

The way that you enable development mode depends on the SDK version. 

#### Version 3.x

Version 3.x uses the standard ASP.NET Core APIs. Call the [UseEnvironment](/dotnet/api/microsoft.extensions.hosting.hostinghostbuilderextensions.useenvironment) method on the [`HostBuilder`](/dotnet/api/microsoft.extensions.hosting.hostbuilder) instance. Pass a string named `development`, as in the following example:

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.UseEnvironment("development");
    builder.ConfigureWebJobs(b =>
            {
                b.AddAzureStorageCoreServices();
            });
    var host = builder.Build();
    using (host)
    {
        host.Run();
    }
}
```

#### Version 2.x

The `JobHostConfiguration` class has a `UseDevelopmentSettings` method that enables development mode.  The following example shows how to use development settings. To make `config.IsDevelopment` return `true` when running locally, set a local environment variable named `AzureWebJobsEnv` with value `Development`.

```cs
static void Main()
{
    config = new JobHostConfiguration();

    if (config.IsDevelopment)
    {
        config.UseDevelopmentSettings();
    }

    var host = new JobHost(config);
    host.RunAndBlock();
}
```

### <a name="jobhost-servicepointmanager-settings"></a>Managing concurrent connections (v2.x)

In version 3.x, the connection limit defaults to infinite connections. If for some reason you need to change this limit, you can use the [MaxConnectionsPerServer](/dotnet/api/system.net.http.winhttphandler.maxconnectionsperserver) property of the [WinHttpHander](/dotnet/api/system.net.http.winhttphandler) class.

For version 2.x, you control the number of concurrent connections to a host by using the [ServicePointManager.DefaultConnectionLimit](https://msdn.microsoft.com/library/system.net.servicepointmanager.defaultconnectionlimit) API. In 2.x, you should increase this value from the default of 2 before starting your WebJobs host.

All outgoing HTTP requests that you make from a function by using `HttpClient` flow through the `ServicePointManager`. Once you hit the `DefaultConnectionLimit`, the `ServicePointManager` starts queueing requests before sending them. Suppose your `DefaultConnectionLimit` is set to 2 and your code makes 1,000 HTTP requests. Initially, only two requests are allowed through to the OS. The other 998 are queued until there’s room for them. That means your `HttpClient` may time out, because it *thinks* it’s made the request, but the request was never sent by the OS to the destination server. So you might see behavior that doesn't seem to make sense: your local `HttpClient` is taking 10 seconds to complete a request, but your service is returning every request in 200 ms. 

The default value for ASP.NET applications is `Int32.MaxValue`, and that's likely to work well for WebJobs running in a Basic or higher App Service plan. WebJobs typically need the Always On setting, and that's supported only by Basic and higher App Service plans. 

If your WebJob is running in a Free or Shared App Service Plan, your application is restricted by the App Service sandbox, which currently has a [connection limit of 300](https://github.com/projectkudu/kudu/wiki/Azure-Web-App-sandbox#per-sandbox-per-appper-site-numerical-limits). With an unbound connection limit in `ServicePointManager`, it's more likely that the sandbox connection threshold will be reached and the site shut down. In that case, setting `DefaultConnectionLimit` to something lower, like 50 or 100, can prevent this from happening and still allow for sufficient throughput.

The setting must be configured before any HTTP requests are made. For this reason, the WebJobs host shouldn't try to adjust the setting automatically; there may be HTTP requests that occur before the host starts and this can lead to unexpected behavior. The best approach is to set the value immediately in your `Main` method before initializing the `JobHost`, as shown in the following example

```csharp
static void Main(string[] args)
{
    // Set this immediately so that it is used by all requests.
    ServicePointManager.DefaultConnectionLimit = Int32.MaxValue;

    var host = new JobHost();
    host.RunAndBlock();
}
```

## Triggers

Functions must be public methods and must have one trigger attribute or the [NoAutomaticTrigger](#manual-trigger) attribute.

### Automatic trigger

Automatic triggers call a function in response to an event. Consider the following example of a function triggered by a message added to Azure Queue storage that reads a blob from Azure Blog storage:

```cs
public static void Run(
    [QueueTrigger("myqueue-items")] string myQueueItem,
    [Blob("samples-workitems/{myQueueItem}", FileAccess.Read)] Stream myBlob,
    ILogger log)
{
    log.LogInformation($"BlobInput processed blob\n Name:{myQueueItem} \n Size: {myBlob.Length} bytes");
}
```

The `QueueTrigger` attribute tells the runtime to call the function whenever a queue message appears in the `myqueue-items` queue. The `Blob` attribute tells the runtime to use the queue message to read a blob in the *sample-workitems* container. The content of the queue message, passed into the function in the `myQueueItem` parameter, is the name of the blob.


### Manual trigger

To trigger a function manually, use the `NoAutomaticTrigger` attribute, as shown in the following example:

```cs
[NoAutomaticTrigger]
public static void CreateQueueMessage(
ILogger logger,
string value,
[Queue("outputqueue")] out string message)
{
    message = value;
    logger.LogInformation("Creating queue message: ", message);
}
```

The way that you manually trigger the function depends on the SDK version.

#### Version 3.x

```cs
static async Task Main(string[] args)
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
        b.AddAzureStorage();
    });
    var host = builder.Build();
    using (host)
    {
        var jobHost = host.Services.GetService(typeof(IJobHost)) as JobHost;
        var inputs = new Dictionary<string, object>
        {
            { "value", "Hello world!" }
        };

        await host.StartAsync();
        await jobHost.CallAsync("CreateQueueMessage", inputs);
        await host.StopAsync();
    }
}
```

#### Version 2.x

```cs
static void Main(string[] args)
{
    JobHost host = new JobHost();
    host.Call(typeof(Program).GetMethod("CreateQueueMessage"), new { value = "Hello world!" });
}
```

## Input and output bindings

Input bindings provide a declarative way to make data from Azure or third-party services available to your code. Output bindings provide a way to update data. The [Get started article](webjobs-sdk-get-started.md) shows an example of each.

You can use a method return value for an output binding, by applying the attribute to the method return value. See the example in the Azure Functions [Triggers and bindings](../azure-functions/functions-bindings-return-value.md) article.

## Binding types

The way that binding types are installed and managed differs between version 3.x and 2.x of the SDK. You can find the package to install for a particular binding type in the **Packages** section of that binding type's [reference article](#binding-reference-information) for Azure Functions. An exception is the Files trigger and binding (for the local file system), which is not supported by Azure Functions.

#### Version 3.x

In version 3.x, the storage bindings are included in the `Microsoft.Azure.WebJobs.Extensions.Storage` package. Call the `AddAzureStorage` extension method in `ConfigureWebJobs` method as shown in the following example:

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
            {
                b.AddAzureStorageCoreServices();
                b.AddAzureStorage();
            });
    var host = builder.Build();
    using (host)
    {
        host.Run();
    }
}
```

To use other trigger and binding types, install the NuGet package that contains them and call the `Add<binding>` extension method implemented in the extension. For example, if you want to use an Azure Cosmos DB binding, install `Microsoft.Azure.WebJobs.Extensions.CosmosDB` and call `AddCosmosDB`, as in the following example:

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
            {
                b.AddAzureStorageCoreServices();
                b.AddCosmosDB();
            });
    var host = builder.Build();
    using (host)
    {
        host.Run();
    }
}
```

To use the Timer trigger or the Files binding, which are part of core services, call the `AddTimers` or `AddFiles` extension methods, respectively.

#### Version 2.x

The following trigger and binding types are included in version 2.x of the `Microsoft.Azure.WebJobs` package:

* Blob storage
* Queue storage
* Table storage

To use other trigger and binding types, install the NuGet package that contains them and call a `Use<binding>` method on the `JobHostConfiguration` object. For example, if you want to use a Timer trigger, install `Microsoft.Azure.WebJobs.Extensions` and call `UseTimers` in the `Main` method, as in this example:

```cs
static void Main()
{
    config = new JobHostConfiguration();
    config.UseTimers();
    var host = new JobHost(config);
    host.RunAndBlock();
}
```

To use the Files binding, install `Microsoft.Azure.WebJobs.Extensions` and call `UseFiles`.

### ExecutionContext

WebJobs lets you bind to an [`ExecutionContext`]. With this binding, you can access the [`ExecutionContext`] as a parameter in your function signature. For example, the following code uses the context object to access the invocation ID, which you can use to correlate all logs produced by a given function invocation.  

```cs
public class Functions
{
    public static void ProcessQueueMessage([QueueTrigger("queue")] string message,
        ExecutionContext executionContext,
        ILogger logger)
    {
        logger.LogInformation($"{message}\n{executionContext.InvocationId}");
    }
}
```

The way that you bind to the [`ExecutionContext`] depends on your SDK version.

#### Version 3.x

Call the `AddExecutionContextBinding` extension method in `ConfigureWebJobs` method as shown in the following example:

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
            {
                b.AddAzureStorageCoreServices();
                b.AddExecutionContextBinding();
            });
    var host = builder.Build();
    using (host)
    {
        host.Run();
    }
}
```

#### Version 2.x

The `Microsoft.Azure.WebJobs.Extensions` package mentioned earlier also provides a special binding type that you can register by calling the `UseCore` method. This binding lets you define an [`ExecutionContext`] parameter in your function signature, which is enabled as follows:

```cs
class Program
{
    static void Main()
    {
        config = new JobHostConfiguration();
        config.UseCore();
        var host = new JobHost(config);
        host.RunAndBlock();
    }
}
```

## Binding configuration

Some trigger and bindings let you configure their behavior. The way that you configure them depends on the SDK version.

* **Version 3.x:** Configuration is set when the `Add<Binding>` method is called in `ConfigureWebJobs`.
* **Version 2.x:** By setting properties in a configuration object that you pass in to the `JobHost`.

These binding-specific settings are equivalent to settings in the [host.json project file](../azure-functions/functions-host-json.md) in Azure Functions.

You can configure the following bindings:

* [Azure CosmosDB trigger](#azure-cosmosdb-trigger-configuration-version-3x)
* [Event Hubs trigger](#event-hubs-trigger-configuration-version-3x)
* [Queue storage trigger](#queue-trigger-configuration)
* [SendGrid binding](#sendgrid-binding-configuration-version-3x)
* [Service Bus trigger](#service-bus-trigger-configuration-version-3x)

### Azure CosmosDB trigger configuration (version 3.x)

The following example shows how to configure the Azure Cosmos DB trigger:

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
        b.AddCosmosDB(a =>
        {
            a.ConnectionMode = ConnectionMode.Gateway;
            a.Protocol = Protocol.Https;
            a.LeaseOptions.LeasePrefix = "prefix1";

        });
    });
    var host = builder.Build();
    using (host)
    {

        host.Run();
    }
}
```

For more details, see the [Azure CosmosDB binding article](../azure-functions/functions-bindings-cosmosdb-v2.md#hostjson-settings).

### Event Hubs trigger configuration (version 3.x)

The following example shows how to configure the Event Hubs trigger:

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
        b.AddEventHubs(a =>
        {
            a.BatchCheckpointFrequency = 5;
            a.EventProcessorOptions.MaxBatchSize = 256;
            a.EventProcessorOptions.PrefetchCount = 512;
        });
    });
    var host = builder.Build();
    using (host)
    {

        host.Run();
    }
}
```

For more details, see the [Event Hubs binding article](../azure-functions/functions-bindings-event-hubs.md#hostjson-settings).

### Queue trigger configuration

The following examples show how to configure the Storage queue trigger:

#### Version 3.x

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
        b.AddAzureStorage(a => {
            a.BatchSize = 8;
            a.NewBatchThreshold = 4;
            a.MaxDequeueCount = 4;
            a.MaxPollingInterval = TimeSpan.FromSeconds(15);
        });
    });
    var host = builder.Build();
    using (host)
    {

        host.Run();
    }
}
```

For more details, see the [Queue storage binding article](../azure-functions/functions-bindings-storage-queue.md#hostjson-settings).

#### Version 2.x

```cs
static void Main(string[] args)
{
    JobHostConfiguration config = new JobHostConfiguration();
    config.Queues.BatchSize = 8;
    config.Queues.NewBatchThreshold = 4;
    config.Queues.MaxDequeueCount = 4;
    config.Queues.MaxPollingInterval = TimeSpan.FromSeconds(15);
    JobHost host = new JobHost(config);
    host.RunAndBlock();
}
```

For more details, see the [host.json v1.x reference](../azure-functions/functions-host-json-v1.md#queues).

### SendGrid binding configuration (version 3.x)

The following example shows how to configure the SendGrid output binding:

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
        b.AddSendGrid(a =>
        {
            a.FromAddress.Email = "samples@functions.com";
            a.FromAddress.Name = "Azure Functions";
        });
    });
    var host = builder.Build();
    using (host)
    {

        host.Run();
    }
}
```

For more details, see the [SendGrid binding article](../azure-functions/functions-bindings-sendgrid.md#hostjson-settings).

### Service Bus trigger configuration (version 3.x)

The following example shows how to configure the Service Bus trigger:

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
        b.AddServiceBus(sbOptions =>
        {
            sbOptions.MessageHandlerOptions.AutoComplete = true;
            sbOptions.MessageHandlerOptions.MaxConcurrentCalls = 16;
        });
    });
    var host = builder.Build();
    using (host)
    {

        host.Run();
    }
}
```

For more details, see the [Service Bus binding article](../azure-functions/functions-bindings-service-bus.md#hostjson-settings).

### Configuration for other bindings

Some trigger and binding types define their own custom configuration type. For example, the File trigger lets you specify the root path to monitor, as in the following examples:

#### Version 3.x

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
        b.AddFiles(a => a.RootPath = @"c:\data\import");
    });
    var host = builder.Build();
    using (host)
    {

        host.Run();
    }
}
```

#### Version 2.x

```cs
static void Main()
{
    config = new JobHostConfiguration();
    var filesConfig = new FilesConfiguration
    {
        RootPath = @"c:\data\import"
    };
    config.UseFiles(filesConfig);
    var host = new JobHost(config);
    host.RunAndBlock();
}
```

## Binding expressions

In attribute constructor parameters, you can use expressions that resolve to values from various sources. For example, in the following code, the path for the `BlobTrigger` attribute creates an expression named `filename`. When used for the output binding, `filename` resolves to the name of the triggering blob.

```cs
public static void CreateThumbnail(
    [BlobTrigger("sample-images/{filename}")] Stream image,
    [Blob("sample-images-sm/{filename}", FileAccess.Write)] Stream imageSmall,
    string filename,
    ILogger logger)
{
    logger.Info($"Blob trigger processing: {filename}");
    // ...
}
```

For more information about binding expressions, see [Binding expressions and patterns](../azure-functions/functions-bindings-expressions-patterns.md) in the Azure Functions documentation.

### Custom binding expressions

Sometimes you want to specify a queue name, a blob name or container, or a table name in code rather than hard-code it. For example, you might want to specify the queue name for the `QueueTrigger` attribute in a configuration file or environment variable.

You can do that by passing in a `NameResolver` object to the `JobHostConfiguration` object. You include placeholders in trigger or binding attribute constructor parameters, and your `NameResolver` code provides the actual values to be used in place of those placeholders. The placeholders are identified by surrounding them with percent (%) signs, as shown in the following example:

```cs
public static void WriteLog([QueueTrigger("%logqueue%")] string logMessage)
{
    Console.WriteLine(logMessage);
}
```

This code lets you use a queue named `logqueuetest` in the test environment and one named `logqueueprod` in production. Instead of a hard-coded queue name, you specify the name of an entry in the `appSettings` collection.

There is a default NameResolver that takes effect if you don't provide a custom one. The default gets values from app settings or environment variables.

Your `NameResolver` class gets the queue name from `appSettings` as shown in the following example:

```cs
public class CustomNameResolver : INameResolver
{
    public string Resolve(string name)
    {
        return ConfigurationManager.AppSettings[name].ToString();
    }
}
```

#### Version 3.x

The resolver is configured using dependency injection. These samples require the following `using` statement:

```cs
using Microsoft.Extensions.DependencyInjection;
```

The resolver is added by calling the [`ConfigureServices`] extension method on  [HostBuilder](/dotnet/api/microsoft.extensions.hosting.hostbuilder), as in the following example:

```cs
static async Task Main(string[] args)
{
    var builder = new HostBuilder();
    var resolver = new CustomNameResolver();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
    });
    builder.ConfigureServices(s => s.AddSingleton<INameResolver>(resolver));
    var host = builder.Build();
    using (host)
    {
        await host.RunAsync();
    }
}
```

#### Version 2.x

Pass your `NameResolver` class in to the `JobHost` object as shown in the following example:

```cs
 static void Main(string[] args)
{
    JobHostConfiguration config = new JobHostConfiguration();
    config.NameResolver = new CustomNameResolver();
    JobHost host = new JobHost(config);
    host.RunAndBlock();
}
```

Azure Functions implements `INameResolver` to get values from app settings, as shown in the example. When you use the WebJobs SDK directly, you can write a custom implementation that gets placeholder replacement values from whatever source you prefer. 

## Binding at runtime

If you need to do some work in your function before using a binding attribute such as `Queue`, `Blob`, or `Table`, you can use the `IBinder` interface.

The following example takes an input queue message and creates a new message with the same content in an output queue. The output queue name is set by code in the body of the function.

```cs
public static void CreateQueueMessage(
    [QueueTrigger("inputqueue")] string queueMessage,
    IBinder binder)
{
    string outputQueueName = "outputqueue" + DateTime.Now.Month.ToString();
    QueueAttribute queueAttribute = new QueueAttribute(outputQueueName);
    CloudQueue outputQueue = binder.Bind<CloudQueue>(queueAttribute);
    outputQueue.AddMessageAsync(new CloudQueueMessage(queueMessage));
}
```

For more information, see [Binding at runtime](../azure-functions/functions-dotnet-class-library.md#binding-at-runtime) in the Azure Functions documentation.

## Binding reference information

Reference information about each binding type is provided in the Azure Functions documentation. Using Storage queue as an example, you'll find the following information in each binding reference article:

* [Packages](../azure-functions/functions-bindings-storage-queue.md#packages---functions-1x) - What package to install in order to include support for the binding in a WebJobs SDK project.
* [Examples](../azure-functions/functions-bindings-storage-queue.md#trigger---example) - The C# class library example applies to the WebJobs SDK; just omit the `FunctionName` attribute.
* [Attributes](../azure-functions/functions-bindings-storage-queue.md#trigger---attributes) - The attributes to use for the binding type.
* [Configuration](../azure-functions/functions-bindings-storage-queue.md#trigger---configuration) - Explanations of the attribute properties and constructor parameters.
* [Usage](../azure-functions/functions-bindings-storage-queue.md#trigger---usage) - What types you can bind to and information about how the binding works. For example: polling algorithm, poison queue processing.
  
For a list of binding reference articles, see **Supported bindings** in the [Triggers and bindings](../azure-functions/functions-triggers-bindings.md#supported-bindings) article for Azure Functions. In that list, the HTTP, webhook, and Event Grid bindings are supported only by Azure Functions, not by the WebJobs SDK.

## Disable attribute 

The [Disable](https://github.com/Azure/azure-webjobs-sdk/blob/master/src/Microsoft.Azure.WebJobs/DisableAttribute.cs) attribute lets you control whether a function can be triggered. 

In the following example, if the app setting `Disable_TestJob` has a value of "1" or "True" (case insensitive), the function will not run. In that case, the runtime creates a log message *Function 'Functions.TestJob' is disabled*.

```cs
[Disable("Disable_TestJob")]
public static void TestJob([QueueTrigger("testqueue2")] string message)
{
    Console.WriteLine("Function with Disable attribute executed!");
}
```

When you change app setting values in the Azure portal, it causes the WebJob to be restarted, picking up the new setting.

The attribute can be declared at the parameter, method, or class level. The setting name can also contain binding expressions.

## Timeout attribute

The [Timeout](https://github.com/Azure/azure-webjobs-sdk/blob/master/src/Microsoft.Azure.WebJobs/TimeoutAttribute.cs) attribute causes a function to be canceled if it doesn't complete within a specified amount of time. In the following example, the function would run for one day without the timeout. With the timeout, the function is canceled after 15 seconds.

```cs
[Timeout("00:00:15")]
public static async Task TimeoutJob(
    [QueueTrigger("testqueue2")] string message,
    CancellationToken token,
    TextWriter log)
{
    await log.WriteLineAsync("Job starting");
    await Task.Delay(TimeSpan.FromDays(1), token);
    await log.WriteLineAsync("Job completed");
}
```

You can apply the Timeout attribute at class or method level, and you can specify a global timeout by using `JobHostConfiguration.FunctionTimeout`. Class or method level timeouts override the global timeout.

## Singleton attribute

Use the [Singleton](https://github.com/Azure/azure-webjobs-sdk/blob/master/src/Microsoft.Azure.WebJobs/SingletonAttribute.cs) attribute to ensure that only one instance of a function runs even when there are multiple instances of the host web app. It does this by implementing [distributed locking](#viewing-lease-blobs).

In the following example, only a single instance of the `ProcessImage` function runs at any given time:

```cs
[Singleton]
public static async Task ProcessImage([BlobTrigger("images")] Stream image)
{
     // Process the image
}
```

### SingletonMode.Listener

Some triggers have built-in support for concurrency management:

* **QueueTrigger** - Set `JobHostConfiguration.Queues.BatchSize` to 1.
* **ServiceBusTrigger** - Set `ServiceBusConfiguration.MessageOptions.MaxConcurrentCalls` to 1.
* **FileTrigger** - Set `FileProcessor.MaxDegreeOfParallelism` to 1.

You can use these settings to ensure that your function runs as a singleton on a single instance. To ensure only a single instance of the function is running when the web app scales out to multiple instances, apply a listener level Singleton lock on the function (`[Singleton(Mode = SingletonMode.Listener)]`). Listener locks are acquired on startup of the JobHost. If three scaled-out instances all start at the same time, only one of the instances acquires the lock and only one listener starts.

### Scope Values

You can specify a **scope expression/value** on the Singleton that ensures that all executions of the function at that scope will be serialized. Implementing more granular locking in this way can allow for some level of parallelism for your function, while serializing other invocations as dictated by your requirements. For example, in the following example the scope expression binds to the `Region` value of the incoming message. When the queue contains three messages in Regions "East", "East", and "West" respectively, then the messages that have region "East" are executed serially while the message with region "West" are executed in parallel with those in "East."

```csharp
[Singleton("{Region}")]
public static async Task ProcessWorkItem([QueueTrigger("workitems")] WorkItem workItem)
{
     // Process the work item
}

public class WorkItem
{
     public int ID { get; set; }
     public string Region { get; set; }
     public int Category { get; set; }
     public string Description { get; set; }
}
```

### SingletonScope.Host

The default scope for a lock is `SingletonScope.Function` meaning the lock scope (the blob lease path) is tied to the fully qualified function name. To lock across functions, specify `SingletonScope.Host` and use a scope ID name that is the same across all of the functions that you don't want to run simultaneously. In the following example, only one instance of `AddItem` or `RemoveItem` runs at a time:

```csharp
[Singleton("ItemsLock", SingletonScope.Host)]
public static void AddItem([QueueTrigger("add-item")] string message)
{
     // Perform the add operation
}

[Singleton("ItemsLock", SingletonScope.Host)]
public static void RemoveItem([QueueTrigger("remove-item")] string message)
{
     // Perform the remove operation
}
```

### Viewing lease blobs

The WebJobs SDK uses [Azure blob leases](../storage/common/storage-concurrency.md#pessimistic-concurrency-for-blobs) under the covers to implement distributed locking. The lease blobs used by Singleton can be found in the `azure-webjobs-host` container in the `AzureWebJobsStorage` storage account under path "locks". For example, the lease blob path for the first `ProcessImage` example shown earlier might be `locks/061851c758f04938a4426aa9ab3869c0/WebJobs.Functions.ProcessImage`. All paths include the JobHost ID, in this case 061851c758f04938a4426aa9ab3869c0.

## Async functions

For information about how to code async functions, see the Azure Functions documentation on [Async Functions](../azure-functions/functions-dotnet-class-library.md#async).

## Cancellation tokens

For information about how to handle cancellation tokens, see the Azure Functions documentation on [cancellation tokens and graceful shutdown](../azure-functions/functions-dotnet-class-library.md#cancellation-tokens).

## Multiple instances

If your web app runs on multiple instances, a continuous WebJob runs on each instance, listening for triggers and calling functions. The various trigger bindings are designed to efficiently share work collaboratively across instances, so that scaling out to more instances allows you to handle more load.

The queue and blob triggers automatically prevent a function from processing a queue message or blob more than once; functions do not have to be idempotent.

The timer trigger automatically ensures that only one instance of the timer runs, so you don't get more than one function instance running at a given scheduled time.

If you want to ensure that only one instance of a function runs even when there are multiple instances of the host web app, you can use the [Singleton attribute](#singleton-attribute).

## Filters

Function Filters (preview) provide a way to customize the WebJobs execution pipeline with your own logic. Filters are similar to [ASP.NET Core Filters](https://docs.microsoft.com/aspnet/core/mvc/controllers/filters). They can be implemented as declarative attributes that are applied to your functions or classes. For more information, see [Function Filters](https://github.com/Azure/azure-webjobs-sdk/wiki/Function-Filters).

## Logging and monitoring

We recommend the logging framework that was developed for ASP.NET, and the [Get started](webjobs-sdk-get-started.md) article shows how to use it. 

### Log filtering

Every log created by an `ILogger` instance has an associated `Category` and `Level`. [LogLevel](/dotnet/api/microsoft.extensions.logging.loglevel) is an enumeration, and the integer code indicates relative importance:

|LogLevel    |Code|
|------------|---|
|Trace       | 0 |
|Debug       | 1 |
|Information | 2 |
|Warning     | 3 |
|Error       | 4 |
|Critical    | 5 |
|None        | 6 |

Each category can be independently filtered to a particular [LogLevel](/dotnet/api/microsoft.extensions.logging.loglevel). For example, you might want to see all logs for blob trigger processing but only `Error` and higher for everything else.

#### Version 3.x

Version 3.x of the SDK relies on the filtering built into .NET Core. The `LogCategories` class lets you define categories for specific functions, triggers, or users. It also defines filters for specific host states, such as `Startup` and `Results`. In this way, you can fine-tune the logging output. If no match is found within the defined categories, the filter falls back to the `Default` value when deciding whether to filter the message.

`LogCategories` requires the following using statement:

```cs
using Microsoft.Azure.WebJobs.Logging; 
```

The following example constructs a filter that by default filters all logs at the `Warning` level. Categories of `Function` or `results` (equivalent to `Host.Results` in version 2.x) are filtered at the `Error` level. The filter compares the current category to all registered levels in the `LogCategories` instance and chooses the longest match. This means that the `Debug` level registered for `Host.Triggers` will match `Host.Triggers.Queue` or `Host.Triggers.Blob`. This allows you to control broader categories without needing to add each one.

```cs
static async Task Main(string[] args)
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
    });
    builder.ConfigureLogging(logging =>
            {
                logging.SetMinimumLevel(LogLevel.Warning);
                logging.AddFilter("Function", LogLevel.Error);
                logging.AddFilter(LogCategories.CreateFunctionCategory("MySpecificFunctionName"),
                    LogLevel.Debug);
                logging.AddFilter(LogCategories.Results, LogLevel.Error);
                logging.AddFilter("Host.Triggers", LogLevel.Debug);
            });
    var host = builder.Build();
    using (host)
    {
        await host.RunAsync();
    }
}
```

#### Version 2.x

In version 2.x of the SDK, the `LogCategoryFilter` class is used to control filtering. The `LogCategoryFilter` has a `Default` property with an initial value of `Information`, meaning that any messages with levels of `Information`, `Warning`, `Error`, or `Critical` are logged, but any messages with levels of `Debug` or `Trace` are filtered away.

As with `LogCategories` in version 23.x, the `CategoryLevels` property allows you to specify log levels for specific categories so you can fine-tune the logging output. If no match is found within the `CategoryLevels` dictionary, the filter falls back to the `Default` value when deciding whether to filter the message.

The following example constructs a filter that by default filters all logs at the `Warning` level. Categories of `Function` or `Host.Results` are filtered at the `Error` level. The `LogCategoryFilter` compares the current category to all registered `CategoryLevels` and chooses the longest match. This means that the `Debug` level registered for `Host.Triggers` will match `Host.Triggers.Queue` or `Host.Triggers.Blob`. This allows you to control broader categories without needing to add each one.

```csharp
var filter = new LogCategoryFilter();
filter.DefaultLevel = LogLevel.Warning;
filter.CategoryLevels[LogCategories.Function] = LogLevel.Error;
filter.CategoryLevels[LogCategories.Results] = LogLevel.Error;
filter.CategoryLevels["Host.Triggers"] = LogLevel.Debug;

config.LoggerFactory = new LoggerFactory()
    .AddApplicationInsights(instrumentationKey, filter.Filter)
    .AddConsole(filter.Filter);
```

### Custom telemetry for Application Insights

The way that you implement custom telemetry for [Application Insights](../azure-monitor/app/app-insights-overview.md) depends on the version of the SDK you are using. To learn how to configure Application Insights, see [Add Application Insights logging](webjobs-sdk-get-started.md#add-application-insights-logging).

#### Version 3.x

Since version 3.x of the WebJobs SDK relies on the .NET Core generic host, there's no longer a custom telemetry factory provided. However, you can add custom telemetry to the pipeline using dependency injection. The examples in this section require the following `using` statements:

```cs
using Microsoft.ApplicationInsights.Extensibility;
using Microsoft.ApplicationInsights.Channel;
```

The following custom implementation of [`ITelemetryInitializer`] lets you add your own [`ITelemetry`](/dotnet/api/microsoft.applicationinsights.channel.itelemetry) to the default [`TelemetryConfiguration`].

```cs
internal class CustomTelemetryInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        // Do something with telemetry.
    }
}
```

Call [`ConfigureServices`] in the builder to add your custom [`ITelemetryInitializer`] to the pipeline.

```cs
static void Main()
{
    var builder = new HostBuilder();
    builder.ConfigureWebJobs(b =>
    {
        b.AddAzureStorageCoreServices();
    });
    builder.ConfigureLogging((context, b) =>
    {
        // Add Logging Providers
        b.AddConsole();

        // If this key exists in any config, use it to enable App Insights
        string appInsightsKey = context.Configuration["APPINSIGHTS_INSTRUMENTATIONKEY"];
        if (!string.IsNullOrEmpty(appInsightsKey))
        {
            // This uses the options callback to explicitly set the instrumentation key.
            b.AddApplicationInsights(o => o.InstrumentationKey = appInsightsKey);
        }
    });
    builder.ConfigureServices(services =>
        {
            services.AddSingleton<ITelemetryInitializer, CustomTelemetryInitializer>();
        });
    var host = builder.Build();
    using (host)
    {

        host.Run();
    }
}
```

When the [`TelemetryConfiguration`] is constructed, all registered types of [`ITelemetryInitializer`] are included. To learn more about working the see [Application Insights API for custom events and metrics](../azure-monitor/app/api-custom-events-metrics.md).

In version 3.x, you no longer have to flush the [`TelemetryClient`] when the host stops. The .NET Core dependency injection system automatically disposes of the registered `ApplicationInsightsLoggerProvider`, which flushes the [`TelemetryClient`].

#### Version 2.x

In version 2.x, the [`TelemetryClient`] created internally by the Application Insights provider for the WebJobs SDK uses the [ServerTelemetryChannel](https://github.com/Microsoft/ApplicationInsights-dotnet/blob/develop/src/ServerTelemetryChannel/ServerTelemetryChannel.cs). When the Application Insights endpoint is unavailable or throttling incoming requests, this channel [saves requests in the web app's file system and resubmits them later](https://apmtips.com/blog/2015/09/03/more-telemetry-channels).

The [`TelemetryClient`] is created by a class that implements `ITelemetryClientFactory`. By default, this is the [`DefaultTelemetryClientFactory`](https://github.com/Azure/azure-webjobs-sdk/blob/dev/src/Microsoft.Azure.WebJobs.Logging.ApplicationInsights/DefaultTelemetryClientFactory.cs).

If you want to modify any part of the Application Insights pipeline, you can supply your own `ITelemetryClientFactory`, and the host will use your class to construct a [`TelemetryClient`]. For example, this code overrides the `DefaultTelemetryClientFactory` to modify a property of the `ServerTelemetryChannel`:

```csharp
private class CustomTelemetryClientFactory : DefaultTelemetryClientFactory
{
    public CustomTelemetryClientFactory(string instrumentationKey, Func<string, LogLevel, bool> filter)
        : base(instrumentationKey, new SamplingPercentageEstimatorSettings(), filter)
    {
    }

    protected override ITelemetryChannel CreateTelemetryChannel()
    {
        ServerTelemetryChannel channel = new ServerTelemetryChannel();

        // change the default from 30 seconds to 15 seconds
        channel.MaxTelemetryBufferDelay = TimeSpan.FromSeconds(15);

        return channel;
    }
}
```

The SamplingPercentageEstimatorSettings object configures [adaptive sampling](https://docs.microsoft.com/azure/application-insights/app-insights-sampling#adaptive-sampling-at-your-web-server). This means that in certain high-volume scenarios, App Insights sends a selected subset of telemetry data to the server.

Once you've created the telemetry factory, you pass it in to the Application Insights logging provider:

```csharp
var clientFactory = new CustomTelemetryClientFactory(instrumentationKey, filter.Filter);

config.LoggerFactory = new LoggerFactory()
    .AddApplicationInsights(clientFactory);
```

## <a id="nextsteps"></a> Next steps

This guide has provided code snippets that demonstrate how to handle common scenarios for working with the WebJobs SDK. For complete samples, see [azure-webjobs-sdk-samples](https://github.com/Azure/azure-webjobs-sdk-samples).

[`ExecutionContext`]: https://github.com/Azure/azure-webjobs-sdk-extensions/blob/v2.x/src/WebJobs.Extensions/Extensions/Core/ExecutionContext.cs
[`TelemetryClient`]: /dotnet/api/microsoft.applicationinsights.telemetryclient
[`ConfigureServices`]: /dotnet/api/microsoft.extensions.hosting.hostinghostbuilderextensions.configureservices
[`ITelemetryInitializer`]: /dotnet/api/microsoft.applicationinsights.extensibility.itelemetryinitializer
[`TelemetryConfiguration`]: /dotnet/api/microsoft.applicationinsights.extensibility.telemetryconfiguration
