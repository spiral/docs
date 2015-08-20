# Configuring your Application
Most of spiral components and services requires external configuration to defined it's behaviour. Configuration data 
usually supplied in array form using `Spiral\Core\ConfiguratorInterface` instance. 

Spiral Framework core implements such interface and uses application `config` directory to load configuration files,
every loaded configuration file will be merged with envrioment specific configuration and stored in permanent cache.

## Enviroment specific configurations
If you want to change some section of configuration file based on active enviroment, simply create directory inside
config folder named as your enviroment and put configuration file in it. Spiral will replace original configuration
sections with sections provided by this file.

## Examples
Let's look into same configuration file for `ViewProvider` component (`config/views.php`):

```php
<?php
/**
 * Configuration of ViewManager component and view engines:
 * - compiled view cache state and location
 * - view namespaces associated with list of source directories
 * - list of view dependencies, used by default compiler to create unique cache name
 * - list of view engines associated with their extension, compiler and default view class
 * - configuration of default spiral compiler it's processors and other options
 * - list of view classes associations (path = class, path must always include namespace)
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
    ],
    'classes'      => []
];
```

If we wish to diable caching in development enviroment we can create file `config/development/views.php` with content like that:

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

Configuration section `cache` in original config will be replaced with new array.