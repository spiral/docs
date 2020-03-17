# Scaffolding
Most of the application code can be generated using the set of console commands.

## Installation
To install the extension:

```bash
$ composer require spiral/scaffolder
```

Make sure to add `Spiral\Scaffolder\Bootloader\ScaffolderBootloader` to your App class:

```php
use Spiral\Scaffolder\Bootloader\ScaffolderBootloader;

class App extends Kernel
{
    // ...

    protected const APP = [
        // ...
        ScaffolderBootloader::class
    ];
}
```

> Attention, the extension will invoke `TokenizerConfig`, make sure to add it at the end of the bootload chain.

## Configuration
You can customize the scaffolder component by replacing declaration generators and their options using the `scaffolder` configuration file.
The default configuration located in the `ScaffolderBootloader`.

## Available Commands
Command            | Description
---                | ---
create:bootloader  | Create Bootloader declaration
create:command     | Create Command declaration
create:config      | Create Config declaration
create:controller  | Create Controller declaration
create:filter      | Create HTTP Request Filter declaration
create:jobHandler  | Create Job Handler declaration
create:middleware  | Create Middleware declaration
create:migration   | Create Migration declaration
create:repository  | Create Entity Repository declaration
create:entity      | Create Entity declaration

### Bootloader
```bash
$ php app.php create:bootloader <name>
```

`<Name>Bootloader` class will be created.

#### Example
```bash
$ php app.php create:bootloader my
```

Output is:
```php
use Spiral\Boot\Bootloader\Bootloader;

class MyBootloader extends Bootloader
{
    protected const BINDINGS = [];

    protected const SINGLETONS = [];

    protected const DEPENDENCIES = [];

    public function boot(): void
    {
    }
}
```

### Command
```bash
$ php app.php create:command <name> [alias]
```

`<Name>Command` class will be created. Command name will be equal to `name` or `alias` (if this value set).

#### Example without `alias`
```bash
$ php app.php create:bootloader my
```

Output is:
```php
use Spiral\Console\Command;

class MyCommand extends Command
{
    protected const NAME = 'my';

    protected const DESCRIPTION = '';

    protected const ARGUMENTS = [];

    protected const OPTIONS = [];

    /**
     * Perform command
     */
    protected function perform(): void
    {
    }
}
```

#### Example with Alias
```bash
$ php app.php create:bootloader my alias
```

Output is:

```php
use Spiral\Console\Command;

class MyCommand extends Command
{
    protected const NAME = 'alias';

    protected const DESCRIPTION = '';

    protected const ARGUMENTS = [];

    protected const OPTIONS = [];

    /**
     * Perform command
     */
    protected function perform(): void
    {
    }
}
```

### Config
```bash
$ php app.php create:config <name>
```

`<Name>Config` class will be created. Also, `<app directory>/config/<name>.php` file will be created if doesn't exists.
Available options:

`reverse (r)` - Using this flag, scaffolder will look for `<app directory>/config/<name>.php` file and create a rich `<Name>Config` class based on the given config file.
The class will include default values and getters; in some cases, it will also include by-key-getters for array values.

Details below: 
If an array-value consists of more than one sub-values with the same types for keys and sub-values,
scaffolder will try to create a by-key-getter method. If a generated key is conflicting with an existing method, by-key-getter will be omitted.

#### Example with empty config file
```bash
$ php app.php create:config my
```

Output config file:

```php
return [];
```

Output config class:

```php
use Spiral\Core\InjectableConfig;

class MyConfig extends InjectableConfig
{
    public const CONFIG = 'my';

    /**
     * @internal For internal usage. Will be hydrated in the constructor.
     */
    protected $config = [];
}
```

#### Example with reversing
```php
//...existing "my.php" config file:
return [
    //will create "getParam()" by-key-getter (successfully singularized name)
    'params'      => [
        'one' => 'param',
        'two' => 'another param',
    ],
    //will create "getParameterBy()" by-key-getter (unsuccessfully singularized name)
    'parameter'   => [
        'one' => 'parameter',
        'two' => 'another parameter',
    ],
    //will create "getValueBy()" by-key-getter (because "getValue()" conflicts with the next "value" field)
    'values'      => [
        1 => 'value',
        2 => 'another value',
    ],
    'value'       => 'third value',
    //won't create by-key-getter due to only 1 sub-value
    'few'        => [
        'one' => 'value',
    ],
    //won't create by-key-getter due to mixed values' types
    'mixedValues' => [
        'one' => 'value',
        'two' => 2,
    ],
    //won't create by-key-getter due to mixed keys' types
    'mixedKeys'   => [
        'one' => 'value',
        2     => 'another value',
    ],
    //won't create by-key-getter to name conflicts
    //(because "getConflict()" and "getConflictBy()" conflicts with the next "conflict" and "conflictBy" fields)
    'conflicts'   => [
        'one' => 'conflict',
        'two' => 'another conflict',
    ],
    'conflict'    => 'third conflic',
    'conflictBy'  => 'fourth conflic',
];
```

```bash
$ php app.php create:config my -r
```

Output is:

```php
use Spiral\Core\InjectableConfig;

class MyConfig extends InjectableConfig
{
    public const CONFIG = 'my';

    /**
     * @internal For internal usage. Will be hydrated in the constructor.
     */
    protected $config = [
        'params'      => [],
        'parameter'   => [],
        'values'      => [],
        'value'       => '',
        'few'         => [],
        'mixedValues' => [],
        'mixedKeys'   => [],
        'conflicts'   => [],
        'conflict'    => '',
        'conflictBy'  => ''
    ];

    /**
     * @return array|string[]
     */
    public function getParams(): array
    {
        return $this->config['params'];
    }

    /**
     * @return array|string[]
     */
    public function getParameter(): array
    {
        return $this->config['parameter'];
    }

    /**
     * @return array|string[]
     */
    public function getValues(): array
    {
        return $this->config['values'];
    }

    /**
     * @return string
     */
    public function getValue(): string
    {
        return $this->config['value'];
    }

    /**
     * @return array|string[]
     */
    public function getFew(): array
    {
        return $this->config['few'];
    }

    /**
     * @return array
     */
    public function getMixedValues(): array
    {
        return $this->config['mixedValues'];
    }

    /**
     * @return array|string[]
     */
    public function getMixedKeys(): array
    {
        return $this->config['mixedKeys'];
    }

    /**
     * @return array|string[]
     */
    public function getConflicts(): array
    {
        return $this->config['conflicts'];
    }

    /**
     * @return string
     */
    public function getConflict(): string
    {
        return $this->config['conflict'];
    }

    /**
     * @return string
     */
    public function getConflictBy(): string
    {
        return $this->config['conflictBy'];
    }

    /**
     * @param string param
     * @return string
     */
    public function getParam(string $param): string
    {
        return $this->config['params'][$param];
    }

    /**
     * @param string parameter
     * @return string
     */
    public function getParameterBy(string $parameter): string
    {
        return $this->config['parameter'][$parameter];
    }

    /**
     * @param int value
     * @return string
     */
    public function getValueBy(int $value): string
    {
        return $this->config['values'][$value];
    }
}
```

### Controller
```bash
$ php app.php create:controller <name>
```

`<Name>Controller` class will be created. Available options:
* `action (a)` (multiple values allowed) - you can add actions using this option
* `prototype (p)` - if set, `PrototypeTrait` will be added

#### Example with empty actions list
```bash
$ php app.php create:controller my
```

Output is:

```php
class MyController
{
}
```

#### Example with `prototype` option
```bash
$ php app.php create:controller my -p
```

Output is:

```php
use Spiral\Prototype\Traits\PrototypeTrait;

class MyController
{
    use PrototypeTrait;
}
```

#### Example with actions list
```bash
$ php app.php create:controller my -a index -a create -a update -a delete
```

Output is:

```php
class MyController
{
    public function index()
    {
    }

    public function create()
    {
    }

    public function update()
    {
    }

    public function delete()
    {
    }
}
```

### HTTP Request Filter
```bash
$ php app.php create:filter <name>
```

`<Name>Filter` class will be created. Available options:
* `entity (e)` - you can pass an `EntityClass` and the filter command will fetch the all the given
class properties into the filter and try to define each property's type based on its type declaration (if php74),
default value or a PhpDoc. Otherwise you can optionally specify filter schema using `field` option.
* `field (f)` (multiple values allowed). 

Full field format is `name:type(source:origin)`. Where `type`, `origin` and `source:origin` are optional and can be omitted, defaults are:
  * type=string
  * source=data
  * origin=<name>
  
> See more about filters in [filters](https://github.com/spiral/filters) package

#### Example with empty fields definition
```bash
$ php app.php create:filter my
```

Output is:
```php
use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [];

    protected const VALIDATES = [];

    protected const SETTERS = [];
}
```

#### Example with fields definition:
```bash
$ php app.php create:filter my -f unknown_val -f str_val:string -f int_val:int -f bool_val:bool(query:from_bool) -f float_val:float(query)
```

Output is:
```php
use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'unknown_val' => 'data:unknown_val',
        'str_val'     => 'data:str_val',
        'int_val'     => 'data:int_val',
        'bool_val'    => 'query:from_bool',
        'float_val'   => 'query:float_val'
    ];

    protected const VALIDATES = [
        'unknown_val' => [
            'notEmpty',
            'string'
        ],
        'str_val'     => [
            'notEmpty',
            'string'
        ],
        'int_val'     => [
            'notEmpty',
            'integer'
        ],
        'bool_val'    => [
            'notEmpty',
            'boolean'
        ],
        'float_val'   => [
            'notEmpty',
            'float'
        ]
    ];

    protected const SETTERS = [];
}
```

#### Example with entity sourcing
```php
//...existing "MyEntity" class:
class MyEntity
{
    protected bool $typedBool;

    public $noTypeString;

    /** @var SomeOtherEntity */
    public $obj;

    /** @var int */
    protected $intFromPhpDoc;

    private $noTypeWithFloatDefault = 1.1;
}
```

```bash
$ php app.php create:filter my -e MyEntity
```

Output is:
```php
use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'typedBool'              => 'data:typedBool',
        'noTypeString'           => 'data:noTypeString',
        'obj'                    => 'data:obj',
        'intFromPhpDoc'          => 'data:intFromPhpDoc',
        'noTypeWithFloatDefault' => 'data:noTypeWithFloatDefault'
    ];

    protected const VALIDATES = [
        'typedBool'              => [
            'notEmpty',
            'boolean'
        ],
        'noTypeString'           => [
            'notEmpty',
            'string'
        ],
        'obj'                    => [
            'notEmpty',
            'string'
        ],
        'intFromPhpDoc'          => [
            'notEmpty',
            'integer'
        ],
        'noTypeWithFloatDefault' => [
            'notEmpty',
            'float'
        ]
    ];

    protected const SETTERS = [];
}
```

### Job Handler
```bash
$ php app.php create:jobHandler <name>
```

`<Name>Job` class will be created.

#### Example
```bash
$ php app.php create:jobHandler my
```

Output is:
```php
use Spiral\Jobs\JobHandler;

class MyJob extends JobHandler
{
    public function invoke(): void
    {
    }
}
```

### Middleware
```bash
$ php app.php create:middleware <name>
```

`<Name>` class will be created.

#### Example
```bash
$ php app.php create:middleware my
```

Output is:
```php
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;

class My implements MiddlewareInterface
{
    /**
     * {@inheritdoc}
     */
    public function process(Request $request, RequestHandlerInterface $handler): Response
    {
        return $handler->handle($request);
    }
}
```

### Migration
```bash
$ php app.php create:migration <name>
```

`<Name>Migration` class will be created. Available options:
* `table (t)` for table name
* `column (c)` (multiple values allowed) for column(s) definition. Will work only with `table` option. Column format is `name:type`.
> See more about migrations in [migrations](https://github.com/spiral/migrations) package

#### Example
```bash
$ php app.php create:migration my
```

Output is:
```php
use Spiral\Migrations\Migration;

class MyMigration extends Migration
{
    /**
     * Create tables, add columns or insert data here
     */
    public function up(): void
    {
    }

    /**
     * Drop created, columns and etc here
     */
    public function down(): void
    {
    }
}
```

#### Example with options
```bash
$ php app.php create:migration my -t my_table -c int_col:int
```

Output is:
```php
use Spiral\Migrations\Migration;

class MyMigration extends Migration
{
    /**
     * Create tables, add columns or insert data here
     */
    public function up(): void
    {
        $this->table('my_table')
            ->addColumn('int_col', 'int')
            ->create();
    }

    /**
     * Drop created, columns and etc here
     */
    public function down(): void
    {
        $this->table('my_table')->drop();
    }
}
```

### Repository
```bash
$ php app.php create:repository <name>
```

`<Name>Repository` class will be created.

#### Example
```bash
$ php app.php create:repository my
```

Output is:
```php
use Cycle\ORM\Select\Repository;

class MyRepository extends Repository
{
}
```

### ORM Entity
```bash
$ php app.php create:entity <name> [<format>]
```

`<Name>Entity` class will be created.
`format` is responsible for the declaration format. Currently, only [annotations](https://github.com/cycle/annotated) format supported. 
Available options:
* `role (r)` - Entity role, defaults to lowercase class name without a namespace
* `mapper (m)` - Mapper class name, defaults to Cycle\ORM\Mapper\Mapper
* `table (t)` - Entity source table, defaults to plural form of entity role
* `accessibility (a)` - accessibility accessor (public, protected, private), defaults to public
* `inflection (i)` - Optional column name inflection, allowed values: tableize (or t), camelize (or c). See [Doctrine inflector](https://github.com/doctrine/inflector)
* `field (f)` - Add field in a format "name:type" (multiple values allowed)
* `repository (e)` - Repository class to represent read operations for an entity, defaults to `Cycle\ORM\Select\Repository`
* `database (d)` - Database name, defaults to null (default database)

#### Example
```bash
$ php app.php create:entity my
```

Output is:
```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
}
```

#### Example with public accessibility
```bash
$ php app.php create:entity my -f field:string
```

Output is:
```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
    /**
     * @Cycle\Column(type = "string")
     */
    public $field;
}
```

#### Example with protected/private accessibility
```bash
$ php app.php create:entity my -f field:string -a protected
```

Output is:

```php

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
    /**
     * @Cycle\Column(type = "string")
     */
    protected $field;

    public function setField(string $value)
    {
        $this->field = $value;
    }

    public function getField()
    {
        return $this->field;
    }
}
```

#### Example with tableize inflection
```bash
$ php app.php create:entity my -f int_field:int -f stringField:string -i t
```

Output is:

```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
    /**
     * @Cycle\Column(type = "int")
     */
    public $int_field;

    /**
     * @Cycle\Column(type = "string", name = "string_field")
     */
    public $stringField;
}
```

#### Example with camelize inflection
```bash
$ php app.php create:entity my -f int_field:int -f stringField:string -i c
```

Output is:

```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
    /**
     * @Cycle\Column(type = "int", name = "intField")
     */
    public $int_field;

    /**
     * @Cycle\Column(type = "string")
     */
    public $stringField;
}
```

#### Example with other options
```bash
$ php app.php create:entity my -r myRole -m MyMapper -t my_table -d my_db -e
```

Output is:

```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity(role = "myRole", mapper = "MyMapper", repository = "my", table = "my_table", database = "my_db")
 */
class My
{
}
```

And the repository class is also created:

```php
use Cycle\ORM\Select\Repository;

class MyRepository extends Repository
{
}
```
