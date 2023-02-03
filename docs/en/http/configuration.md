# HTTP — Getting started

The web application bundle (`spiral/app`) is shipped with a pre-configured HTTP component. You will need several extensions
to enable it in alternative builds.

## Installation

To install the extension:

```terminal
composer require spiral/nyholm-bridge
```

Activate the component by adding two bootloaders:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...

    // Fast PSR-7 implementation
    \Spiral\Nyholm\Bootloader\NyholmBootloader::class,

    // HTTP core
    \Spiral\Bootloader\Http\HttpBootloader::class,

    // PSR-15 handler      
    \Spiral\Bootloader\Http\RouterBootloader::class,

    // ...
];
```

> **Note**
> See how to use custom PSR-15 handler [here](../cookbook/psr-15.md).

Make sure to configure [routing](../http/routing.md).

## Configuration

The HTTP extension can be configured via `app/config/http.php` file:

```php app/config/http.php
return [
    // default base path
    'basePath'   => '/',
    
    // default headers
    'headers'    => [
        'Content-Type' => 'text/html; charset=UTF-8'
    ],

    // application level middleware
    'middleware' => [
        // middleware class name
    ],
];
```

> **Note**
> The default configuration will be used if such file does not exist.