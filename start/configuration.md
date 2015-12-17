# Configuring your Application

**BADLY OUTDATED**

Most of spiral components and services require external configuration to define it's behaviour and settings. Configuration data usually supplied in array form using `Spiral\Core\ConfiguratorInterface` instance. 

Spiral Framework core implements such interface and uses application `config` directory to store configurations in separate php files, every loaded configuration file will be merged with environment specific setttings and stored in permanent cache.

## Enviroment specific configurations
If you want to change some section of configuration file based on active enviroment, simply create directory inside config folder named as your environment and put configuration file in it. Spiral will replace original configuration sections with sections provided by this file.

> Configuration data will be merged using array_merge function, meaning only top level of config will be replaced.

## Examples
Let's look into sample configuration file for `ViewManager` component (`config/views.php`):

```php
<?php
/**
 * Configuration of ViewManager component and view engines:
 * - compiled view cache state and location
 * - view namespaces associated with list of source directories
 * - list of view dependencies, used by default compiler to create unique cache name
 * - list of view engines associated with their extension, compiler and default view class
 * - configuration of default spiral compiler it's processors and other options
 */
return [
    'cache'        => [
        'enabled'   => true,
        'directory' => directory("cache") . '/views'
    ],
    'namespaces'   => [
        'default'  => [
            directory("application") . '/views'
        ]
    ],
    'dependencies' => [
    ],
    'engines'      => [
        'default' => [
            'extensions' => [
                'php'
            ],
            'compiler'   => 'Spiral\Views\Compiler',
            'view'       => 'Spiral\Views\View'
        ]
    ],
    'compiler'     => [
        'processors' => [
            'Spiral\Views\Processors\ExpressionsProcessor' => [],
            'Spiral\Views\Processors\TranslateProcessor'   => [],
            'Spiral\Views\Processors\TemplateProcessor'    => [],
            'Spiral\Views\Processors\EvaluateProcessor'    => [],
            'Spiral\Views\Processors\PrettifyProcessor'    => []
        ]
    ]
];
```

If we wish to disable caching in development enviroment we can create file `config/development/views.php` with content like that:

```php
<?php
/**
 * Disables view cache in development enviroment.
 */
return [
    'cache'        => [
        'enabled'   => false,
        'directory' => directory("cache") . '/views'
    ]
];
```

Configuration section `cache` will be replaced with enviroment specific data.

> Due enviroment value is always a string you can create your own local enviroment under any name.
