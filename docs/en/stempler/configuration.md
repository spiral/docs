# Stempler - Installation and Configuration

The Stempler engine provides a powerful and flexible template engine with an ability to customize it on the lexer, parser,
and AST compilation levels. By default, the driver is enabled with the web build of spiral skeleton application and provides 
support for Blade-like directives and echoing, HTML components, stacks, and more.

To install the extensions in alternative bundles:

```bash
composer require spiral/stempler-bridge
```

> **Note**
> Please note that the spiral/framework >= 2.7 already includes this component.

Make sure to add `Spiral\Stempler\Bootloader\StemplerBootloader` to your application kernel.

## Configuration

The Stempler bridge comes pre-configured. To replace and alter the default configuration, create a
file `app/config/views/stempler.php` with the following content:

```php
<?php

declare(strict_types=1);

use Spiral\Stempler\Builder;
use Spiral\Stempler\Directive;
use Spiral\Stempler\Transform\Finalizer;
use Spiral\Stempler\Transform\Visitor;
use Spiral\Views\Processor;

return [
    'directives' => [
        // available Blade-style directives
        Directive\PHPDirective::class,
        Directive\RouteDirective::class,
        Directive\LoopDirective::class,
        Directive\JsonDirective::class,
        Directive\ConditionalDirective::class,
        Directive\ContainerDirective::class
    ],
    'processors' => [
        // cache depended source processors (i.e. LocaleProcessor)
        Processor\ContextProcessor::class
    ],
    'visitors'   => [
        Builder::STAGE_PREPARE   => [
            // visitors to be invoked before transformations
            Visitor\DefineBlocks::class,
            Visitor\DefineAttributes::class,
            Visitor\DefineHidden::class
        ],
        Builder::STAGE_TRANSFORM => [
            // visitors to be invoked during transformations
        ],
        Builder::STAGE_FINALIZE  => [
            // visitors to be invoked on after the transformations is over
            Visitor\DefineStacks::class,
            Finalizer\StackCollector::class,
        ],
        Builder::STAGE_COMPILE   => [
            // visitors to be invoked on compilation stage
        ]
    ]
];
``` 

However, it's recommended that you use `StemplerBootloader` to properly configure the component. Several configuration options are available.

> **Note**
> Grammar and Parser configurations not exposed in the current Stempler Bridge version.

### Register custom Directive

To register a custom directive:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class CustomBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        StemplerBootloader::class
    ];

    public function boot(StemplerBootloader $stempler): void
    {
        $stempler->addDirective(Directive::class);
    }
}
```

### Register custom AST Visitor

To register a custom AST visitor:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class CustomBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        StemplerBootloader::class
    ];

    public function boot(StemplerBootloader $stempler): void
    {
        $stempler->addVisitor(Visitor::class);
    }
}
```

### Register source-code pre-processor

To register a custom source code pre-processor (cache specific):

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class CustomBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        StemplerBootloader::class
    ];

    public function boot(StemplerBootloader $stempler): void
    {
        $stempler->addProcessor(Processor::class);
    }
}
```

> **Note**
> Source-code pre-processing is considered internal functionality, so avoid using it in favor of custom AST processors.

## Additional Modules

The bridge comes with an additional bootloader called `Spiral\Stempler\Bootloader\PrettyPrintBootloader`, which is responsible for pretty printing the HTML content of your templates.
