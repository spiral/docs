# Debug - Dumping Variables

If you're using Spiral Framework 3.x, you might notice that there aren't any built-in tools for dumping variables.

However, you can use third-party packages like `symfony/var-dumper` for dumping variables. If you want to dump the 
contents of your variables to the RoadRunner error log, you can create a function like the one shown in the example below:

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
    function dumprr(mixed $value, mixed ...$values): mixed
    {
        $previous = $_SERVER['VAR_DUMPER_FORMAT'] ?? false;
        unset($_SERVER['VAR_DUMPER_FORMAT']);

        if (!\defined('STDERR')) {
            \define('STDERR', \fopen('php://stderr', 'wb'));
        }
        static $dumper = new CliDumper(STDERR);

        //
        // Output modifiers
        //
        $cloner = new VarCloner();
        // remove File and Line definitions from a custom closure dump
        /** @psalm-suppress InvalidArgument */
        $cloner->addCasters(ReflectionCaster::UNSET_CLOSURE_FILE_INFO);

        // Set new handler and store previous one
        $prevent = VarDumper::setHandler(static fn ($value) => $dumper->dump($cloner->cloneVar($value)));
        $result = VarDumper::dump($value);

        foreach ($values as $v) {
            VarDumper::dump($v);
        }

        // Reset handler
        VarDumper::setHandler($prevent);

        if ($previous) {
            $_SERVER['VAR_DUMPER_FORMAT'] = $previous;
        }

        return $result;
    }
}
```

> **Note**
> The `spiral/app` contains `symfony/var-dumper` package, and there's also a helpful function `dumprr` that you can use.
