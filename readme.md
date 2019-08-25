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
    * Configuration
    * [Console Commands](basic/commands.md)
* Framework
    * Design Approach
    * RoadRunner
    * Kernel, Environment
    * Bootloaders
    * Configuration
    * Container and Factories    
    * IoC scopes
    * Auto-wiring
    * Config Objects
    * [Static Memory](framework/memory.md)
    * [Finalizers](framework/finalizers.md)
* Cookbook
    * Quick Start
    * Scaffolding
    * Singleton Services
    * Database Scaffolding
    * Cycle ORM
    * Simple CRUD
    * Queue and Tasks
    * Request Validation
    * Stempler Views
* Components
    * Files and Directories
    * Code Generation
    * Data Encryption
    * Validation
    * Pagination
    * RBAC Authorization
    * Entity Models
    * Static Analysis Tools
    * Prometheus Metrics
* Console
    * [Configuration](console/configuration.md)
    * [User Commands](console/commands.md)
* Web and HTTP
    * Configuration
    * Request Lifecycle
    * Input, Request and Response
    * PSR-7 and PSR-15 
    * Routing
    * Middleware
    * [Cookies](http/cookies.md)
    * Session
    * CSRF protection
* Request Objects
    * Configuration
    * Filter Entity
    * Nested Filters
    * Error mapping
* Queues and Jobs
    * Configuration
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
* Internalization
    * Configuration
    * Indexation and Exporting
    * View localization
    * Say Trait
* Stempler Markup Engine
    * Configuration
    * TODO ?
    * Extensions
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
* Extensions
    * DotEnv
    * PSR-6 cache
    * Monolog
* Debug and Profiling
    * [Dumping variables](debug/dumps.md)
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
* Roadmap
    * Vault
