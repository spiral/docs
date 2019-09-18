# Table of Contents [v2.*]

* [Documentation for v1.0](https://github.com/spiral/docs/tree/master)

* Getting Started
    * [About Framework](about/spiral.md)
    * [Installation](about/install.md)
    * [Versioning](about/semver.md)
    * [Contributing](about/contributing.md)
    * [LICENSE](license.md)
* Basics
    * [Workers and Application Lifecycle](basic/workers.md)
    * [Application Structure](basic/structure.md)
    * [Default Configuration](basic/configuration.md)
    * [Console Commands](basic/commands.md)
* Framework
    * Design Approach
    * [Application Server](framework/application-server.md)
    * Kernel, Environment
    * [Container and Factories](framework/container.md)
    * [Bootloaders](framework/bootloaders.md)
    * Config Objects
    * IoC scopes
    * [Auto Wiring](framework/auto-wiring.md)
    * [Static Memory](framework/memory.md)
    * [Finalizers](framework/finalizers.md)
* Cookbook
    * Quick Start
    * Scaffolding
    * [Prototyping](cookbook/prototype.md)
    * IoC scopes and Singletons
    * Database Scaffolding
    * Simple REST CRUD
    * Queue and Jobs
    * [Custom PSR-15 Handlers](cookbook/psr-15.md)
    * Integrating Golang library
* Components
    * Files and Directories
    * Code Generation
    * [Data Encryption](component/encrypter.md)
    * Validation
    * Pagination
    * RBAC Authorization
    * Entity Models
    * [Static Analysis Tools](component/tokenizer.md)
    * [Prometheus Metrics](component/metrics.md)
* Console
    * [Installation and Configuration](console/configuration.md)
    * [User Commands](console/commands.md)
* Web and HTTP
    * Installation and Configuration
    * Request Lifecycle
    * Read User Input
    * Request and Response
    * Routing
    * Error Pages
    * [Middleware](http/middleware.md)
    * Golang Middleware
    * [Cookies](http/cookies.md)
    * Session
    * CSRF protection
* Request/Filter Objects
    * Installation and Configuration
    * Filter Entity
    * Nested Filters
    * Error Mapping
* Queues and Jobs
    * Installation and Configuration
    * Standalone Usage
    * Running Jobs
    * Golang Consumers
    * Scheduling Tasks
* GRPC API
    * Installation and Configuration
    * Generating Service Code
    * Passing Metadata and Errors
    * Golang Services
    * GPRC client code
* Views
    * Installation and Configuration
    * View object
    * [Plain PHP templates](views/native.md)
    * Twig templates
* Stempler templates
    * Installation and Configuration
    * Basic Usage
    * Inheritance
    * Components
    * Directives
    * [AST Modifications](stempler/visitors.md)
* Internalization
    * Installation and Configuration
    * Indexation and Exporting
    * Translate Views
    * Say Trait
* DBAL
    * Installation and Configuration
    * Databases and Drivers
    * Query Builders
    * [Transactions](database/transactions.md)
    * ! [Schema Introspection](database/introspection.md)
    * ! [Schema Declaration](database/declaration.md)
    * [Migrations](database/migrations.md)
    * [Errata](database/errata.md)
* Cycle DataMapper ORM
    * [Installation and Configuration](cycle/configuration.md)
    * [Transactions](cycle/transactions.md)
    * [Full Documentation](cycle/documentation.md)
    * [Console Commands](cycle/commands.md)
* Storage Engine 
    * Installation and Configuration
    * Supported Drivers
* Debug and Profiling
    * [Dumping Variables](debug/dumps.md)
    * RoadRunner Gotchas
    * Logging
    * Handle Exceptions
    * XDebug
* Advanced
    * Controllers and HMVC
    * Auto-Configuration
    * Performance Tuning
    * Docker and Kubernetes
    * Custom Dispatchers
* Extensions
    * [Dotenv](extension/dotenv.md)   
    * [Monolog](extension/monolog.md)
