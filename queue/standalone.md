# Queue and Jobs - Standalone Usage
You can run `spiral/jobs` as a standalone extension. 

> Make sure to read [Running Jobs](/queue/jobs.md) to understand the principles of job dispatching. The extension configuration is the same as described [here](/queue/configuration.md).

## Installation
The RoadRunner build with enabled jobs extension is available [here](https://github.com/spiral/framework/releases).

## PHP Worker
You would have to create a PHP worker to consume tasks from the queue. The primary class you have to implement is 
`Spiral\Jobs\HandlerRegistryInterface` which is responsible for providing job handler for a given job type.

For our purposes, we can use `Spiral\Jobs\Registry\ContainerRegistry` which can automatically convert job name into the class
name and request class instance from the associated PSR-11 container.

> We can use a spiral container provided by `spiral/core`.

```php
use Spiral\Jobs\Registry\ContainerRegistry;
use Spiral\Core\Container;

$registry = new ContainerRegistry(new Container());
```

Now we can configure our worker:

```php
use Spiral\Core\Container;
use Spiral\Goridge;
use Spiral\Jobs;
use Spiral\RoadRunner;

$rr = new RoadRunner\Worker(new Goridge\StreamRelay(STDIN, STDOUT));
```

To consume jobs:

```php
$consumer = new Jobs\Consumer($registry);
$consumer->serve($rr);
```

The final snippet will look like:

```php
/**
 * @var Goridge\RelayInterface $relay
 */

use Spiral\Core\Container;
use Spiral\Goridge;
use Spiral\Jobs;
use Spiral\RoadRunner;

require 'bootstrap.php';

$rr = new RoadRunner\Worker(new Goridge\StreamRelay(STDIN, STDOUT));

$registry = new Jobs\Registry\ContainerRegistry(new Container());

$consumer = new Jobs\Consumer($registry);
$consumer->serve($rr);
```

> You only need `spiral/jobs` and `spiral/core` to run this worker.
