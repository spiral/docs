# Database and Data Entities
One of the notable part of any application is set of models dedicated to represent data stored in some database or source. At this moment spiral provides
two generic data entities you can use in your application - `ORM\Record` and `ODM\Document`. Both entities can be scaffolded using console toolkit and will
be located in 'application/classes/Database' directory.

Both ORM and ODM entities provides `ActiveRecord` like behaviour so you can use them directly in your code, hovewer i recommend to look at [Services] (services.md) for this purposes as it will allow you to make your code more modular.

## What is DataEntity
Generic purposes of any entity is to provide access to its data using set of getters, setters and accessors. In addition to that spiral count that every created entity
can and must be validated before save. You can read more about base DataEntity (which used for both ORM and ODM) [here] (/entities/overview.md).