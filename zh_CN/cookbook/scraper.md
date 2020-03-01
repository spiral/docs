# 速查手册 - 网站爬虫

Spiral 队列和 RoadRunner 服务器可以用来实现不同于传统 web 设置的应用程序。这篇教程中会演示开发一个在命令行下使用的简单网站爬虫程序。

爬取到的数据会存储在 `runtime` 文件夹中。

> 教程开发的代码只是为了进行基础演示，有很多需要改进的地方，不建议直接使用。

## 安装依赖

我们会基于 Spiral 的命令行项目框架 [spiral/app-cli](https://github.com/spiral/app-cli) 创建这个命令行程序。`app-cli` 是 Spiral 提供的不含 ORM, HTTP 和其它扩展的精简项目模板，通常用于开发控制台应用程序。

执行下面的命令创建新项目：

```bash
$ composer create-project spiral/app-cli scraper
$ cd scraper
```

为了实现网站爬虫所需的功能，要用到的扩展组件包括这些：

组件                | 说明
---                      | ---
spiral/jobs              | 队列任务支持
spiral/scaffolder        | 快速脚手架（开发环境用）
spiral/prototype         | 快速原型开发辅助（开发环境用）
paquettg/php-html-parser | 解析 HTML

首先安装依赖项，并下载应用服务器：

```bash
$ composer require spiral/jobs paquettg/php-html-parser spiral/scaffolder spiral/prototype
$ ./vendor/bin/spiral get 
```

在 `App\App` 类中激活安装的扩展组件：

```php
namespace App;

use Spiral\Bootloader;
use Spiral\Framework\Kernel;
use Spiral\Prototype\Bootloader as Prototype;
use Spiral\Scaffolder\Bootloader\ScaffolderBootloader;

class App extends Kernel
{
    protected const LOAD = [
        Bootloader\DebugBootloader::class,
        Bootloader\CommandBootloader::class,
        Bootloader\Jobs\JobsBootloader::class,

        // 以下两个依赖在开发完成后可以移除
        ScaffolderBootloader::class,
        Prototype\PrototypeBootloader::class
    ];
}
```

配置完成后记得要执行一次 `php app.php configure` 来确保项目已经正确安装。

```bash
$ php app.php conf
Configuring project:

[runtime] verify `runtime` directory access
Verifying runtime directory... exists
Runtime directory permissions were updated.

All done!
```

## 配置应用服务器

作为演示，我们使用默认的一个内存队列。在项目的根目录下创建 `.rr.yaml` 文件：

```yaml
jobs:
  dispatch:
    # 所有 App\Job\* 任务都分发到 local 队列
    app-job-*.pipeline: "local"

  pipelines:
    # 把 local 队列关联到 `ephemeral` 队列服务器（内置的基于内存的队列服务器）
    local.broker: "ephemeral"
  
  # 消费 local 队列
  consume: ["local"]
  
  # 进程命令和进程数
  workers:
    command: "php app.php"
  
    # 对 IO 密集型应用程序，可以适当增加工作进程数量
    pool.numWorkers: 16
```

## 创建任务处理程序

接下来创建一个简单的任务处理程序，它负责扫描网站，爬取 HTML 内容，并在指定的深度限制范围内跟随网页中的链接跳转和爬取。所有抓到的内容会存储在 `runtime` 文件夹下。

> 因为安装了命令脚手架，可以通过 `php app.php create:job scrape` 来创建任务处理类。为了缩短篇幅，我们的示例中不使用 CURL 而是是用 `file_get_contents`.

```php
namespace App\Job;

use PHPHtmlParser\Dom;
use Spiral\Jobs\JobHandler;
use Spiral\Prototype\Traits\PrototypeTrait;

class ScrapeJob extends JobHandler
{
    use PrototypeTrait;

    public function invoke(int $depth, string $url): void
    {
        if ($this->files->exists(directory('runtime') . md5($url) . '.html')) {
            // 跳过已抓取的页面
            return;
        }

        $body = file_get_contents($url);
        $this->store($url, $body);

        $dom = new Dom();
        $dom->load($body);

        foreach ($dom->find('a') as $a) {
            $next = $this->nextURL($url, $a->href);

            if ($next !== null && $depth > 1) {
                $this->queue->push(self::class, ['depth' => $depth - 1, 'url' => $next]);
            }
        }
    }

    private function store(string $url, string $body)
    {
        $this->files->write(directory('runtime') . md5($url) . '.html', $body);

        $this->files->append(
            directory('runtime') . 'scrape.log',
            sprintf("%s,%s,%s\n", date('c'), md5($url), $url)
        );
    }

    private function nextURL(string $base, ?string $target): ?string
    {
        if ($target == null) {
            return null;
        }

        $base_url = parse_url($base);
        $target_url = parse_url($target);

        if (isset($target_url['scheme']) && isset($target_url['host'])) {
            if ($target_url['host'] !== $base_url['host']) {
                // 只抓取与起点同域名下的链接
                return null;
            }

            // url 包含 scheme 和 host 的完整链接直接返回
            return $target;
        }

        if (!isset($target_url['path'])) {
            return null;
        }

        // 这里是把站内相对链接变成完整链接返回
        return sprintf("%s://%s%s", $base_url['scheme'], $base_url['host'], $target_url['path']);
    }
}
```

## 创建命令

通过脚手架，执行 `php app.php create:command scrape` 来创建控制台命令类：

```php
namespace App\Command;

use App\Job\ScrapeJob;
use Spiral\Console\Command;
use Spiral\Jobs\QueueInterface;
use Symfony\Component\Console\Input\InputArgument;

class ScrapeCommand extends Command
{
    public const NAME        = 'scrape';
    public const DESCRIPTION = '爬取网页';
    public const ARGUMENTS   = [
        ['url', InputArgument::REQUIRED, '要爬取的 URL'],
        ['depth', InputArgument::OPTIONAL, '要扫描的深度', 10],
    ];

    protected function perform(QueueInterface $queue): void
    {
        $queue->push(ScrapeJob::class, [
            'url'   => $this->argument('url'),
            'depth' => (int)$this->argument('depth')
        ]);

        $this->writeln("开始爬取!");
    }
}
```

## 测试一下

首先启动应用服务器：

```bash
$ ./spiral serve -v -d
```

> 这里添加 `-v` 参数可以输出详细的信息

另开一个终端，通过命令行命令开始爬取任意的 URL（要保持应用服务器运行中）：

```bash
$ php app.php scrape https://some-website.com/ 5
```

> 上面的命令从 `https://some-website.com/` 开始，进行深度为 5 的爬取

应用服务器还提供了以下查看队列状态的命令，可以通过交互式的控制台查看抓取了多少页面：

```bash
$ ./spiral jobs:stat -i
```

> 实例应用会反复爬取同一个页面很多次（由于网页中反复出现相同的链接），实际项目中要使用适当的数据库或者锁定机制来避免这种情况。
