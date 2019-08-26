# ORM
Spiral Framework includes an ORM engine to help you manage your persistence data.
ORM is capable of schema generation, multiple data carry mechanisms and support relation extensions.

## Table of Contents
* [Record and RecordEntity](/v1.0.0/orm/orm/entities.md)
* [Repositories and Selectors](/v1.0.0/orm/orm/repositories.md)
* [Accessors and Filters](/v1.0.0/orm/orm/accessors.md)
* [Column Objects](/v1.0.0/orm/orm/columns.md)
* [Scaffolding and Migrations](/v1.0.0/orm/orm/scaffolding.md)
* [Transactions](/v1.0.0/orm/orm/transactions.md)
* [Relations](/v1.0.0/orm/orm/relations.md)
* [Morphed Relations](/v1.0.0/orm/orm/morphed-relations.md)
* [Pre-compiled Relations](/v1.0.0/orm/orm/late-binding.md)
* [Query Models](/v1.0.0/orm/orm/query.md)
* [Eager loading](/v1.0.0/orm/orm/loading.md)
* [Recursive Relations](/v1.0.0/orm/orm/recursive-relations.md)
* [Hybrid Databases](/v1.0.0/orm/orm/odm-bridge.md)
* [Custom Relations](/v1.0.0/orm/orm/custom-relations.md)

## ActiveRecord vs DataMapper
Spiral ORM uses features specific to both ActiveRecord and DataMapper approach, though, at this moment there is no native support to map data to pure php objects (see `RecordInterface`). 

Consider using Doctrine or writing your own hydration using `RecordSelector->fetchData` method and existed relation loaders. 

## Schemas
Please note, ORM engine uses second level cache/schema to store information about mapping between persistence layer and your data entity models. Do not forget to run command `spiral orm:schema -m` to update the schema cache and generate migrations.

## Standalone Usage
You are able to use ORM component separately from framework, take a look at initialization [here](https://github.com/spiral/orm/tree/master/tests/ORM).
