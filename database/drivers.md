#DBAL Drivers
List of drivers with integration notes included.

## MySQL Driver
MySQL driver if fully supported by DBAL layer with all functionality being available.

Additional features include ability to specify table engine at moment of creation, by default engine is set to `InnoDB`.

## Postgres Driver
Postgres driver intergration includes custom InsertQuery builder and modified QueryCompiler in order to retrieve last insert ID value based on auto-incremented primary key of target table. InsertQuery automaticall adds `RETURNING {primary key}` into generated SQL query.

## SQLite Driver
SQLite has full support of query builders but schema generation functionality work differently compared to other drivers.

