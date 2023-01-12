# Extensions - Dotenv

The default application skeleton includes the ability to read ENV values from the `.env` file located at the root of the
project.

## Configuration

To install Dotenv extension in a non-default application skeleton:

```bash
composer require spiral/dotenv-bridge
```

Add the following bootloader to your application:

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    \Spiral\DotEnv\Bootloader\DotenvBootloader::class,
    
    //...
]
```

> **Note**
> Make sure to add the bootloader to the top of the list to alter Env.

The values from the `.env` the file will be copied to your env and available via `Spiral\Boot\EnvironmentInterface`
or the `env` function.

> **Note**
>  This component overwrites system ENV values.

## Pre-Processing

Remember that the values in `.env` will be pre-processed, the following changes will take place:

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
