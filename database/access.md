# Database - Access Database
Follow the configuration instructions [here](/database/overview.md).

## Access the Database
Once DBAL component is configured properly you can access your databases in controllers and services multiple ways:

```php
namespace App\Controller;

use Spiral\Database\DatabaseManager;

class HomeController 
{
    public function index(DatabaseManager $dbal)
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
}
```

### Method and Constructor Injections
DBAL component fully support [IoC injections](/framework/container.md) based on database name and their aliases:

```php
public function index(Database $database, Database $primary, Database $slave)
{
    //Database is an alias for "primary"
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

### Prototype
Access `Spiral\Database\DatabaseProviderInterface` and default database instance using `PrototypeTrait`:

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        dump($this->dbal);
        dump($this->db); // default db
    }
}
```

## Run Queries
To run database query use method `query`:

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

To execute update or delete statement use alternative method `execute`:

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

> Read how to use query builders [here](/database/query-builders.md).