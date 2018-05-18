title: Advanced dependency injection in ASP.NET Core
author: stevejgordon
description: Learn how ASP.NET Core implements dependency injection and how to use it.
manager: 
ms.author: 
ms.custom: 
ms.date: 05/18/2018
ms.prod: asp.net-core
ms.technology: aspnet
ms.topic: article
uid: fundamentals/dependency-injection/advanced
---
# Advanced dependency injection in ASP.NET Core

By [Steve Gordon](https://www.stevejgordon.co.uk)

The default extensions for the services container provided by ASP.NET Core provide additional overloads which can be used in more advanced situations.

## Registering multiple implementations

The service container supports the registration of multiple implementations for a service. These can then resolved via DI as an `IEnumerable<T>` which includes all registered implementations.

For example, in some cases it can be useful to register multiple concrete implementations for a given interface. This allows new implementations to be added, without the need to update dependent classes.

Take the following interface for example:

```csharp
public interface IDataEnricher
{
    bool CanEnrich(MessageData message);
    void Enrich(MessageData message);
}
```

This interface defines a data enricher which supports conditionally enriching a `MessageData` object. The structure of the `MessageData` in this example contains properties which may be set with data received from an external system.

It's possible to create multiple implementations of the `IDataEnricher` which perform different enriching activities. This allows each to have a single responsibility and be easily tested in isolation. There are two methods defined in this case.

`CanEnrich` takes a `MessageData` object and is expected to return a boolean indicating whether the enricher supports enrichment that `MessageData`.

`Enrich` takes a `MessageData` object and is expected to enrich its properties with some additional data based on the current values.

As an example here is one possible enricher:

```csharp
public class LocationEnricher : IDataEnricher
{
    private readonly ILocationLookupService _locationLookup;

    public LocationEnricher(ILocationLookupService locationLookup)
    {
        _locationLookup = locationLookup;
    }

    public bool CanEnrich(MessageData message)
    {
        return !string.IsNullOrEmpty(message.IpAddress) &&
            message.EventType == "FailedLogin";
    }

    public void Enrich(MessageData message)
    {
        message.City = _locationLookup.GetCityFromIp(message.IpAddress);
    }
}
```

This enricher is intended to process messages with an `EventType` of "FailedLogin". It will do this by enriching the message with the city of the user using the `IpAddress` from the message. The `CanEnrich` method inspects the incoming message and assesses if it meets the criteria for enrichment. In this case if the `EventType` is "FailedLogin" and there is an IpAddress present, it returns true. Th `Enrich` method performs the actual enrichment using a service to get the city using the `IpAddress` from the message.

An example of another enricher:

```csharp
public class DateEnricher : IDataEnricher
{
    public bool CanEnrich(MessageData message)
    {
        return message.DateTimeLocal != default(DateTime);
    }

    public void Enrich(MessageData message)
    {
        message.DayOfWeek = message.DateTimeLocal.DayOfWeek.ToString();
    }
}
```

In the `CanEnrich` method a check is made to ensure that a `DateTimeLocal` property is set to a non default value. The `Enrich` method uses the `DateTimeLocal` property, populating another property with the `DayOfWeek`.

These implementations can both be registered with the service collection against their common interface:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<IDataEnricher, LocationEnricher>();
    services.AddSingleton<IDataEnricher, DateEnricher>();

    services.AddSingleton<MessageDataProcessor>();
}
```

Once registered in this way, it's possible to request all implementations from the service provider by requesting an `IEnumerable<IDataEnricher>`.

```csharp
public class MessageDataProcessor
{
    private readonly IEnumerable<IDataEnricher> _enrichers;

    public MessageProcessor(IEnumerable<IDataEnricher> enrichers)
    {
        _enrichers = enrichers;
    }

    public void ProcessMessage(MessageData message)
    {
        var applicableEnrichers = _enrichers.Where(e => e.CanEnrich(message));

        foreach(var enricher in applicableEnrichers)
        {
            enricher.Enrich(message);
        }
    }
}
```

This class will receive all registered enrichers via constructor injection. These can be stored and enumerated over. In this example, a LINQ query is used to find all enrichers that can enrich the current method. This is determined by calling their `CanEnrich` method.

Each matching enricher is then called in turn, passing in the message to be enriched.

The advantage of this approach is that should need arise for a third enricher; it can be defined and registered with DI. With no changes to the `MessageDataProcessor` the new enricher will take effect and process messages which is can enrich.

## Registering generic types

TODO

## Registering implementation factory delegates

TODO