# Installation and Requiments
Spiral Framework uses **Composer** to resolve dependencies and install required components. You can check how to install composer [here](https://getcomposer.org/download/).

## Server Requiments
Make sure that your server is configured with following PHP version and extensions:
* PHP 7.0+
* OpenSSL Extension
* MbString Extension
* Tokenizer Extension
* PDO Extension with desired database drivers
* MongoDB extension (optional)

## Installation
One of fastest ways to install a fresh spiral application is to use the composer command:

```
php composer.phar create-project spiral/application
```

Initial application installation will automatically configure framework and enviroment to ensure that all nesessary directories have the correct permissions and are available for application, also it will perform translator indexation and view compilation (you can register you own commands in configure sequence, see [**Console Dispatcher**](/console/commands.md)).

You can also configure your application at any moment, by simply executing:

```
./spiral configure
```

Make sure that `spiral` has right execute permissions (`chmod +x spiral`, also you can rename `spiral` and `spiral.cli` to any name you want).

> You can run command(s) with increaced verbosity to get more details.

## Enviroment settings
Default spiral application and framework core utilize [dotenv package](https://github.com/vlucas/phpdotenv) which provides you ability to keep your enviroment settings in `.env` file, thought you can replace default implementation to avoid un-nesessary filesystem requests, follow [this link](/application/environment.md) for more instructions.

> Enviroment file `.env` will be automatically created and pre-configured as moment of application installation.

## Configuration
Your application confuration files are located in `app/config` directory, you can alter it's values to mount additional extensions or change some of application workflow.

Every configuration configuration file are represented by a simple PHP script which has to return array as it's result:

```php
<?php
/**
 * EncrypterManager component configuration file. Attention, configs might include runtime code
 * which depended on environment values only.
 *
 * @see EncrypterConfig
 */

return [
    /*
     * Encryption key can be found in .env file. You can generate new encryption key via console
     * command "app:key".
     */
    'key'    => env('SPIRAL_KEY'),

    /*
     * Default encryption algorithm.
     */
    'cipher' => 'aes-256-cbc'
];
```

If you wish to make your configuration be dependend on enviroment settings, use function `env(key, default)`.

> Attention, your configuration files will be cached relatively to current enviroment, you should not inlude any code which does not depend of `env` values!

Check `ConfiguratorInterface` and `ConfigFactory` in order to implement additional config loading methods (i.e. yaml).