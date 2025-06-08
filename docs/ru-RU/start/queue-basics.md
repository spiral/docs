# Начало работы — Первая фоновая задача

В этом руководстве я проведу вас через процесс создания и запуска фоновой задачи (job) с использованием Spiral и сервера приложений [RoadRunner](https://roadrunner.dev/). Это позволит вам выполнять задачи асинхронно, давая возможность вашему приложению продолжать свою работу, пока фоновая задача выполняется в фоне.

Давайте разберем основные шаги, которые нужно выполнить:

## Создание задачи (Job)

Для создания вашей первой задачи без усилий используйте команду генерации кода:

```terminal
php app.php create:jobHandler PingSite
```

> **Примечание**
> Подробнее о генерации кода читайте в разделе [Основы — Генерация кода](../basics/scaffolding.md#job-handler).

После выполнения этой команды следующий вывод подтвердит успешное создание:

```output
Declaration of '[32mPingSiteJob[39m' has been successfully written into '[33mapp/src/Endpoint/Job/PingSiteJob.php[39m'.
```

Теперь давайте добавим логику в наш только что созданный обработчик задач.

Вот пример задачи, которая отправляет `GET`-запрос на указанный сайт:

```php app/src/Endpoint/Job/PingSiteJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class PingSiteJob extends JobHandler
{
    public function invoke(HttpClientInterface $client, string $site): void
    {
        $response = $client->request('GET', $site);
        
        // делаем что-то с ответом ...
    }
}
```

## Конфигурация

Убедитесь, что плагин jobs включен в конфигурационном файле RoadRunner `.rr.yaml`:

```yaml .rr.yaml
rpc:
  listen: 'tcp://127.0.0.1:6001'

jobs:
  consume: { }

# ...
```

Далее нам нужно настроить наше приложение для отправки задач в RoadRunner. Откройте конфигурационный файл `app/config/queue.php` и примените изменения ниже:

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\MemoryCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'pipelines' => [
        'memory' => [
            'connector' => new MemoryCreateInfo('local'),
            'consume' => true,
        ]
    ],
            
    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'default' => 'memory',
        ],
    ],
];
```

Эти настройки конфигурируют наше приложение для создания нового `in-memory` пайплайна для сервера RoadRunner. Всякий раз, когда мы помещаем задачу в этот пайплайн, она будет добавлена в очередь `in-memory`. RoadRunner затем отправит её потребителю для обработки.

## Запуск задачи

Теперь, когда наша задача и RoadRunner настроены, мы можем создать консольную команду для помещения задачи в очередь.

Давайте создадим команду, которая поместит `PingSiteJob` в очередь:

```terminal
php app.php create:command PingSite
```

```php app/src/Endpoint/Console/PingSiteCommand.php
namespace App\Endpoint\Console;

use App\Endpoint\Job\PingSiteJob;
use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Command;
use Spiral\Queue\QueueInterface;

#[AsCommand(name: 'ping:site', description: 'Ping site')]
final class PingSiteCommand extends Command
{
    #[Argument(description: 'Site to ping')]
    public string $site;

    public function __invoke(QueueInterface $queue): int
    {
        $id = $queue->push(PingSiteJob::class, [
            'site' => $this->site,
        ]);

        $this->writeln(\sprintf('Job %s pushed', $id));

        return self::SUCCESS;
    }
}
```

В приведенном примере мы внедряем `QueueInterface` в метод. Это позволяет контейнеру внедрения зависимостей автоматически разрешить его и предоставить экземпляр подключения к очереди по умолчанию, указанного в конфигурационном файле.

#### Запуск сервера RoadRunner

Для запуска задачи нам сначала нужно запустить сервер RoadRunner следующей командой:

```terminal
./rr serve
```

#### Выполнение консольной команды

Теперь пришло время выполнить нашу консольную команду и поместить задачу в очередь. Используйте следующую команду:

```terminal
php app.php ping:site "https://google.com"
```

Вы должны увидеть следующий вывод:

```output
Job [32m3332e595-9774-434c-908c-3c419f80c967[39m pushed
```

После того как задача помещена в очередь, она будет подхвачена RoadRunner, который затем передаст её потребителю для обработки.

Вот и всё! Поздравляем с созданием вашей первой фоновой задачи с использованием Spiral и RoadRunner. С этой настройкой вы можете легко добавлять больше задач и выполнять задачи асинхронно для обеспечения бесперебойной работы вашего приложения.

<hr>

## Что дальше?

Теперь углубитесь в основы, прочитав некоторые статьи:

* [Очереди и задачи](../queue/configuration.md)
* [Перехватчики (Interceptor) очередей](../queue/interceptors.md)
* [Создание консольной команды](../console/commands.md)
* [Генерация кода](../basics/scaffolding.md)
