# Database Abstraction Layer (DBAL)
Spiral provides simple but yet powerful way to manage connections and operations realated to multiple database sources. DBAL mainly focuses on unification between databases rather that trying to get 100% of DBMS specific feature set, however you can always use direct queries to bypass spiral abstractions and talk to database directly.
As this moment Spiral DBAL support MySQL, SQLite, PostgresSQL and SQLServer (Windows) databases.

## Principle of Work and Abstractions Overview
Before jumping to details let's review show list of features DBAL can provide us:
* Abstractions for Database, Driver, Table, Column, Index and Foreign Key entities
* Ability to read Database/Table schemas including columns, indexes, foreign keys
* Ability to write Database/Table schemas using declarative way (including columns, foreign keys and indexes syncing)
* Query builders for Select, Update, Delete and Insert queries with fluent syntax (we all love it)
* Ability to cache select queries using [StoreInterface] (/components/cache.md)
* Set of generic communication interfaces (DatabaseInterface, TableInterface, Schema\ColumnInterface and etc)
* Migrations mechanism

To better understand component iehahry let's try to describe it's basic DBAL classes:

Class         | Description
---           | ---
Driver        | Responsible for DBMS specific set of functions and used by Databases to hide implementation specific functionality. Represent PDO connection.
QueryCompiler | Responsible for conversion of set of query parameters (where tokens, table names and etc) into sql to be send into specific Driver.
Database      | High level abstraction at top of Driver. Multiple databases can use same driver but different by table prefix. Databases usually linked to real database or logical portion of database (filtered by prefix).
Table         | Represent table level abstraction with simplified access to SelectQuery associated with such table.
QueryBuilder  | [QueryBuilder classes] (builders.md) generate set of control tokens for query compilers, this is query level abstraction.
QueryResult   | Wraps at top of PDOStatements and provides ability to iterate though results.

Also you can find set of classes and interfaces used to describe and declare desired table schema, check schema [reading] (reading.md) and writing (syncing.md) classes.

## Configuring
Spiral Database Component store it's configuration in `application/config/database.php` file by default. We can alter such file to specify needed connections, databases and database aliases.

### Connection
Before creating our first database we have to specify database connection. Connection usually described using PDO DNS, driver class related to specific DBMS and other options. Let's try to review examples of default connections to all 4 spiral databases:

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

As you can see all connections has specified name and self explanatory set of options. Please note that "profiling" will enable query logging and benchmarking on connection level.

> Options key "options" (yep) contains PDO specific connection options.

### Databases
Once you have your connections set up, you can create set of databases to be reading and writing data using such connection, every database must have unique name and be associated with one of connections (you can associate multiple databases with one connection). Additionally you can specify table prefix on database level, such thing will automatically modify table names in queries generated using query builders and provide you ability to create logical databases using one physical source.

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
Another part of DBAL config may look confising, but it will have much more sence once we will jump to examples. Such part are responsible for specifying default database name and set of name aliases which can point to itself or to specific database name, let's try to view an example:

```php
'default'     => 'default',
'aliases'     => [
    'default'  => 'primary',
    'database' => 'primary',
    'db'       => 'primary',
    'slave'    => 'secondary'
],
```

In given example we selected default database under name "default", which is an alias for database "primary". Aliases can be useful in situation when you might want to separate data locations/ids without creating different physical sources.

## Check connections
Once you done with DBAL configuration you can check if spiral can connect to listed databases using console command "db:list", such command will provide you list of databases, their prefixes, tables and table sizes. It may looks like:

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
After we made sure that spiral can talk to our databases we can start working with our component, first step will be to receive an instance of Database. We can achieve this goal by getting instance of `DatabasesInterface` or `DatabaseManager` (provider) first. Database component can also be retrived using short binding "dbal" (avaiable in application services including controllers and commands).

```php
public function index(DatabaseManager $dbal)
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
You might remember one specific feature of [Spiral IoC container] (/framework/container.md) - controllable injections. Such feature provides ability to resolve dependency in constructor and method injections based on context parameter, Database component support such feature and uses parameter name to resolve database. Such ability can simplify our code a lot and makes possible to write controller actions like:

```php
public function index(Database $database, Database $primary, Database $slave)
{
    //Database is an alias for "primary"
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

> As result we can add little bit of magic to our code but make it much more readable. As you can see aliases playing a big role here due you can use then to create different variable names based on context.
