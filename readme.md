# Table of Contents [v2.*]

* [Documentation for v1.0](https://github.com/spiral/docs/tree/master)

* Getting Started
    * [About Spiral Framework](about/spiral.md)
    * [Versioning](about/semver.md)
    * [Installation](about/install.md)
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
    * Kernel, Environment
    * Bootloaders
    * Configuration
    * Config Objects
    * Container and Factories
    * IoC scopes
    * Auto Wiring
    * [Static Memory](framework/memory.md)
    * [Finalizers](framework/finalizers.md)
* Cookbook
    * Quick Start
    * Scaffolding
    * Prototyping
    * Singleton and Stateful Services
    * Database Scaffolding
    * Cycle ORM
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
    * [Configuration](console/configuration.md)
    * [User Commands](console/commands.md)
* Web and HTTP
    * Configuration
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
    * Configuration
    * Filter Entity
    * Nested Filters
    * Error Mapping
* Queues and Jobs
    * Configuration
    * Standalone Usage
    * Running Jobs
    * Scheduling Tasks
* GRPC API
    * Configuration
    * Generating Service Code
    * GPRC client code
* Views
    * Configuration
    * View object
    * Native PHP templates
    * Twig Templates
* Stempler Views
    * Configuration
    * Basic Usage
    * Inheritance
    * Components
    * Directives
* Internalization
    * Configuration
    * Indexation and Exporting
    * Translate Views
    * Say Trait
* DBAL
    * Configuration
    * Databases and Drivers
    * Query Builders
    * [Transactions](database/transactions.md)
    * [Schema Introspection](database/introspection.md)
    * [Schema Declaration](database/declaration.md)
    * Migrations
    * [Errata](database/errata.md)
* Cycle DataMapper ORM
    * [Configuration](cycle/configuration.md)
    * [Full Documentation](cycle/documentation.md)
    * [Console Commands](cycle/commands.md)
    * Extensions
* Storage Engine 
    * Configuration
    * Supported Drivers
* Debug and Profiling
    * [Dumping Variables](debug/dumps.md)
    * RoadRunner
    * Logging
    * Handle Exceptions
    * XDebug
* Advanced
    * RoadRunner customization
    * Controllers and HMVC
    * Auto-configuration
    * Performance optimizations
    * Docker and Kubernetes
    * Custom Dispatchers
* Extensions
    * DotEnv
    * PSR-6 cache
    * [Monolog](extension/monolog.md)
