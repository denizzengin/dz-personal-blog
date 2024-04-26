---
title: .Net 8 Keyed Services 🚀
date: 2024-04-25
modified: 2024-04-25
tags: [.NET, .NET8, KeyedService, Dependency Injection, Scoped, c#]
description: You’ll find the use case of `.NET Dependency Injection Keyed Service` which came with .NET 8.
image: "/net-keyed-service/keyed-service-output.png"
comments: true
---
This is my first personal blog post, which is why it's special for me. I decided to start with a good feature which has been coming recently. You’ll find the use case of `.NET Dependency Injection Keyed Service` which came with .NET 8.

We go over a use case and it would be good to explain it first,

Let's assume that we have a system that consists of multiple modular applications. They need to be sync some data and we called them as `Entity`.
It does not important to know the history of requirement just imagine scenario. We need to listen `Student` and `Teacher` entities for now.

We are the consumer/listener side and source side is publishing events as a general structure like below,

{% highlight ruby %}
{
"eventName" : "Student",
"RecordId" : "AC6B075D-8369-4FF9-BC7F-08DC62C199A0",
"Content" : "{ "name" : "John", "studentNumber":1 }"  
}
{% endhighlight %}

In our scenario, somehow we got this information and saved into our persistent place, most likely it is database.
In that moment, we need to process these events and that's the place where we take into consideration to KeyedService feature.
We will create `Processor` per `Entity` and use them dynamically getting by `EventName` using `KeyedService` feature.

The first we go to create two project one for Application(classlibrary) and one for BackgroundWorker(worker),


```bash
$ mkdir KeyedService
$ cd KeyedService
$ mkdir src
$ dotnet sln new --name KeyedService # add new sln named KeyedService
$ dotnet new console --name KeyedService.Application # change output type to class library or when you create change type to `classlib`
$ dotnet sln add .\src\KeyedService.Application\KeyedService.Application.csproj
$ dotnet new worker --name KeyedService.BackgroundServices # add new background service project with worker template
$ dotnet sln add .\src\KeyedService.BackgroundServices\KeyedService.BackgroundServices.csproj
```

We now ready to go coding. We will start from `KeyedService.Application` because the core logic is here. Open solution with your favorite IDE
and add new class named `DataChangedEventDto` as below,

{% highlight c# %}
public class DataChangedEventDto
{
        // This field will used as a keyed service key.
        public required string EventName { get; set; }

        // We can check already processed or not.
        public Guid RecordId { get; set; }

        /// <summary>
        /// Json format of Student or Teacher
        /// </summary>
        public string? Content { get; set; }

    }

{% endhighlight %}

After that, we can create data-transfer-objects(aka DTO, Dto) to deserialize and map from `DataChangedEventDto.Content`.
I kept a minimum number of props as much as possible to make it easy to understand. They're fields that might a Student or Teacher have.

{% highlight c# %}
public class StudentDataSyncDto
{
    // required and cannot be null name of student
    public required string Name { get; set; }

    public uint StudentNumber { get; set; }

    public string UrgencyParentPhoneCall { get; set; } = string.Empty;

}

public class TeacherDataSyncDto
{
    public required string Name { get; set; }

    public string? Division { get; set; }

    public int RegistrationNumber { get; set; }
}
{% endhighlight %}

At this point, we are ready to create our processor structure because we have Source Dto and target Dtos so we can start to set up how we handle them.
It takes general event dto and `CancellationToken` to cancel or stop the process of `async` operation.

{% highlight c# %}
public interface IDataSyncProcessor
{
    Task<bool> RunAsync(DataChangedEventDto changedEventDto, CancellationToken cancellationToken);
}
{% endhighlight %}

We create concrete processors for `Student` and `Teacher` as below,

{% highlight c# %}
public class StudentDataSyncProcessor(ILogger<StudentDataSyncProcessor> logger)
    : IDataSyncProcessor
{
    public Task<bool> RunAsync(DataChangedEventDto changedEventDto, CancellationToken cancellationToken)
    {
        logger.LogInformation("{processorName} is processing {dtoName}", nameof(StudentDataSyncProcessor), nameof(StudentDataSyncDto));

        return Task.FromResult(true);
    }

}

public class TeacherDataSyncProcessor(ILogger<StudentDataSyncProcessor> logger) : IDataSyncProcessor
{
    public Task<bool> RunAsync(DataChangedEventDto changedEventDto, CancellationToken cancellationToken)
    {
        // Do Teacher entity specific creation and saving db via dbContext etc.
        logger.LogInformation("{processorName} is processing {dtoName}", nameof(TeacherDataSyncProcessor), nameof(TeacherDataSyncDto));

        return Task.FromResult(true);
    }

}
{% endhighlight %}

These two processor classes are making only imaginary things for test purposes but they could be advanced scenarios in a real worl scenarios.
So they are just logging some context to show us `hey, I'm here` nothing more.

<strong> How do we use or call them from the BackgroundServices project ? </strong>

In order to make that, we can create a dependency injection file.
They are injected in extension methods. As you see they are added as `KeyedScoped` and set `serviceKey` which required parameter to able to call that.

<strong>Note: </strong> The reason for using `Scoped` service lifetime is that we will use it in the background job and don't want to use `Singleton` and `Transient`.(For more details visit [here][here])

{% highlight c# %}

public static class DependencyInjection
{
    public static IServiceCollection AddDataSyncProcessors(this IServiceCollection services)
    {
        // it would be good to use nameof(EntityName) structure instead of string
        // but we have no entity object because of scope of this blog post goes to show keyed services
        services.AddKeyedScoped<IDataSyncProcessor, StudentDataSyncProcessor>("Student");
        services.AddKeyedScoped<IDataSyncProcessor, TeacherDataSyncProcessor>("Teacher");

        return services;
    }

}

{% endhighlight %}

<strong>Note: </strong> Note: Microsoft internal implementation adding keyed service does not prevent or throw exceptions for duplicated keys. It will take the latest one and override another.
Therefore, you should be careful when giving them `serviceKey`.

Below usage is valid,

```
services.AddKeyedScoped<IDataSyncProcessor, StudentDataSyncProcessor>("Student");
services.AddKeyedScoped<IDataSyncProcessor, TeacherDataSyncProcessor>("Teacher");
services.AddKeyedScoped<IDataSyncProcessor, TeacherDataSyncProcessor>("Student");
```

<strong>Note: </strong>  Before going further you should ensure that the below packages installed to `KeyedService.Application` project. Otherwise, you get errors when compiling.

```
- Microsoft.Extensions.DependencyInjection
- Microsoft.Extensions.Logging
```

If it's not, you can install them using manage packager in Visual Studio or adding them using as defined,

```
$ dotnet add package ....
```

<strong> Note: </strong> From this point, all changes will be in KeyedService.BackgroundServices project.

The last part is calling `Keyed Services` from Background Service. Go and create new a class named `DataSyncWorker` which works as a background job and processes some sample events.

The below code, we added `IHostedService.StartAsync` and `BackgroundService.ExecuteAsync` methods. 
In the below code, we added the `IHostedService.StartAsync` and `BackgroundService.ExecuteAsync` methods. We need only `ExecuteAsync` method but in our case, we are creating fake data so we need it. `ExecuteAsync` method is empty for now just added some recurring mechanisms.

{% highlight c# %}

public class DataSyncWorker(ILogger<DataSyncWorker> logger, IServiceScopeFactory serviceScopeFactory) : BackgroundService
{
    private readonly ILogger<DataSyncWorker> _logger = logger;
    private List<DataChangedEventDto> dataChangedEvents = [];

    public override Task StartAsync(CancellationToken cancellationToken)
    {
        dataChangedEvents = [
            new() {
                EventName = "Student",
                RecordId = Guid.NewGuid(),
                Content = JsonSerializer.Serialize(new StudentDataSyncDto { Name = "John", StudentNumber = 1})
            },
            new() {
                EventName = "Teacher",
                RecordId = Guid.NewGuid(),
                Content = JsonSerializer.Serialize(new TeacherDataSyncDto { Name = "John", RegistrationNumber = 2, Division = "Science"})
            }
       ];

        _logger.LogInformation("{serviceName} service started to work...", nameof(DataSyncWorker));
        return base.StartAsync(cancellationToken);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            //todo

            await Task.Delay(4000, stoppingToken); // wait 4 seconds to re-work
        }
    }
}
{% endhighlight %}

We can implement the core logic of job now. We should create an `IServiceScopeFactory.CreateScope` to get keyed-service from background-job. Otherwise, its lifetime will be `Singleton`.Because background job itself is registered as `Singleton`.Then, we are getting related processor by iterating and calling `GetRequiredKeyedService` method with `serviceKey` parameter.

{% highlight c# %}
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        using var scope = serviceScopeFactory.CreateScope();

        foreach (var eventDto in dataChangedEvents)
        {
            var dataSyncProcessor = scope.ServiceProvider.GetRequiredKeyedService<IDataSyncProcessor>(eventDto.EventName);

            await dataSyncProcessor.RunAsync(eventDto, stoppingToken);
        }

        if (_logger.IsEnabled(LogLevel.Information))
        {
            _logger.LogInformation("Worker running at: {time}", DateTimeOffset.Now);
        }

        await Task.Delay(4000, stoppingToken); // wait 4 seconds to re-work
    }
}
{% endhighlight %}

The main point is here how getting related service. We are getting that service from eventName. For instance, Student or Teacher. It's automatically resolved at runtime per eventName we are able to use it to process events. If we need another event processor just add it as a separate implementation, inject it and ready for use without any change the background job service. 

```
var dataSyncProcessor = scope.ServiceProvider.GetRequiredKeyedService<IDataSyncProcessor>(eventDto.EventName);
```

Before run and test it we need one last thing that add `DataSyncWorker` as hosted service and inject KeyedService.Application processors to use in that service in `Program.cs`.

<strong>Note: </strong> It could be seen a bit different if you use `Program.Main` style program but they are both same.

{% highlight c# %}
using KeyedService.BackgroundServices;
using KeyedService.Application;

var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddDataSyncProcessors();
builder.Services.AddHostedService<DataSyncWorker>();

var host = builder.Build();
host.Run();
{% endhighlight %}

You can click Debug button and will see console output as below,

<figure>
    <img src="{{ page.image }}" alt="net-keyed-service">
    <figcaption>Expected Console Output</figcaption>
</figure>

**In conclusion**, keyed services are good for some use cases and they work fine with **Open-Closed** principle.

### Sample Repository

- [.Net8 Keyed Service](https://github.com/denizzengin/blog-dotnet-keyed-service)

### Resources

- [Dependency injection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-8.0)
- [Scoped Service](https://learn.microsoft.com/en-us/dotnet/core/extensions/scoped-service)

[here]: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-8.0

