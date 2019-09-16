# Table of Contents [v2.*]

* [Documentation for v1.0](https://github.com/spiral/docs/tree/master)

* Getting Started
    * [About Spiral Framework](about/spiral.md)
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
    * Application Server
    * [Container and Factories](framework/container.md)
    * Kernel, Environment
    * [Bootloaders](framework/bootloaders.md)
    * Config Objects
    * IoC scopes
    * Auto Wiring
    * [Static Memory](framework/memory.md)
    * [Finalizers](framework/finalizers.md)
* Cookbook
    * Quick Start
    * Scaffolding
    * [Prototyping](cookbook/prototype.md)
    * Singleton and Stateful Services
    * Database Scaffolding
    * Simple CRUD
    * Queue and Tasks
    * Custom PSR-15 Handlers
    * Request Validation
* Components
    * Files and Directories
    * Code Generation
    * Data Encryption
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
    * Scheduling Tasks
* GRPC API
    * Installation and Configuration
    * Generating Service Code
    * GPRC client code
* Views
    * Installation and Configuration
    * View object
    * Native PHP templates
    * Twig Templates
* Stempler Views
    * Installation and Configuration
    * Basic Usage
    * Inheritance
    * Components
    * Directives
    * AST Modifications 
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
    * RoadRunner
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
