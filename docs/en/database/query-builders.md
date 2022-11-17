# Database - Query Builders

You can read how to work with Database using manually written queries [here](/database/access.md).

The DBAL component includes a set of query builders used to unify the way of working with different databases
and simplify migration to different DBMS over the lifetime of the application.

## Before we start

To demonstrate query building abilities, let's first declare a sample table in our default database:

```php
namespace App\Controller;

use Cycle\Database\Database;

class HomeController
{
    public function index(Database $db): void
    {
        $schema = $db->table('test')->getSchema();

        $schema->primary('id');
        $schema->datetime('time_created');
        $schema->enum('status', ['active', 'disabled'])->defaultValue('active');
        $schema->string('name', 64);
        $schema->string('email');
        $schema->double('balance');
        $schema->save();
    }
}
```

> **Note**
> You can read more about declaring database schemas [here](https://cycle-orm.dev/docs/database-declaration/2.x/en).

> **Note**
> You can find Full documentation about using Query builder [here](https://cycle-orm.dev/docs/database-query-builders/2.x/en).
