# Advanced - Custom Dispatcher

It is possible to invoke application kernel using a custom data source, for example, **Kafka**, **state-machine**
events, or attach to user-defined interrupt. In this section, we will try to demonstrate how to write a RoadRunner
service plugin and a kernel dispatcher to consume data from this service. It is a good starting point for anyone who is
interested in building custom plugins for RoadRunner or in using the Spiral Framework to build scalable and extensible
web applications.

In this example, we will be sending "ticks" to the kernel every second. 

> **Attention**
> Make sure to read about an [application server](../framework/application-server.md) first. This article expects that
> you are proficient in writing Golang code.

## RoadRunner Service plugin

One way to take advantage of RoadRunner's performance is to use its plugin system, which allows you to extend the
functionality of the server and customize it to fit your needs.

In this tutorial, we'll show you how to create a simple RoadRunner plugin called "ticker", which will periodically send
ticks to the PHP workers with defined interval. This can be useful for tasks such as sending periodic updates to clients
or running scheduled tasks.

### Prerequisites

Before we begin, you'll need to have the following installed on your machine:

- [Go](https://golang.org/doc/install)
- [Velox](https://github.com/roadrunner-server/velox/releases)- the official RoadRunner builder tool. It allows you
  to build custom RoadRunner binaries from github and gitlab repositories.

> **Note**
> Read more how to create a RoadRunner plugin [here](https://roadrunner.dev/docs/plugins-intro/) and how to build a
> binary with custom plugins [here](https://roadrunner.dev/docs/app-server-build).

### Plugin Configuration

Here is an example of how to configure the ticker plugin in `.rr.yaml`:

```yaml
version: '2.7'

server:
  command: php app.php

ticker:
  interval: 1s
  pool:
    num_workers: 2
```

As you can see, our configuration allows us to define the interval between ticks in the format `1s, 1m, 10s, ...` and
configure the worker pool. The `interval` field specifies the amount of time to wait between ticks.

Let's create a configuration file `config.go` for our service:

```go
package ticker

import (
	"time"
	"github.com/roadrunner-server/sdk/v3/pool"
)

type Config struct {
	Interval time.Duration `mapstructure:"interval"`
	Pool     *pool.Config  `mapstructure:"pool"`
}

func (c *Config) InitDefaults() {
	if c.Pool == nil {
		c.Pool = &pool.Config{}
	}

    // Init default pool settings
	c.Pool.InitDefaults()

	// use default interval 1s when inteval is not defined or defined with wrong value 
	if c.Interval == 0 {
		c.Interval = time.Second
	}
}
```

In the `config.go` file, we defined a struct called `Config` to store the plugin configuration. It has an
`Interval` field for storing the tick interval and a `Pool` field for storing the worker pool configuration.
The `InitDefaults` function sets default values for these fields if they are not specified in the `.rr.yaml` file.
The default interval is set to 1 second, and the default worker pool configuration is set to the default values provided
by the RoadRunner SDK.

### Plugin Service

We have defined the configuration for our ticker plugin, let's move on to creating the plugin service.

The plugin service is responsible for managing the workers and sending the ticks to them. To create the service, create
a new file called `plugin.go` and add the following code to it:

```go
package ticker

import (
	"context"
	"fmt"
	"sync"
	"time"

	"github.com/roadrunner-server/errors"
	"github.com/roadrunner-server/sdk/v3/payload"
	"github.com/roadrunner-server/sdk/v3/pool"
	"github.com/roadrunner-server/sdk/v3/pool/static_pool"
	"github.com/roadrunner-server/sdk/v3/worker"
	"go.uber.org/zap"
)

type Configurer interface {
	// UnmarshalKey takes a single key and unmarshal it into a Struct.
	UnmarshalKey(name string, out any) error

	// Has checks if config section exists.
	Has(name string) bool
}

// Server creates workers for the application.
type Server interface {
	NewPool(ctx context.Context, cfg *pool.Config, env map[string]string, _ *zap.Logger) (*static_pool.Pool, error)
}

type Pool interface {
	// Workers returns worker list associated with the pool.
	Workers() (workers []*worker.Process)

	// Exec payload
	Exec(ctx context.Context, p *payload.Payload) (*payload.Payload, error)

	// Reset kill all workers inside the watcher and replaces with new
	Reset(ctx context.Context) error

	// Destroy all underlying stack (but let them to complete the task).
	Destroy(ctx context.Context)
}

const (
	rrMode     string = "RR_MODE"
	pluginName string = "ticker"
)

type Plugin struct {
	mu     sync.RWMutex
	cfg    *Config
	server Server
	stopCh chan struct{}
	pool   Pool
}

func (p *Plugin) Init(cfg Configurer, server Server) error {
	// If config file doesn't contain plugin section, ignore it
    if !cfg.Has(pluginName) {
		return errors.E(errors.Disabled)
	}

	// read plugin config
	err := cfg.UnmarshalKey(pluginName, &p.cfg)
	if err != nil {
		return err
	}

	p.cfg.InitDefaults()

	p.stopCh = make(chan struct{}, 1)
	p.server = server

	return nil
}

func (p *Plugin) Serve() chan error {
	errCh := make(chan error, 1)

	var err error
	p.mu.Lock()
    // Create workers pool
	p.pool, err = p.server.NewPool(context.Background(), p.cfg.Pool, map[string]string{rrMode: pluginName}, nil)
	p.mu.Unlock()

	if err != nil {
		errCh <- err
		return errCh
	}

	go func() {
		var numTicks = 0
		var lastTick time.Time
        // Be careful with ticker! You should always stop it
		ticker := time.NewTicker(p.cfg.Interval)
		defer ticker.Stop()

		for {
			select {
			case <-p.stopCh:
				return
			case <-ticker.C:
				p.mu.RLock()
				_, err2 := p.pool.Exec(context.Background(), &payload.Payload{
					Context: []byte(fmt.Sprintf(`{"lastTick": %v}`, lastTick.Unix())),
					Body:    []byte(fmt.Sprintf(`{"tick": %v}`, numTicks)),
				})
				p.mu.RUnlock()
				if err != nil {
					errCh <- err2
					return
				}

				numTicks++
				lastTick = time.Now()
			}
		}
	}()

	return errCh
}

func (p *Plugin) Reset() error {
	p.mu.RLock()
	defer p.mu.RUnlock()

	if p.pool == nil {
		return nil
	}

	ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
	defer cancel()
	err := p.pool.Reset(ctx)
	if err != nil {
		return err
	}

	return nil
}

func (p *Plugin) Stop() error {
	p.stopCh <- struct{}{}
	return nil
}

func (p *Plugin) Name() string {
	return pluginName
}

func (p *Plugin) Weight() uint {
	return 10
}
```

When RoadRunner starts PHP workers, it can pass a value for the `RR_MODE` variable to indicate which plugin should be
used. The Spiral Framework can then use the value of this variable to choose the appropriate dispatcher for the current
environment.

### Build the RoadRunner Binary with Velox

Next, we will use [Velox](https://roadrunner.dev/docs/app-server-build) to build a custom RoadRunner binary with our
plugin.

Create a new file called `plugins.toml` and add the following configuration:

```toml
[velox]
build_args = ['-trimpath', '-ldflags', '-s -X github.com/roadrunner-server/roadrunner/v2/internal/meta.version=v2.12.1.custom -X github.com/roadrunner-server/roadrunner/v2/internal/meta.buildTime=00:00:00']

[roadrunner]
ref = "v2.12.1"

[github]
[github.token]
token = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# ref -> master, commit or tag
[github.plugins]
logger = { ref = "master", owner = "roadrunner-server", repository = "logger" }
server = { ref = "master", owner = "roadrunner-server", repository = "server" }
ticker = { ref = "main", owner = "roadrunner-php", repository = "rr-examples", folder = "ticker" }


[log]
level = "debug"
mode = "development"
```

Then, run the following command to build the RoadRunner binary:

```bash
vx build -c plugins.toml -o .
```

## Application Dispatcher

At first, we need to install `spiral/roadrunner-worker` package:

```bash
composer require spiral/roadrunner-worker
```

Now we can create our dispatcher:

```php
namespace App\Dispatcher;

use Psr\Container\ContainerInterface;
use Spiral\Boot\DispatcherInterface;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\FinalizerInterface;
use Spiral\RoadRunner\Worker;

final class TickerDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env,
        private readonly FinalizerInterface $finalizer,
        private readonly ContainerInterface $container
    ) {
    }

    public function canServe(): bool
    {
        return $this->env->get('RR_MODE') === 'ticker';
    }

    public function serve(): void
    {
        /** @var Worker $worker */
        $worker = $this->container->get(Worker::class);

        while ($payload = $worker->waitPayload()) {
            $data = \json_decode($payload->body, true);
            
            // Handle tick ... 
        
            // Respond Answer
            $worker->respond(new \Spiral\RoadRunner\Payload('OK'));

            // reset some stateful services
            $this->finalizer->finalize();
        }
    }
}
```

> **Note**
> Read more about the Dispatchers [here](../framework/dispatcher.md).

Create a Bootloader to register our dispatcher in the kernel:

```php
namespace App\Bootloader;

use App\Dispatcher\TickerDispatcher;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

final class TickerBootloader extends Bootloader
{
    public function boot(KernelInterface $kernel, TickerDispatcher $ticker): void
    {
        $kernel->addDispatcher($ticker);
    }
}
```

Now we can run our application:

```bash
./rr serve
```

> **Note:**
> You can find code of ticker plugin [here](https://github.com/roadrunner-php/rr-examples)