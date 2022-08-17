# Database - Access Database

Follow the configuration instructions [here](/database/configuration.md).

## Access the Database

Once DBAL component configured properly, you can access your databases in controllers and services multiple ways:

```php
namespace App\Controller;

use Cycle\Database\DatabaseManager;

class HomeController 
{
    public function index(DatabaseManager $dbal): void
    {
        //Default database
        dump($dbal->database());
        
        //Default database over shortcut
        dump($this->db);
    
        //Using alias default which points to primary database
        dump($dbal->database('default'));
    
        //Secondary
        dump($dbal->database('slave'));
    
        //Short binding + database name
        dump($this->dbal->database('secondary'));
    }
}
```

### Method and Constructor Injections

DBAL component fully support [IoC injections](/framework/container.md) based on database name and their aliases:

```php
public function index(Database $database, Database $primary, Database $slave): void
{
    //Database is an alias for "primary"
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

### Prototype

Access `Cycle\Database\DatabaseProviderInterface` and default database instance using `PrototypeTrait`:

```php
namespace App\Controller;

use Cycle\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): void
    {
        dump($this->dbal);
        dump($this->db); // default db
    }
}
```

## Run Queries

To run the database query use method `query`:

```php
dump(
    $db->query(
        'SELECT * FROM users WHERE id > ?',
        [
            1
        ]
    )->fetchAll()
);
```

To execute update or delete statement use the alternative method `execute`:

```php
dump(
    $db->execute(
        'DELETE FROM users WHERE id > ?',
        [
            1
        ]
    ) // number of affected rows 
);
```

> **Note**
> Read how to use query builders [here](https://cycle-orm.dev/docs/database-query-builders/2.x/en).
