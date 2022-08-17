# Scaffolding

Most of the application code can be generated using the set of console commands.

## Installation

To install the extension:

```bash
composer require spiral/scaffolder
```

> **Note**
> Please note that the spiral/framework >= 2.7 already includes this component.

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

> **Note**
> Attention, the extension will invoke `TokenizerConfig`, make sure to add it at the end of the bootload chain.

## Configuration

You can customize the scaffolder component by replacing declaration generators and their options using the `scaffolder`
configuration file. The default configuration located in
the [ScaffolderBootloader](https://github.com/spiral/scaffolder/blob/master/src/Bootloader/ScaffolderBootloader.php#L59)
.

### Adding custom declarations via ScaffolderBootloader

Some components can provide their own declarations to create elements using the Scaffolder. 
Such components can register their custom declarations with the `ScaffolderBootloader`:
```php
namespace App\Bootloader;

use Spiral\Scaffolder\Bootloader\ScaffolderBootloader as BaseScaffolderBootloader;

class ScaffolderBootloader extends Bootloader
{
    public const DEPENDENCIES = [
        BaseScaffolderBootloader::class
    ];

    public function boot(BaseScaffolderBootloader $scaffolder): void
    {
        $scaffolder->addDeclaration('declarationName', [
            'namespace' => 'Namespace',
            'postfix'   => '', // like a Repository, Controller, etc
            'class'     => MyDeclaration::class, // declaration class
            'options'   => [
                // some custom options
            ],
        ]);
    }
}
```

## Available Commands

| Command           | Description                    |
|-------------------|--------------------------------|
| create:bootloader | Create Bootloader declaration  |
| create:command    | Create Command declaration     |
| create:config     | Create Config declaration      |
| create:controller | Create Controller declaration  |
| create:jobHandler | Create Job Handler declaration |
| create:middleware | Create Middleware declaration  |

Some packages may provide their own Commands. For example, the `Cycle Bridge` package (if installed) provides Commands:

| Command            | Description                          
|--------------------|--------------------------------------|
| create:migration   | Create Migration declaration         |
| create:repository  | Create Entity Repository declaration | 
| create:entity      | Create Entity declaration            | 

> **Note**
> Read more about `Cycle Bridge` package and available Commands [here](https://spiral.dev/docs/packages-cycle-bridge).

### Bootloader

```bash
php app.php create:bootloader <name>
```

`<Name>Bootloader` class will be created.

#### Example

```bash
php app.php create:bootloader my
```

Output is:

```php
use Spiral\Boot\Bootloader\Bootloader;

class MyBootloader extends Bootloader
{
    protected const BINDINGS = [];
    protected const SINGLETONS = [];
    protected const DEPENDENCIES = [];

    public function init(): void
    {
    }

    public function boot(): void
    {
    }
}
```

### Command

```bash
php app.php create:command <name> [alias]
```

`<Name>Command` class will be created. Command name will be equal to `name` or `alias` (if this value set).

#### Example without `alias`

```bash
php app.php create:command my
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
    protected function perform(): int
    {
    }
}
```

#### Example with alias

```bash
php app.php create:command my alias
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
    protected function perform(): int
    {
    }
}
```

### Config

```bash
php app.php create:config <name>
```

`<Name>Config` class will be created. Also, `<app directory>/config/<name>.php` file will be created if it doesn't
exist.

Available options:

`reverse (r)` - Using this flag, scaffolder will look for `<app directory>/config/<name>.php` file and create a
rich `<Name>Config` class based on the given config file. The class will include default values and getters; in some
cases, it will also include by-key-getters for array values.

Details below:
If an array-value consists of more than one sub-values with the same types for keys and sub-values, scaffolder will try
to create a by-key-getter method. If a generated key is conflicting with an existing method, by-key-getter will be
omitted.

#### Example with empty config file

```bash
php app.php create:config my
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

    /** @internal For internal usage. Will be hydrated in the constructor. */
    protected array $config = [];
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
php app.php create:config my -r
```

Output is:

```php
use Spiral\Core\InjectableConfig;

class MyConfig extends InjectableConfig
{
    public const CONFIG = 'my';

    /** @internal For internal usage. Will be hydrated in the constructor. */
    protected array $config = [
        'params' => [],
        'parameter' => [],
        'values' => [],
        'value' => '',
        'few' => [],
        'mixedValues' => [],
        'mixedKeys' => [],
        'conflicts' => [],
        'conflict' => '',
        'conflictBy' => '',
    ];

    /** @return string[] */
    public function getParams(): array
    {
        return $this->config['params'];
    }

    /** @return string[] */
    public function getParameter(): array
    {
        return $this->config['parameter'];
    }

    /** @return string[] */
    public function getValues(): array
    {
        return $this->config['values'];
    }

    public function getValue(): string
    {
        return $this->config['value'];
    }

    /** @return string[] */
    public function getFew(): array
    {
        return $this->config['few'];
    }

    public function getMixedValues(): array
    {
        return $this->config['mixedValues'];
    }

    /** @return string[] */
    public function getMixedKeys(): array
    {
        return $this->config['mixedKeys'];
    }

    /** @return string[] */
    public function getConflicts(): array
    {
        return $this->config['conflicts'];
    }

    public function getConflict(): string
    {
        return $this->config['conflict'];
    }

    public function getConflictBy(): string
    {
        return $this->config['conflictBy'];
    }

    public function getParam(string $param): string
    {
        return $this->config['params'][$param];
    }

    public function getParameterBy(string $parameter): string
    {
        return $this->config['parameter'][$parameter];
    }

    public function getValueBy(int $value): string
    {
        return $this->config['values'][$value];
    }
}
```

### Controller

```bash
php app.php create:controller <name>
```

`<Name>Controller` class will be created. Available options:

* `action (a)` (multiple values allowed) - you can add actions using this option
* `prototype (p)` - if set, `PrototypeTrait` will be added

#### Example with empty actions list

```bash
php app.php create:controller my
```

Output is:

```php
class MyController
{
}
```

#### Example with `prototype` option

```bash
php app.php create:controller my -p
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
php app.php create:controller my \
      -a index \
      -a create \
      -a update \
      -a delete
```

Output is:

```php
use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

class MyController
{
    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function index(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function create(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function update(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function delete(): ResponseInterface
    {
    }
}
```

### Job Handler

```bash
php app.php create:jobHandler <name>
```

`<Name>Job` class will be created.

#### Example

```bash
php app.php create:jobHandler my
```

Output is:

```php
use Spiral\Queue\JobHandler;

class MyJob extends JobHandler
{
    public function invoke(): void
    {
    }
}
```

### Middleware

```bash
php app.php create:middleware <name>
```

`<Name>` class will be created.

#### Example

```bash
php app.php create:middleware my
```

Output is:

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class My implements MiddlewareInterface
{
    /**
     * {@inheritdoc}
     */
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler,
    ): ResponseInterface {
        return $handler->handle($request);
    }
}

```
