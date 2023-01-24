# Overview — About Spiral Framework

## Meet Spiral Framework

The Spiral Framework is a robust and powerful framework built by [Spiral Scout](https://spiralscout.com/) RnD team,
designed for the development and maintenance of massive enterprise applications. It is built on strict architecture
design principles and utilizes components from Symfony. At its core, Spiral incorporates a classic MVC approach and
features a [routing system](../http/routing.md) similar to Laravel and Symfony. The framework also
supports [dependency injection](../framework/container.md) with autowiring and **real** singleton services.

> It is designed to be intuitive and easy to use, with a focus on developer experience similar to that of
> Laravel and Symfony.

## Performance

One of the key features of Spiral is its exceptional performance. The framework
utilizes [RoadRunner](../start/server.md), an application server specifically designed for PHP, to warm up code
only once at [bootload phase](../framework/lifecycle.md), and then communicates with it per individual request. This
approach results in a significant performance boost, providing **2x faster performance out of the box**, compared to
other PHP frameworks.

## Inspiration

Spiral also draws inspiration from frameworks like [Spring](https://spring.io/)
and [ASP.net](https://dotnet.microsoft.com/en-us/apps/aspnet), making it a versatile and feature-rich option for
developers looking to build high-performance and scalable applications. The framework also benefits from a strong and
active community, providing support and resources for developers to take advantage of its full capabilities.

## Aspect-Oriented Programming

Spiral utilizes the benefits
of [Aspect-Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming) (AOP) through the use of
[interceptors](../framework/interceptors.md). Interceptors are a key feature of AOP and allow developers to intercept
method calls and perform specific actions before, after, or around the method call. This can be useful for logging,
security, caching, and other cross-cutting concerns.

This approach allows developers to keep the business logic of their application separate from cross-cutting concerns,
making the code more maintainable and easier to understand. Additionally, it allows developers to reuse the same
interceptors across multiple methods or classes, without having to repeat the same code multiple times.

## Features

### Queue orchestration

Spiral utilizes all of the features of RoadRunner to provide a powerful and efficient way
to [process and consume](../queue/configuration.md) messages from multiple queue providers. The queue orchestration
happens inside the application server, while PHP is responsible for the low-level processing of messages, resulting in
a highly concurrent and scalable event bus. This allows you to build systems that can automatically adapt to different
load strategies, and even consume **up to 100k events per second** without much configuration hassle.

### Temporal workflows

For more complex and massive applications, Spiral provides an integration with Temporal workflow engine, which
eliminates the need to worry about cross-service communication, retries, timers and other concerns. This makes it easy
to build fault-tolerant subscription processes with trial periods, multiple email notifications and cancellation
policies.

### Cache brokers

In addition to the powerful queueing system, Spiral also provides an easy to use [caching layer](../basics/cache.md)
that works with a variety of brokers, and is [PSR-16](https://www.php-fig.org/psr/psr-16/) compatible. This caching
layer can also work without any broker, providing a flexible caching solution that can adapt to your specific needs.

### GRPC

Spiral framework provides an easy integration with [GRPC](../grpc/configuration.md), allowing developers to take
advantage of these performance benefits and data schema management in their application.

One of the key advantages of GRPC is its efficiency and low latency. Because it uses a binary protocol, it is much
faster than traditional REST-based APIs, which rely on text-based protocols like HTTP and JSON. Another important
feature of GRPC is its support for strict data schemas and automatic generation of DTOs (data transfer objects).

GRPC is language agnostic by default, so it allows you to call Spiral from other languages and vice-versa in an easy and
fast way. This can help you to build a more flexible and scalable microservices architecture.

### WebSockets

Spiral has an integration with [Centrifugo](../websockets/configuration.md) server, which is a high-performance,
open-source websocket server. It allows you to easily add real-time functionality to your application, such as push
notifications, live updates, and chat functionality.

The integration with Spiral allows you to easily add Centrifugo server to your application, and take advantage of its
functionality without having to worry about the underlying infrastructure.

Additionally, Centrifugo server provides a powerful and flexible pub-sub model, which allows you to easily send and
receive messages through RPC calls.

## Microservices

Spiral is not only a powerful framework for building enterprise-level applications, but it also provides a lot of tools
to simplify the design and maintenance of microservices. The framework can be easily packed into a single Docker
container and provides healthchecks that are compatible with Kubernetes, making it easy to deploy and manage in a
containerized environment.

> Additionally, the memory footprint of Spiral is very small, and it supports process forking, allowing you to 
> gracefully stop containers without losing payloads.

## Debugging

Spiral also provides official support of [Open Telemetry](../advanced/telemetry.md) which covers most of the framework's
components, making it easy to debug and understand what's going on in your application.

<hr>

Spiral framework is well-suited for engineers of all levels. If you have used Symfony or Laravel before, you'll find a
lot of similarities in Spiral and the rest of the infrastructure components are carefully integrated and easy to learn.

Overall, Spiral is a framework that makes building powerful and efficient enterprise applications a breeze. It focuses
on architecture correctness, provides a ton of instruments for vertical and horizontal scalability, is built on
well-known community libraries and PSR standards, works much faster than any similar class engine without memory leaks,
and makes it easy to build large-scale processes.

If you want to tackle complex engineering problems with ease and build high-performance and scalable applications,
Spiral is the framework for you.
