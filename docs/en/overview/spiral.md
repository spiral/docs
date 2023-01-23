# Overview — About

## Meet Spiral Framework

Spiral Framework is a High-Performance PHP/Go Full-Stack framework and group of over sixty PSR-compatible components.
The Framework execution model is based on hybrid runtime where some services (GRPC, Queue, WebSockets, etc.) are handled by
Application Server [RoadRunner](https://github.com/spiral/roadrunner) and the PHP code of your application remains in
memory on a permanent basis (anti-memory leak tools included).

## Features

- Battle-tested since 2013
- [Lightning fast full-stack PHP framework](https://www.techempower.com/benchmarks/#section=data-r21&hw=ph&test=fortune&l=zik073-v&c=6)
- PSR-{2,3,4,6,7,11,15,16,17} compliant
- Powerful [RoadRunner](https://roadrunner.dev/) application server and resident memory application kernel
- Native support of queue (AMQP, Kafka, SQS, Beanstalk) and background PHP workers
- Native support of cache (PSR-16)
- GRPC server and client
- TCP server
- HTTPS, HTTP/2+Push, encrypted cookies, sessions, CSRF-guard
- The [ORM](https://github.com/cycle/orm) you will use for the next 25 years
- MySQL, MariaDB, SQLite, PostgreSQL, SQLServer support, auto-migrations
- Intuitive scaffolding and prototyping (it literally writes code for you)
- Helpful class discovery via static analysis
- Authentication, RBAC security, validation, and encryption
- Dynamic template engine to create your own HTML tags (or just use Twig)
- MVC, HMVC, CQRS, Queue-oriented, RPC-oriented, CLI apps... any apps
