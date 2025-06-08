# Продвинутые возможности — Метрики приложения

Как профессионал, вы знаете важность отслеживания ключевых метрик для вашего приложения. С [Prometheus](https://prometheus.io/) вы можете собирать и хранить данные временных рядов, такие как метрики приложения, и использовать его мощный язык запросов для анализа и визуализации этих данных в реальном времени с помощью инструмента, такого как [Grafana](https://grafana.com/), что экономит ваше время и усилия по созданию собственной панели управления с нуля.

![Grafana dashboard](https://user-images.githubusercontent.com/773481/205066017-ecddefc4-1d07-4428-b3ad-af49baadad0a.png)

Spiral и плагин [RoadRunner metrics](https://roadrunner.dev/docs/plugins-metrics) предоставляют возможность собирать метрики приложения и предоставлять их для Prometheus.

> **Примечание**
> Здесь вы можете узнать больше о [метриках Prometheus](https://prometheus.io/docs/concepts/data_model/).

## Установка

Сначала вам нужно установить пакет [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge).

> **Примечание**
> Пакет `spiral/roadrunner-bridge` позволяет использовать [плагин метрик](https://roadrunner.dev/docs/lab-metrics) RoadRunner с Spiral. Этот пакет предоставляет RPC API для метрик и загрузчик (bootloader) для вашего приложения.

После установки пакета вы можете добавить `Spiral\RoadRunnerBridge\Bootloader\MetricsBootloader` в список загрузчиков:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\MetricsBootloader::class,
        // ...
    ];
}
```

Читайте больше о загрузчиках в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\MetricsBootloader::class,
    // ...
];
```

Читайте больше о загрузчиках в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

## Конфигурация

Сервис метрик не требует конфигурации в приложении. Однако вы должны активировать сервис в `.rr.yaml`:

```yaml
rpc:
  listen: tcp://127.0.0.1:6001

# ...

metrics:
  # адрес клиента prometheus (путь /metrics добавляется автоматически)
  address: 127.0.0.1:2112
```

> **Примечание**
> Вы можете просмотреть метрики по умолчанию на http://127.0.0.1:2112

## Использование

### Объявление метрик приложения

Есть два способа объявления специфичных для приложения метрик в приложении Spiral:

:::: tabs
::: tab RoadRunner
Используя файл `.rr.yaml`:

```yaml .rr.yaml
metrics:
  address: 127.0.0.1:2112

  collect:
    registered_users:
      type: counter
      help: "Total registered users counter."
```

:::

::: tab PHP
Объявление метрик в PHP коде

```php app/src/Application/Bootloader/MetricsBootloader.php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;

class MetricsBootloader extends Bootloader
{
    //...

    public function boot(MetricsInterface $metrics): void
    {
        $metrics->declare(
            'registered_users',
            Collector::counter()->withHelp('Total registered users counter.')
        );
    }
}
```

:::

::::

Чтобы заполнить метрику из приложения, используйте `Spiral\RoadRunner\Metrics\MetricsInterface`:

```php
use Spiral\RoadRunner\Metrics\MetricsInterface; 

class UserRegistrationHandler
{
    public function __construct(
        private readonly MetricsInterface $metrics
    ) {
    }

    public function handle(User $user): void
    {
        // Сохранить пользователя в базе данных

        $this->metrics->add('registered_users', 1);
    }
}
```

> **Читайте больше**
> Поддерживаемые типы: gauge, counter, summary, histogram. Читайте больше о типах метрик в [официальной документации Prometheus](https://prometheus.io/docs/concepts/metric_types/).

### Метрики с тегами

Использование метрик с тегами (также известных как метрики с метками) позволяет присоединять дополнительные метаданные к вашим метрикам, что может быть полезно для фильтрации, группировки и агрегации данных.

**Некоторые преимущества использования метрик с метками включают:**

- **Увеличенная детализация**: Вы можете присоединить несколько меток к метрике, позволяя вам разрезать и анализировать данные различными способами.
- **Лучшая организация**: Метки могут помочь вам группировать и организовывать ваши метрики, упрощая поиск и понимание данных, которые вы ищете.
- **Упрощенные запросы**: Вы можете использовать метки для фильтрации и агрегации данных ваших метрик, упрощая извлечение значимых инсайтов из данных.

:::: tabs
::: tab RoadRunner
Используя файл `.rr.yaml`:

```yaml .rr.yaml
metrics:
  address: 127.0.0.1:2112

  collect:
    registered_users:
      type: histogram
      help: "Total registered users counter."
      labels: [ "type" ]
```

:::

::: tab PHP
Объявление метрик в PHP коде

```php app/src/Application/Bootloader/MetricsBootloader.php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;

class MetricsBootloader extends Bootloader
{
    //...

    public function boot(MetricsInterface $metrics): void
    {
        $metrics->declare(
            'registered_users',
            Collector::counter()->withHelp('Total registered users counter.')->withLabels('type')
        );
    }
}
```

:::

::::

В примере метрика `registered_users` объявлена с меткой под названием `type`. При добавлении данных к метрике вы можете указать значение для метки type, такое как `customer`, `admin` и т.д. Это позволяет различать разные типы пользователей при анализе данных метрик.

```php
use Spiral\RoadRunner\Metrics\MetricsInterface; 

class UserRegistrationHandler
{
    public function __construct(
        private readonly MetricsInterface $metrics
    ) {
    }

    public function handle(User $user): void
    {
        // Сохранить пользователя в базе данных

        $this->metrics->add('registered_users', 1, ['customer']);
        
        // или
        
        $this->metrics->add('registered_users', 1, ['admin']);
    }
}
```

## Декораторы

### Повторные попытки отправки метрик

Иногда вы можете столкнуться с проблемами при отправке метрик в плагин метрик RoadRunner. Соединение может оборваться или плагин может быть временно недоступен. Для таких сценариев вы можете использовать декоратор повторных попыток. Он позволяет указать, сколько раз и как часто вы хотите повторять попытки.

Вот пример использования декоратора повторных попыток:

```php
$factory = new \Spiral\RoadRunner\Metrics\MetricsFactory();
$rpc = $container->get(\Spiral\Goridge\RPC\RPCInterface::class);

$metrics = $factory->create($rpc, new \Spiral\RoadRunner\Metrics\MetricsOptions(
    retryAttempts: 3,
    retrySleepMicroseconds: 50,
));
```

### Подавление исключений

Иногда вы можете не хотеть, чтобы ошибки останавливали ваше приложение при отправке метрик. Например, если вы обрабатываете платежи, ошибка метрики не должна нарушать транзакцию. Вы можете подавить эти ошибки.

Вот пример подавления исключений:

```php
$factory = new \Spiral\RoadRunner\Metrics\MetricsFactory();
$rpc = $container->get(\Spiral\Goridge\RPC\RPCInterface::class);

$metrics = $factory->create(
    $rpc,
    new \Spiral\RoadRunner\Metrics\MetricsOptions(
        suppressExceptions: true,
    ),
));
```

Альтернативно, используйте декоратор `Spiral\RoadRunner\Metrics\SuppressExceptionsMetrics`:

```php
$metrics = new \Spiral\RoadRunner\Metrics\SuppressExceptionsMetrics(
    $container->get(\Spiral\RoadRunner\Metrics\MetricsInterface::class),
));
```

### Регистрация декораторов в контейнере

Вместо определения декораторов непосредственно в коде вашего приложения, лучше настроить их в контейнере.

Вот простое руководство:

```php app/src/Application/Bootloader/MetricsBootloader.php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;
use Spiral\Goridge\RPC\RPCInterface;

class MetricsBootloader extends Bootloader
{
    const SINGLETONS = [
        MetricsInterface::class => [self::class, 'createMetrics'],
    ];
    
    private function createMetrics(RPCInterface $rpc): MetricsInterface
    {
        $factory = new \Spiral\RoadRunner\Metrics\MetricsFactory();
        
        return $factory->create(
            $rpc,
            new \Spiral\RoadRunner\Metrics\MetricsOptions(
                suppressExceptions: true,
                retryAttempts: 3,
                retrySleepMicroseconds: 50,
            ),
        ));
    }
}
```

> **Примечание**
> Не забудьте зарегистрировать `MetricsBootloader` в ядре приложения.

---

## Пример приложения

Есть хороший пример приложения [**Demo ticket booking system**](https://github.com/spiral/ticket-booking), построенного на Spiral Framework, который следует принципам микросервисов и позволяет разработчикам создавать переиспользуемые, независимые и легкие в поддержке компоненты.

В этом демо-приложении вы можете найти пример использования плагина метрик RoadRunner.

В целом, наша демо-система бронирования билетов — отличный пример того, как Spiral и другие инструменты могут быть использованы для создания современного и эффективного приложения. Мы надеемся, что вы получите удовольствие от его использования и узнаете больше о возможностях нашего фреймворка и других инструментов, которые мы использовали.

**Счастливых (ненастоящих) покупок билетов!**
