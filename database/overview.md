# Database Abstraction Layer (DBAL) - TO BE UPDATED
Spiral provides a simple but powerful way to manage connections and operations related to multiple database sources. The DBAL focuses mainly on unifying databases rather than trying to get 100% of the specific DBMS feature set. However, you can always use direct queries to bypass the spiral abstractions and speak directly to the database.
At this moment, Spiral DBAL supports MySQL, SQLite, PostgresSQL and SQLServer (Windows) databases.

## Principle of Work and Abstractions Overview
Before jumping into the details, let's review the list of features DBAL allows us to do:
* Abstractions for Database, Driver, Table, Table Schema, Column, Index and Foreign Key entities
* Ability to read Database/Table schemas including columns, indexes, foreign keys
* Ability to write Database/Table schemas using declarative way (including columns, foreign keys and indices syncing)
* Query builders for Select, Update, Delete and Insert queries with fluent syntax (you know, everyone loves it)
* Ability to cache select queries using [StoreInterface] (/components/cache.md)
* Set of generic communication interfaces (DatabaseInterface, TableInterface, Schema\ColumnInterface and etc)

To better understand component hierarchy let's describe the basic DBAL classes:

Class         | Description
---           | ---
Driver        | Responsible for specific set of DBMS functions and used by Databases to hide specific implementation functionality. Represents PDO connection.
QueryCompiler | Responsible for the conversion of any set of query parameters (where tokens, table names, etc) into sql to be sent to a specific Driver.
Database      | High level abstraction at the top of Driver. Multiple databases can use same driver but different table prefix. Databases are usually linked to real database or logical portion of database (filtered by prefix).
Table         | Represent table level abstraction with simplified access to SelectQuery associated with such table.
QueryBuilder  | [QueryBuilder classes] (builders.md) generate set of control tokens for query compilers, this is query level abstraction.
QueryResult   | Wraps at top of PDOStatements and provides the ability to iterate through results.

Also you can find a set of classes and interfaces used to describe and declare desired table schema. Check the schema [reading] (reading.md) and [writing] (syncing.md) classes.

## Configuring
The Spiral Database Component stores it's configuration in the `application/config/database.php` file by default. We can alter this file to specify any needed connections, databases and database aliases.

### Connection
Before creating our first database, we have to specify the database connection. The Connection is usually described using PDO DNS, the driver class related to a specific DBMS or some other options. Let's review examples of default connections to all 4 Spiral databases:

```php
'mysql'     => [
    'driver'     => Drivers\MySQL\MySQLDriver::class,
    'connection' => 'mysql:host=127.0.0.1;dbname=demo',
    'profiling'  => true,
    'username'   => 'root',
    'password'   => 'root',
    'options'    => []
],
'postgres'  => [
    'driver'     => Drivers\Postgres\PostgresDriver::class,
    'connection' => 'pgsql:host=127.0.0.1;dbname=spiral',
    'profiling'  => true,
    'username'   => 'postgres',
    'password'   => '',
    'options'    => []
],
'sqlite'    => [
    'driver'     => Drivers\Sqlite\SqliteDriver::class,
    'connection' => 'sqlite:spiral.db',
    'profiling'  => true,
    'username'   => 'sqlite',
    'password'   => '',
    'options'    => []
],
'sqlServer' => [
    'driver'     => Drivers\SqlServer\SqlServerDriver::class,
    'connection' => 'sqlsrv:Server=SPIRAL\SQLEXPRESS;Database=spiral',
    'profiling'  => true,
    'username'   => null,
    'password'   => null,
    'options'    => []
]
```

As you can see, all connections have a specified name and self explanatory set of options. Please note that "profiling" will enable query logging and benchmarking on connection level.

> Options key "options" (yep) contains PDO specific connection options.

### Databases
Once you have your connections set up, you can create a set of databases to be reading and writing data using such connection. Every database must have a unique name and be associated with one of the connections (you can associate multiple databases with one connection). Additionally, you can specify table prefix on the database level. This  will automatically modify the table names in the queries generated using query builders and lets you create a logical database using one physical source.

```php
'primary'     => [
    'connection'  => 'mysql',
    'tablePrefix' => 'primary_'
],
'secondary' => [
    'connection'  => 'postgres',
    'tablePrefix' => 'secondary_',
],
```

### Aliases
This area is responsible for specifying the default database name and the set of name aliases which can point to itself or to specific database name. Let's check out the example below:

```php
'default'     => 'default',
'aliases'     => [
    'default'  => 'primary',
    'database' => 'primary',
    'db'       => 'primary',
    'slave'    => 'secondary'
],
```

In the example we showed, we named the the default database "default", which is an alias for the database "primary". Aliases can be useful in situation where you might want to separate data locations/ids without creating different physical sources.

## Check connections
Once you're done with the DBAL configuration you can make sure that spiral can connect to your listed databases using the console command "db:list". This command will provide you list of databases, their prefixes, tables and table sizes. It could look like:

```
+------------+-----------+----------+------------+-----------+------------------+----------------+
| Name (ID): | Database: | Driver:  | Prefix:    | Status:   | Tables:          | Count Records: |
+------------+-----------+----------+------------+-----------+------------------+----------------+
| primary    | demo      | MySQL    | primary_   | connected | license_user_map | 0              |
|            |           |          |            |           | licenses         | 0              |
|            |           |          |            |           | profiles         | 3              |
|            |           |          |            |           | role_user_map    | 0              |
|            |           |          |            |           | roles            | 0              |
|            |           |          |            |           | users            | 5              |
+------------+-----------+----------+------------+-----------+------------------+----------------+
| secondary  | spiral    | Postgres | secondary_ | connected | licenses         | 0              |
|            |           |          |            |           | license_user_map | 0              |
|            |           |          |            |           | profiles         | 4              |
|            |           |          |            |           | users            | 4              |
|            |           |          |            |           | roles            | 0              |
|            |           |          |            |           | role_user_map    | 0              |
+------------+-----------+----------+------------+-----------+------------------+----------------+
```

## Access the Database
After we make sure that Spiral can talk to our databases, we can start working with our component. The first step will be to get an instance of Database. We can do this first by getting an instance of `DatabasesInterface` or `DatabaseManager` (provider). Database component can also be retrieved using the short binding "dbal" (avaiable in application services including controllers and commands).

```php
protected function indexAction(DatabaseManager $dbal)
{
    //Default database
    dump($dbal->db());

    //Using alias default which points to primary database
    dump($dbal->db('default'));

    //Secondary
    dump($dbal->db('slave'));

    //Short binding + database name
    dump($this->dbal->db('secondary'));
}
```

### Controllable Injections
You might remember the one specific feature of [Spiral IoC container] (/framework/container.md) - controllable injections. This feature lets us resolve the dependency in constructor and method injections based on a context parameter. The Database component supports this feature and uses the parameter name to resolve the database. This simplifies our code and makes it possible to write controller actions (or init methods, or constructors) such as:

```php
protected function indexAction(Database $database, Database $primary, Database $slave)
{
    //Database is an alias for "primary"
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

> As result we can add a touch of magic to our code but make it much more readable. As you can see, aliases play a big role here because you can use them to create different variable names based on context.
