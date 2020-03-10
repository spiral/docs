# Table of Contents

* Overview
    * [About Framework](about/spiral.md)
    * [Contributing](about/contributing.md)
    * [Versioning](about/semver.md)
    * [LICENSE](license.md)
* Getting Started
    * [Installation](start/install.md)
    * [Application Lifecycle](start/workers.md)
    * [Directory Structure](start/structure.md)
    * [Configuration](start/configuration.md)
    * [Console Commands](start/commands.md)
* The Basics
    * [Quick Start](basics/quick-start.md)
    * [Scaffolding](basics/scaffolding.md)
    * [Prototyping](basics/prototype.md)
* Framework
    * [Design Approach](framework/design.md)
    * [Application Server](framework/application-server.md)
    * [Kernel and Environment](framework/kernel.md)
    * [Container and Factories](framework/container.md)
    * [Bootloaders](framework/bootloaders.md)
    * [Config Objects](framework/config.md)
    * [IoC scopes](framework/scopes.md)
    * [Static Memory](framework/memory.md)
    * [Finalizers](framework/finalizers.md)
* Cookbook
    * [Domain Core and Controllers](cookbook/domain-core.md)
    * [Container Injectors](cookbook/injector.md) 
    * [Integrate Golang service to PHP](cookbook/golang-library.md)
    * [Custom Dispatcher](cookbook/custom-dispatcher.md)
* Components
    * [Files and Directories](component/files.md)
    * [Code Generation](component/reactor.md)
    * [Static Analysis Tools](component/tokenizer.md)
    * [Prometheus Metrics](component/metrics.md)
    * [Data Grids](component/data-grid.md)
* Console
    * [Installation and Configuration](console/configuration.md)
    * [User Commands](console/commands.md)
* HTTP
    * [Installation and Configuration](http/configuration.md)
    * [Request Lifecycle](http/lifecycle.md)
    * [Request and Response](http/request-response.md)
    * [Routing](http/routing.md)
    * [Error Pages](http/errors.md)
    * [Middleware](http/middleware.md)
    * [Golang Middleware](http/golang.md)
    * [Cookies](http/cookies.md)
    * [Session](http/session.md)
    * [CSRF protection](http/csrf.md)
    * [Custom PSR-15 Handlers](http/psr-15.md)
* Security
    * [Data Encryption](security/encrypter.md)
    * [Validation Interfaces](security/validation.md)
    * [Role Based Access Control](security/rbac.md)
    * [User Authentication](security/authentication.md)
* Request Validation
    * [Installation and Configuration](filters/configuration.md)
    * [Filter Object](filters/filter.md)
    * [Composite Filters](filters/composite.md)
* Databases
    * [Installation and Configuration](database/configuration.md)
    * [Access Database](database/access.md)
    * [Database Isolation](database/isolation.md)
    * [Query Builders](database/query-builders.md)
    * [Transactions](database/transactions.md)
    * [Schema Introspection](database/introspection.md)
    * [Schema Declaration](database/declaration.md)
    * [Migrations](database/migrations.md)
    * [Errata](database/errata.md)
* Cycle ORM
    * [Installation and Configuration](cycle/configuration.md)
    * [Transactions](cycle/transactions.md)
    * [Without Annotations](cycle/manual.md)
    * [Full Documentation](cycle/documentation.md)
    * [Console Commands](cycle/commands.md)
* Queue and Jobs
    * [Installation and Configuration](queue/configuration.md)
    * [Console Commands](queue/commands.md)
    * [Running Jobs](queue/jobs.md)
    * [Standalone Usage](queue/standalone.md)
    * [Website Scraper](queue/scraper.md)
* Views
    * [Installation and Configuration](views/configuration.md)
    * [Rendering Views](views/render.md)
    * [Plain PHP templates](views/native.md)
    * [Twig templates](views/twig.md)
* Stempler Templating
    * [Installation and Configuration](stempler/configuration.md)
    * [Basic Usage](stempler/basics.md)
    * [Directives](stempler/directives.md)
    * [Inheritance and Stacks](stempler/inheritance.md)
    * [Components and Props](stempler/components.md)
    * [Advancing DSL](stempler/advanced.md)
    * [AST Modifications](stempler/visitors.md)
* Internalization
    * [Installation and Configuration](i18n/configuration.md)
    * [Import and Export](i18n/export.md)
    * [View Localization](i18n/views.md)
    * [Say Trait](i18n/say-trait.md)
* GRPC
    * [Installation and Configuration](grpc/configuration.md)
    * [Service Code](grpc/service.md)
    * [Client SDK](grpc/client.md)
    * [Golang Services](grpc/golang.md)
    * [Data Streaming](grpc/streaming.md)
* Event Broadcasting
    * [Installation and Configuration](broadcast/configuration.md)
    * [Publish and Consume](broadcast/publish.md)
    * [WebSockets](broadcast/websockets.md)
* Debug and Profiling
    * [Dumping Variables](debug/dumps.md)
    * [XDebug](debug/xdebug.md)
    * [Handling Exceptions](debug/exceptions.md)
* Extensions
    * [Code Style](extension/code-style.md)
    * [Dotenv](extension/dotenv.md)   
    * [Monolog](extension/monolog.md)
    * [Sentry](extension/sentry.md)
