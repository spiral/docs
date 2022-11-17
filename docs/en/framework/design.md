# Framework - Design Approach

The framework components attempt to follow the principle of pragmatic (KISS) design. Practically, it means that each component must follow the given rules:

- avoid cross dependencies when possible
- prioritize composition over inheritance
- prefer smaller but richer interfaces
- avoid magic

## Hybrid Runtime

The framework relies on the application server to run some of its services. PHP codebase is mostly centered around quick
delivery of efficient business logic. The application server, Golang based, is focused on efficiently solving infrastructure tasks.

> **Note**
> Spiral application server is a customized version of [RoadRunner](https://roadrunner.dev).

![High Level Architecture Diagram](https://user-images.githubusercontent.com/773481/180764832-d91daec4-36fb-4651-ace3-64eac6f289c8.png)

> **Note**
> Read about the application lifecycle [here](/start/workers.md).
