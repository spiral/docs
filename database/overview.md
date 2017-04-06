# Database Abstraction Layer (DBAL)
Spiral provides a simple but powerful way to manage connections and operations related to multiple database sources.
The DBAL focuses mainly on unifying database access rather than trying to get 100% of the specific DBMS feature set. However, you can always use direct queries to bypass the spiral abstractions.

Spiral DBAL supports MySQL, SQLite, PostgresSQL and SQLServer (Windows) databases.

## Entities
Database component represent by set of entities:

* DatabaseManager - primary database factory and repository, this component can accessed using `dbal` shortcut.
* Database - common abstraction for your databases, might represent real database or prefix based partition.
* Driver - underlying API of Database, can work with multiple Database instances.
* Table - facade for a specific table and it's common functionality.
* AbstractTable - table schema introspection and declaration.
* QueryBuilder - set of fluent classes to compile SQL queries specific to each DBMS.

## Configure
The Spiral Database Component stores it's configuration in the `application/config/databases.php` file.
Configuration include set of options for each database driver, database-driver association and database aliases. 
 
### Connections
To create new database connection add new section or alter existed options of `connections` section of your configuration, you are able to use `env` function to keep your passwords and usernames separately. 

```php
'mysql'     => [
    'driver'     => Drivers\MySQL\MySQLDriver::class,
    'connection' => 'mysql:host=127.0.0.1;dbname=' . env('DB_NAME'),
    'profiling'  => env('DEBUG', false),
    'username'   => env('DB_USERNAME'),
    'password'   => env('DB_PASSWORD'),
    'options'    => []
],
'postgres'  => [
    'driver'     => Drivers\Postgres\PostgresDriver::class,
    'connection' => 'pgsql:host=127.0.0.1;dbname=' . env('DB_NAME'),
    'profiling'  => env('DEBUG', false),
    'username'   => env('DB_USERNAME'),
    'password'   => env('DB_PASSWORD'),
    'options'    => []
],
'runtime'   => [
    'driver'     => Drivers\SQLite\SQLiteDriver::class,
    'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
    'profiling'  => env('DEBUG', false),
    'username'   => 'sqlite',
    'password'   => '',
    'options'    => []
],
'sqlServer' => [
    'driver'     => Drivers\SQLServer\SQLServerDriver::class,
    'connection' => 'sqlsrv:Server=MY-PC;Database=' . env('DB_NAME'),
    'profiling'  => env('DEBUG', false),
    'username'   => env('DB_USERNAME'),
    'password'   => env('DB_PASSWORD'),
    'options'    => []
],
```

> Use connection option `options` to set PDO specific attributes.

### Databases
In order to access connected database we have to add it into `databases` section first:

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

> Prefix is optional and can be skipped.

### Aliases
You application and modules can access database multiple different ways. Database aliasing allows you to use separate databases with relation to one physical database: 

```php
'default'     => 'default',
'aliases'     => [
    'default'  => 'primary',
    'database' => 'primary',
    'db'       => 'primary',
    'slave'    => 'secondary'
],
```

Example:

```php
public function indexAction(Database $db, Database $slave)
{
    dump($this->db); //linked to "default" database => 'primary'

}
```

## Check Connection
Spiral framework bundle includes set of commands using to check your database connection and structure.
Run `spiral db:list` in order to view all configured databases:

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
