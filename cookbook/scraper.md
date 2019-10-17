# Cookbook - Website Scraper
We can use Spiral Queue and RoadRunner server to implement applications different from classic web setup. In this
tutorial we will try to implement a simple web-scraper application for CLI usage.

The scraped data will be stored in a `runtime` folder. 

> The produced code only demonstrates the capabilities and can be improved a lot.

## Installing Dependencies
We will base our application on `spiral/app-cli` - the minimalistic spiral build without ORM, HTTP and other extensions.

```bash
$ composer create-project spiral/app-cli scraper
$ cd scraper
```

To implement all needed features we will need a set of extensions:

Extension                | Comment
---                      | ---
spiral/jobs              | Queue support
spiral/scaffolder        | Faster scaffolding (dev only)
spiral/prototype         | Faster prototyping (dev only)
paquettg/php-html-parser | Parsing HTML

To install all needed packages and download app server:

```bash
$ composer require spiral/jobs spiral/scaffolder spiral/prototype paquettg/php-html-parser
$ ./vendor/bin/spiral get 
```

Activate the installed extensions in your `App\App`:

```php
namespace App;

use Spiral\Bootloader;
use Spiral\Framework\Kernel;
use Spiral\Prototype\Bootloader as Prototype;
use Spiral\Scaffolder\Bootloader as Scaffolder;

class App extends Kernel
{
    protected const LOAD = [
        Bootloader\DebugBootloader::class,
        Bootloader\CommandBootloader::class,
        Bootloader\Jobs\JobsBootloader::class,

        Scaffolder\ScaffolderBootloader::class,
        Prototype\PrototypeBootloader::class
    ];
}
```

Make sure to run `php app.php configure` to ensure proper installation.

## Configure App Server
Let's configure application server with one default queue in memory. Create `.rr.yaml` file in a root of the project:

```yaml
jobs:
  dispatch:
    # dispatch all the App\Job\* into local pipeline
    app-job-*.pipeline: "local"

  pipelines:
    # associate local pipeline with `ephemeral` (in-memory) broker
    local.broker: "ephemeral"
  
  # consume local pipeline
  consume: ["local"]
  
  # endpoint and worker count
  workers:
    command: "php app.php"
  
    # increase number of workers for IO bound applications
    pool.numWorkers: 16
```

## Create Job Handler
Now, let's write a simple job handler which will scan website, get the HTML content and jump by links util the specific
depth is reached. All the content will be stored in `runtime` directory.

> Create JobHandler via `php app.php create:job scrape`. We are not going to use CURL for simplicity.

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
            // already scrapped
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
                // only same domain
                return null;
            }

            // full path
            return $target;
        }

        if (!isset($target_url['path'])) {
            return null;
        }

        // relative path
        return sprintf("%s://%s%s", $base_url['scheme'], $base_url['host'], $target_url['path']);
    }
}
```

## Create command
Create command to start scraping `php app.php create:command scrape`:

```php
namespace App\Command;

use App\Job\ScrapeJob;
use Spiral\Console\Command;
use Spiral\Jobs\QueueInterface;
use Symfony\Component\Console\Input\InputArgument;

class ScrapeCommand extends Command
{
    public const NAME        = 'scrape';
    public const DESCRIPTION = 'Scape the page';
    public const ARGUMENTS   = [
        ['url', InputArgument::REQUIRED, 'Url to scape'],
        ['depth', InputArgument::OPTIONAL, 'Depth of scanning', 10],
    ];

    protected function perform(QueueInterface $queue): void
    {
        $queue->push(ScrapeJob::class, [
            'url'   => $this->argument('url'),
            'depth' => (int)$this->argument('depth')
        ]);

        $this->writeln("Started!");
    }
}
```

## Test it
Launch application server first:

```bash
$ ./spiral serve -v -d
```

Scape any URL via console command (keep the server running):

```bash
$ php app.php scrape https://some-website.com/ 5
```

To observe how many pages are being scraped via interactive console:

```bash
$ ./spiral jobs:stat -i
```

> The demo solution will scan some pages multiple times, use proper database or lock mechanism to avoid that.
