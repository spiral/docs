# ODM Engine
Spiral ODM engine provides ability to define strict MongoDB schema with support of data inheritance, composition and aggregation.

## Table of Contents
* [MongoDB Databases](/old/odm/odm/databases.md)
* [Documents and DocumentEntity](/old/odm/odm/entities.md)
* [Accessors and Filters](/old/odm/odm/accessors.md)
* [Repositories and Selectors](/old/odm/odm/repositories.md)
* [Scaffolding](/old/odm/odm/scaffolding.md)
* [Compositions and Aggregations](/old/odm/odm/oop.md)
* [Inheritance](/old/odm/odm/inheritance.md)

## Schemas
Please note, ODM engine use second level cache to store information about mapping between persistence layer and your data entity models. Do not forget to run command `spiral odm:schema` to update schema cache.

## Standalone Usage
You are able to use ODM component separately from framework, take a look at initialization [here](https://github.com/spiral/odm/tree/master/tests/ODM).

> Component configuration and bindings are pre-installed in your application, require `spiral/odm` to enable mongodb support.