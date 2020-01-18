# Stempler - Installation and Configuration
Stempler engine provides powerful and flexible template engine with an ability to customize it on lexer, parser and AST
compilation levels. By default engine is enabled with web build of spiral skeleton application and provide support
for Blade-like directives and echoing, html components, stacks and others.

In order to install the extensions in alternative bundles:

```bash
$ composer require spiral/stempler-bridge
```

Make sure to add `Spiral\Stempler\Bootloader\StemplerBootloader` to your application kernel.

## Configuration
The stempler bridge comes pre-configured. To replace and alter the default configuration create file `app/config/views/stempler.php`
with the following content:

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
            Visitor\DefineHidden::class,
            Visitor\DefineStacks::class
        ],
        Builder::STAGE_TRANSFORM => [
            // visitors to be invoked during transformations
        ],
        Builder::STAGE_FINALIZE  => [
            // visitors to be invoked on after the transformations is over
            Finalizer\StackCollector::class,
        ],
        Builder::STAGE_COMPILE   => [
            // visitors to be invoked on compilation stage
        ]
    ]
];
``` 

However, it is recommend to use `StemplerBootloader` in order to properly configure the component. A number of configuration
options is available.

> Grammar and Parser configurations are not exposed in current Stempler Bridge version.

### Register custom Directive
To register custom directive.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class CustomBootloader extends Bootloader
{
    protected const DEPENDENCIES = [StemplerBootloader::class];

    public function boot(StemplerBootloader $stempler)
    {
        $stempler->addDirective(Directive::class);
    }
}
```

### Register custom AST Visitor
To register custom AST visitor.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class CustomBootloader extends Bootloader
{
    protected const DEPENDENCIES = [StemplerBootloader::class];

    public function boot(StemplerBootloader $stempler)
    {
        $stempler->addVisitor(Visitor::class);
    }
}
```

### Register source-code pre-processor
To register custom source code pre-processor (cache specific):

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class CustomBootloader extends Bootloader
{
    protected const DEPENDENCIES = [StemplerBootloader::class];

    public function boot(StemplerBootloader $stempler)
    {
        $stempler->addProcessor(Processor::class);
    }
}
```

> The source-code pre-processing is considered internal functionality, avoid using it in favor of custom AST processors.

## Additional Modules
The bridge comes with additional bootloader `Spiral\Stempler\Bootloader\PrettyPrintBootloader` responsible for pretty 
printing of HTML content of your templates.
