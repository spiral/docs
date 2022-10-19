# Working with Databases
Follow configuration instructions [here](/database/overview.md).

## Access the Database
Once DBAL component is configured properly you can access your databases in controllers and services multiple ways:

```php
protected function indexAction(DatabaseManager $dbal)
{
    //Default database
    dump($dbal->db());
    
    //Default database over shortcut
    dump($this->db);

    //Using alias default which points to primary database
    dump($dbal->db('default'));

    //Secondary
    dump($dbal->db('slave'));

    //Short binding + database name
    dump($this->dbal->db('secondary'));
}
```

### Controllable Injections
DBAL component fully support [controllable injections](/framework/container.md) based on database name and their aliases:

```php
protected function indexAction(Database $database, Database $primary, Database $slave)
{
    //Database is an alias for "primary"
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

### Create Databases in Runtime
If you wish to create database manually use `addDatabase` method of DatabaseManager component:

```php
$this->dbal->createDatabase('new', 'prefix_', 'mysql');
```

You can also push database using existed instance:

```php
$this->dbal->addDatabase(new Database(
    $this->dbal->driver('mysql'),
    'name',
    'prefix_'
));
```

## Create Connections in Runtime
Same way you can create one or more connections without need to alter configuration:

```php
$this->dbal->createDriver(
    'mysql-connection',
    MySQLDriver::class,
    'mysql:host=127.0.0.1;dbname=mydb',
    'username',
    'password'
);
```