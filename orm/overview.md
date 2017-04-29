# ORM
Spiral Framework includes an ORM engine to help you manage your persistence data.
ORM is capable of schema generation, multiple data carry mechanisms and support relation extensions.

## Table of Contents
* [Record and RecordEntity](/orm/entities.md)
* [Repositories and Selectors](/orm/repositories.md)
* [Accessors and Filters](/orm/accessors.md)
* [Column Objects](/orm/columns.md)
* [Scaffolding and Migrations](/orm/scaffolding.md)
* [Transactions](/orm/transactions.md)
* [Relations](/orm/relations.md)
* [Morphed Relations](/orm/morphed-relations.md)
* [Pre-compiled Relations](/orm/late-binding.md)
* [Query Models](/orm/query.md)
* [Eager loading](/orm/loading.md)
* [Recursive Relations](/orm/recursive-relations.md)
* [Hybrid Databases](/orm/odm-bridge.md)
* [Custom Relations](/orm/custom-relations.md)

## ActiveRecord vs DataMapper
Spiral ORM uses features specific to both ActiveRecord and DataMapper approach, though, at this moment there is no native support to map data to pure php objects (see `RecordInterface`). 

Consider using Doctrine or writing your own hydration using `RecordSelector->fetchData` method and existed relation loaders. 

## Schemas
Please note, ORM engine uses second level cache/schema to store information about mapping between persistence layer and your data entity models. Do not forget to run command `spiral orm:schema -m` to update the schema cache and generate migrations.

## Standalone Usage
You are able to use ORM component separately from framework, take a look at initialization [here](https://github.com/spiral/orm/tree/master/tests/ORM).
