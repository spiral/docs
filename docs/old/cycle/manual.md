# Cycle - Use without Attributes

You can define ORM Schema manually, without need to use annotations.

## Bootloader

Create Bootloader to describe and bind your Schema to `Cycle\ORM\SchemaInterface`:

```php
namespace App\Bootloader;

use App\Database\User;
use Cycle\ORM\Mapper\Mapper;
use Cycle\ORM\Schema;
use Cycle\ORM\SchemaInterface;
use Spiral\Boot\Bootloader\Bootloader;

class ORMBootloader extends Bootloader
{
    protected const SINGLETONS = [
        SchemaInterface::class => [self::class, 'schema']
    ];

    private function schema(): SchemaInterface
    {
        return new Schema([
            User::class => [
                Schema::ROLE        => 'user',
                Schema::MAPPER      => Mapper::class,
                Schema::DATABASE    => 'default',
                Schema::TABLE       => 'user',
                Schema::PRIMARY_KEY => 'id',
                Schema::COLUMNS     => ['id', 'email', 'balance'],
                Schema::SCHEMA      => [],
                Schema::RELATIONS   => []
            ]
        ]);
    }
}
```

Bind the Bootloader instead of `Cycle\AnnotatedBootloader`:

```php
protected const LOAD = [
    // ...
    Framework\Cycle\CycleBootloader::class,
    Framework\Cycle\ProxiesBootloader::class,
    App\Bootloader\ORMBootloader::class,
    // ...
];
```

You are ready to go.

> You can delete the `cycle/annotated` extension.
