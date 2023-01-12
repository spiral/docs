# Cycle ORM - Installation and Configuration

> **Note**
> Cycle ORM is included into default [Web skeleton](https://github.com/spiral/app).
>

## Installation

To install the component in alternative bundles or as a standalone library:

```bash
composer require spiral/cycle-bridge
```

Activate the bootloader `Spiral\Cycle\Bootloader\BridgeBootloader` in your application:

```php app/src/Application/Kernel.php
use Spiral\Cycle\Bootloader as CycleBridge;

protected const LOAD = [
    // ...
    CycleBridge\BridgeBootloader::class,
    // ...
];
```

You can exclude `Spiral\Cycle\Bootloader\BridgeBootloader` bootloader and select only the necessary bootloaders by writing them
separately.

```php
use Spiral\Cycle\Bootloader as CycleBridge;

protected const LOAD = [
    // ...

    // Database
    CycleBridge\DatabaseBootloader::class,
    CycleBridge\MigrationsBootloader::class,

    // Close the database connection after every request automatically (Optional)
    // CycleBridge\DisconnectsBootloader::class,

    // ORM
    CycleBridge\SchemaBootloader::class,
    CycleBridge\CycleOrmBootloader::class,
    CycleBridge\AnnotatedBootloader::class,
    CycleBridge\CommandBootloader::class,

    // Validation (Optional)
    CycleBridge\ValidationBootloader::class,

    // DataGrid (Optional)
    CycleBridge\DataGridBootloader::class,

    // Database Token Storage (Optional)
    CycleBridge\AuthTokensBootloader::class,
    
    // Migrations and Cycle Scaffolders (Optional)
    CycleBridge\ScaffolderBootloader::class,
];
```

### Migration from Spiral Framework v2.8

If you are migrating from Spiral Framework 2.8, you have to get rid of old packages in composer.json:

```json
"require": {
    "spiral/database": "^2.3",
    "spiral/migrations": "^2.0",
    "cycle/orm": "^1.0",
    "cycle/proxy-factory": "^1.0",
    "cycle/annotated": "^2.0",
    "cycle/migrations": "^1.0",
    ...
},
```

Then you need to replace some of bootloaders with the ones that are provided by the package.

```php
use Spiral\Cycle\Bootloader as CycleBridge;

protected const LOAD = [
    // ...

    // Databases
    // OLD
    // Framework\Database\DatabaseBootloader::class,
    // Framework\Database\MigrationsBootloader::class,
    // Close the database connection after every request automatically (Optional)
    // Framework\Database\DisconnectsBootloader::class,

    // NEW
    CycleBridge\DatabaseBootloader::class,
    CycleBridge\MigrationsBootloader::class,
    CycleBridge\DisconnectsBootloader::class,

    // ORM
    // OLD
    // Framework\Cycle\CycleBootloader::class,
    // Framework\Cycle\ProxiesBootloader::class,
    // Framework\Cycle\AnnotatedBootloader::class,

    // NEW
    CycleBridge\SchemaBootloader::class,
    CycleBridge\CycleOrmBootloader::class,
    CycleBridge\AnnotatedBootloader::class,

    // ...

    // DataGrid (Optional)
    // OLD
    // \Spiral\DataGrid\Bootloader\GridBootloader::class,

    // NEW
    CycleBridge\DataGridBootloader::class,

    // Database Token Storage (Optional)
    // OLD
    // Framework\Auth\TokenStorage\CycleTokensBootloader::class,

    // NEW
    CycleBridge\AuthTokensBootloader::class,

    // Framework commands
    // ...
    CycleBridge\CommandBootloader::class,
];
```

> **Note**
> `CycleBridge\CommandBootloader` should be added after `\Spiral\Bootloader\CommandBootloader`
> to replace Cycle ORM 1 commands with new ones

## Configuration

You can create a config file `app/config/cycle.php` if you want to configure Cycle ORM:

```php
use Cycle\ORM\SchemaInterface;

return [
    'schema' => [
        /**
         * true (Default) - Schema will be stored in a cache after compilation.
         * It won't be changed after entity modification. Use `php app.php cycle` to update schema.
         *
         * false - Schema won't be stored in a cache after compilation.
         * It will be automatically changed after entity modification. (Development mode)
         */
        'cache' => false,

        /**
         * The CycleORM provides the ability to manage default settings for
         * every schema with not defined segments
         */
        'defaults' => [
            SchemaInterface::MAPPER => \Cycle\ORM\Mapper\Mapper::class,
            SchemaInterface::REPOSITORY => \Cycle\ORM\Select\Repository::class,
            SchemaInterface::SCOPE => null,
            SchemaInterface::TYPECAST_HANDLER => [
                \Cycle\ORM\Parser\Typecast::class
            ],
        ],

        'collections' => [
            'default' => 'array',
            'factories' => [
                'array' => new \Cycle\ORM\Collection\ArrayCollectionFactory(),
                // 'doctrine' => new \Cycle\ORM\Collection\DoctrineCollectionFactory(),
                // 'illuminate' => new \Cycle\ORM\Collection\IlluminateCollectionFactory(),
            ],
        ],

        /**
         * Schema generators (Optional)
         * null (default) - Will be used schema generators defined in bootloaders
         */
        'generators' => null,

        // 'generators' => [
        //        \Cycle\Schema\Generator\ResetTables::class,
        //        \Cycle\Annotated\Embeddings::class,
        //        \Cycle\Annotated\Entities::class,
        //        \Cycle\Annotated\TableInheritance::class,
        //        \Cycle\Annotated\MergeColumns::class,
        //        \Cycle\Schema\Generator\GenerateRelations::class,
        //        \Cycle\Schema\Generator\GenerateModifiers::class,
        //        \Cycle\Schema\Generator\ValidateEntities::class,
        //        \Cycle\Schema\Generator\RenderTables::class,
        //        \Cycle\Schema\Generator\RenderRelations::class,
        //        \Cycle\Schema\Generator\RenderModifiers::class,
        //        \Cycle\Annotated\MergeIndexes::class,
        //        \Cycle\Schema\Generator\GenerateTypecast::class,
        // ],
    ],

    /**
     * Custom relation types for entities
     */
    'customRelations' => [
        // \Cycle\ORM\Relation::EMBEDDED => [
        //     \Cycle\ORM\Config\RelationConfig::LOADER => \Cycle\ORM\Select\Loader\EmbeddedLoader::class,
        //     \Cycle\ORM\Config\RelationConfig::RELATION => \Cycle\ORM\Relation\Embedded::class,
        // ]
    ],

    /**
     * Prepare all internal ORM services (mappers, repositories, typecasters...)
     */
    'warmup' => false,
];
```
