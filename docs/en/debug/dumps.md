# Debug - Dumping Variables

Spiral Framework 3.0 doesn't have any  out of the box tools for dumping variables.

But you can use third-party packages like `symfony/var-dumper` to view the content of your variables and instances.

If you want to dump the contents of your variables to the RoadRunner error log, you need to create a function, like in 
the example below: 

```php
<?php

declare(strict_types=1);

use Symfony\Component\VarDumper\Caster\ReflectionCaster;
use Symfony\Component\VarDumper\Cloner\VarCloner;
use Symfony\Component\VarDumper\Dumper\CliDumper;
use Symfony\Component\VarDumper\VarDumper;

if (!\function_exists('dumprr')) {
    /**
     * Dump value int STDERR.
     */
    function dumprr(mixed $value): mixed
    {
        if (!\defined('STDERR')) {
            \define('STDERR', \fopen('php://stderr', 'wb'));
        }

        static $dumper = new CliDumper(STDERR);

        //
        // Output modifiers
        //
        $cloner = new VarCloner();
        // remove File and Line definitions from a custom closure dump
        $cloner->addCasters(ReflectionCaster::UNSET_CLOSURE_FILE_INFO);

        // Set new handler and store previous one
        $prevent = VarDumper::setHandler(static fn ($value) => $dumper->dump($cloner->cloneVar($value)));
        $result = VarDumper::dump($value);

        // Reset handler
        VarDumper::setHandler($prevent);

        return $result;
    }
}
```

> The package is included into `spiral/app`, and there's also a helper function `dumprr`.
