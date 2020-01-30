# Framework - Design Approach

## Architecture Principles

## Hybrid Runtime
The framework relies on application server to run some of its services. PHP codebase is mostly centered around quick delivery
of efficient business logic. The application server, Golang based, are focused on efficiently solving the infrastructure tasks.

> Spiral application server is customized version of [RoadRunner](https://roadrunner.dev).

![High Level Architecture Diagram](https://user-images.githubusercontent.com/796136/64451724-762d0800-d0ed-11e9-8c34-9c054a7bb0bd.png)

> Read about application lifecycle [here](/basic/workers.md).

## Code Quality and Standards
