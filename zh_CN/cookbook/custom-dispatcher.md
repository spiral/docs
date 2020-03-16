# 速查手册 - 自定义调度器

在 Spiral 中，可以通过自定义数据源（例如 Kafka, 状态机事件或附加到用户定义的中断）来调用应用内核。在本章节中，我们来演示如何编写 RoadRunner 服务和内核调度器，并用该内核调度器来消费服务产生的数据。在示例中，我们每秒钟向内核发送一次“tick”。

> 提示：请务必先阅读[应用服务器](/framework/application-server)章节。本文涉及的内容需要您精通 Golang 编程语言。

## RoadRunner 服务

首先，我们要通过封装的进程服务器创建 RoadRunner 服务。请参考阅读以下文档：

- https://roadrunner.dev/docs/beep-beep-service
- https://roadrunner.dev/docs/library-standalone-usage

我们要创建的服务需要一些配置项：

```go
package ticker

import (
	"errors"
	"github.com/spiral/roadrunner"
	"github.com/spiral/roadrunner/service"
)

// Config 负责配置 RoadRunner 的 HTTP 服务器。
type Config struct {
    // Interval 定义了 tick 的间隔，单位是秒
	Interval int

    // Workers 配置 rr 服务器和工作进程池
	Workers *roadrunner.ServerConfig
}

// Hydrate 必须使用给定的 Config 数据填充 Config 的值。如果传入的 Config 格式不符合，必须返回错误
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
		return errors.New("间隔时间不能小于 1 秒")
	}

	return nil
}
```

写好配置对象之后，接下来实现服务：

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

    // 声明我们的服务是针对应用内核的
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
                // 省略了错误处理
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

代码写完之后，可以在构建 roadrunner 时启用刚刚创建的服务，也可以在 `.rr` 配置中启用：

在 RoadRunner 中启用：

```go
rr.Container.Register(ticker.ID, &ticker.Service{})
```

在配置文件中启用：

```yaml
ticker:
    internal: 1
    workers.command: "php app.php"
```

## 应用调度器

然后就可以创建我们的调度器：

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

            // 在这里做点啥
            file_put_contents('ticks.txt', $numTick . "\n", FILE_APPEND);

            $worker->send("OK");

            // 重置有状态服务
            $this->finalizer->finalize();
        }
    }
}
```

还没完，别忘了创建一个引导程序，在内核中注册我们的调度器：

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

至此，就可以构建应用服务器，然后测试刚刚实现的调度器了。
