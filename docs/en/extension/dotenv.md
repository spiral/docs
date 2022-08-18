# Extensions - Dotenv

Default application skeleton includes the ability to read ENV values from `.env` file located in the root of the
project.

## Configuration

To install Dotenv extension in non-default application skeleton:

```bash
composer require spiral/dotenv-bridge
```

Add the following bootloader to your application:

```php
    protected const LOAD = [
        Spiral\DotEnv\Bootloader\DotenvBootloader::class,
        
        //...
    ]
```

> **Note**
> Make sure to add a bootloader at the top of the list to alter Env.

The values from the `.env` the file will be copied into your env and available via `Spiral\Boot\EnvironmentInterface`
or `env` function.

> **Note**
> Attention, this component overwrites system ENV values.

## Pre-Processing

Please remember that values in `.env` will be pre-processed, following changes will occur:

| Value   | PHP Value |
|---------|-----------|
| true    | true,     |
| (true)  | true,     |
| false   | false,    |
| (false) | false,    |
| null    | null,     |
| (null)  | null,     |
| empty   | ''        |

> **Note**
> The quotes around strings will be stripped automatically.

## Enabling overwriting ENV variables

By default, new values don't overwrite previously set ones. This behavior can be changed by setting the `overwrite` 
parameter to `true` in the `Spiral\Boot\Environment` class:

```php
use Spiral\Boot\Environment;

$app = App::create([...]);

$app->run(new Environment([
    'APP_ENV' => 'production'
], overwrite: true));
```
