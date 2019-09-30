# Advanced - Custom Dispatcher
It is possible to invoke application kernel using a custom data source, for example, Kafka, NSQ or attach to user-defined
interrupt. In this section, we will try to demonstrate how to write RoadRunner service and kernel dispatcher to consume
data from this service. In this example, we will be sending "ticks" to the kernel every second.

> Attention, make sure to read about [application server](/framework/application-server.md) first. This article expects
> that you are proficient in writing Golang code.

## RoadRunner Service
First, let's create RoadRunner service with encapsulated worker server. Check these articles for the references:
- https://roadrunner.dev/docs/beep-beep-service
- https://roadrunner.dev/docs/library-standalone-usage

We will need a configuration for our service:

```go
package ticker

import (
	"errors"
	"github.com/spiral/roadrunner"
	"github.com/spiral/roadrunner/service"
)

// Config configures RoadRunner HTTP server.
type Config struct {
	// Interval defines tick internal in seconds.
	Interval int

	// Workers configures rr server and worker pool.
	Workers *roadrunner.ServerConfig
}

// Hydrate must populate Config values using a given Config source. Must return an error if Config is not valid.
func (c *Config) Hydrate(cfg service.Config) error {
	if c.Workers == nil {
		c.Workers = &roadrunner.ServerConfig{}
	}

	c.Workers.InitDefaults()

	if err := cfg.Unmarshal(c); err != nil {
		return err
	}

	c.Workers.UpscaleDurations()

	if c.Interval < 1 {
		return errors.New("interval must be at least one second")
	}

	return nil
}
```

And the service itself:

```go
package ticker

import (
	"fmt"
	"github.com/spiral/roadrunner"
	"github.com/spiral/roadrunner/service/env"
	"time"
)

const ID = "ticker"

type Service struct {
	cfg  *Config
	env  env.Environment
	stop chan interface{}
}

func (s *Service) Init(cfg *Config, env env.Environment) (bool, error) {
	s.cfg = cfg
	s.env = env
	return true, nil
}

func (s *Service) Serve() error {
	s.stop = make(chan interface{})

	if s.env != nil {
		if err := s.env.Copy(s.cfg.Workers); err != nil {
			return nil
		}
	}

	// identify our service for app kernel
	s.cfg.Workers.SetEnv("rr_ticker", "true")

	rr := roadrunner.NewServer(s.cfg.Workers)
	defer rr.Stop()

	if err := rr.Start(); err != nil {
		return err
	}

	go func() {
		var (
			numTicks = 0
			lastTick time.Time
		)

		for {
			select {
			case <-s.stop:
				return
			case <-time.NewTicker(time.Second * time.Duration(s.cfg.Interval)).C:
				// error handling is omitted
				rr.Exec(&roadrunner.Payload{
					Context: []byte(fmt.Sprintf(`{"lastTick": %v}`, lastTick.Unix())),
					Body:    []byte(fmt.Sprintf(`{"tick": %v}`, numTicks)),
				})

				lastTick = time.Now()
				numTicks++
			}
		}
	}()

	<-s.stop
	return nil
}

func (s *Service) Stop() {
	close(s.stop)
}
```

We can enable this service in our roadrunner build and `.rr` configuration:

```go
rr.Container.Register(ticker.ID, &ticker.Service{})
```

In configuration:

```yaml
ticker:
    internal: 1
    workers.command: "php app.php"
```

## Application Dispatcher
Now we can create our dispatcher:

```php
namespace App\Dispatcher;

use Psr\Container\ContainerInterface;
use Spiral\Boot\DispatcherInterface;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\FinalizerInterface;
use Spiral\RoadRunner\Worker;

class TickerDispatcher implements DispatcherInterface
{
    /** @var EnvironmentInterface */
    private $env;

    /** @var FinalizerInterface */
    private $finalizer;

    /** @var ContainerInterface */
    private $container;

    public function __construct(
        EnvironmentInterface $env,
        FinalizerInterface $finalizer,
        ContainerInterface $container
    ) {
        $this->env = $env;
        $this->finalizer = $finalizer;
        $this->container = $container;
    }

    public function canServe(): bool
    {
        return (php_sapi_name() == 'cli' && $this->env->get('RR_TICKER') !== null);
    }

    public function serve()
    {
        /** @var Worker $worker */
        $worker = $this->container->get(Worker::class);

        while (($body = $worker->receive($ctx)) !== null) {
            $lastTick = json_decode($ctx)->lastTick;
            $numTick = json_decode($body)->tick;

            // do something
            file_put_contents('ticks.txt', $numTick . "\n", FILE_APPEND);

            $worker->send("OK");

            // reset some stateful services
            $this->finalizer->finalize();
        }
    }
}
```

Create bootloader to register our dispatcher in the kernel:

```php
namespace App\Bootloader;

use App\Dispatcher\TickerDispatcher;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

class TickerBootloader extends Bootloader
{
    public function boot(KernelInterface $kernel, TickerDispatcher $ticker)
    {
        $kernel->addDispatcher($ticker);
    }
}
```

We can build our application server and test dispatcher now.