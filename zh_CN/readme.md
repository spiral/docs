# 目录

*  开始
    * [框架介绍](about/spiral.md)
    * [安装指引](about/install.md)
    * [版本说明](../about/semver.md)
    * [贡献指引](../about/contributing.md)
    * [许可协议](../license.md)
* Basics
    * [Application Workers](../basic/workers.md)
    * [Application Structure](../basic/structure.md)
    * [Default Configuration](../basic/configuration.md)
    * [Console Commands](../basic/commands.md)
* Framework
    * [Design Approach](../framework/design.md)
    * [Application Server](../framework/application-server.md)
    * Kernel, Environment
    * [Container](../framework/container.md)
    * [Bootloaders](../framework/bootloaders.md)
    * [Config Objects](../framework/config.md)
    * [IoC scopes](../framework/scopes.md)
    * [Static Memory](../framework/memory.md)
    * [Finalizers](../framework/finalizers.md)
* Cookbook
    * Quick Start
    * [Scaffolding](../cookbook/scaffolding.md)
    * [Prototyping](../cookbook/prototype.md)
    * [Injectors](../cookbook/injector.md)
    * [Domain Core and Controllers](../cookbook/domain-core.md)
    * [Website Scraper](../cookbook/scraper.md)
    * [Custom PSR-15 Handlers](../cookbook/psr-15.md)
    * [Integrate Golang service to PHP](../cookbook/golang-library.md)
    * [Custom Dispatcher](../cookbook/custom-dispatcher.md)
    * Testing Application
* Components
    * [Files and Directories](../component/files.md)
    * [Code Generation](../component/reactor.md)
    * [Pagination](../component/pagination.md)
    * [Static Analysis Tools](../component/tokenizer.md)
    * [Prometheus Metrics](../component/metrics.md)
    * [Data Grids](../component/data-grid.md)
* Security
    * [Data Encryption](../../security/encrypter.md)
    * [Validation](../../security/validation.md)
    * [Role Based Access Control](../../security/rbac.md)
    * [User Authentication](../../security/authentication.md)
* Console
    * [Installation and Configuration](../console/configuration.md)
    * [User Commands](../console/commands.md)
* HTTP
    * [Installation and Configuration](../../http/configuration.md)
    * [Request Lifecycle](../../http/lifecycle.md)
    * [Request and Response](../../http/request-response.md)
    * [Routing](../../http/routing.md)
    * [Error Pages](../../http/errors.md)
    * [Middleware](../../http/middleware.md)
    * [Golang Middleware](../../http/golang.md)
    * [Cookies](../../http/cookies.md)
    * [Session](../../http/session.md)
    * [CSRF protection](../../http/csrf.md)
* Filter / Request Form
    * [Installation and Configuration](../filters/configuration.md)
    * [Filter Object](../filters/filter.md)
    * [Composite Filters](../filters/composite.md)
* Queue and Jobs
    * [Installation and Configuration](../queue/configuration.md)
    * [Console Commands](../queue/commands.md)
    * [Running Jobs](../queue/jobs.md)
    * [Standalone Usage](../queue/standalone.md)
* Views
    * [Installation and Configuration](../views/configuration.md)
    * [Rendering Views](../views/render.md)
    * [Plain PHP templates](../views/native.md)
    * [Twig templates](../views/twig.md)
* Stempler templates
    * [Installation and Configuration](../stempler/configuration.md)
    * Basic Usage
    * Inheritance
    * Components and Props
    * Directives
    * [AST Modifications](../stempler/visitors.md)
* Internalization
    * Installation and Configuration
    * Import and Export
    * Translate Views
    * Say Trait
* Databases
    * [Installation and Configuration](../../database/configuration.md)
    * [Access Database](../database/access.md)
    * [Database Isolation](../database/isolation.md)
    * [Query Builders](../database/query-builders.md)
    * [Transactions](../database/transactions.md)
    * [Schema Introspection](../database/introspection.md)
    * [Schema Declaration](../database/declaration.md)
    * [Migrations](../database/migrations.md)
    * [Errata](../database/errata.md)
* Cycle ORM
    * [Installation and Configuration](../cycle/configuration.md)
    * [Transactions](../cycle/transactions.md)
    * [Full Documentation](../cycle/documentation.md)
    * [Console Commands](../cycle/commands.md)
* Event Broadcasting
    * Installation and Configuration
    * WebSocket Client
    * Standalone Usage
* GRPC
    * Installation and Configuration
    * Generating Service Code
    * Passing Metadata and Errors
    * Golang Services
    * GRPC client code
* Debug and Profiling
    * [Dumping Variables](../debug/dumps.md)
    * XDebug
    * [Handling Exceptions](../debug/exceptions.md)
* Extensions
    * [Code Style](../extension/code-style.md)
    * [Dotenv](../extension/dotenv.md)   
    * [Monolog](../extension/monolog.md)
    * [Sentry](../extension/sentry.md)
