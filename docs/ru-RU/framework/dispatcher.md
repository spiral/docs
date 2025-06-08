# Framework — Диспетчеры

Одной из ключевых особенностей Spiral является поддержка множественных диспетчеров ядра, которые отвечают за маршрутизацию входящих запросов к соответствующему обработчику на основе текущего окружения.

Давайте представим, что у нас есть два диспетчера: `console` и `http`.

Вот пример `http` диспетчера:

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\DispatcherInterface;

final class HttpDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {
    }
    
    public function canServe(): bool
    {
        return $this->env->get('RR_MODE') === 'http';
    }
    
    public function serve(): void
    {
        // Обработка HTTP запросов
    }
}
```

И пример `console` диспетчера:

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\DispatcherInterface;

final class ConsoleDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {
    }
    
    public function canServe(): bool
    { 
        return (PHP_SAPI === 'cli' && $this->env->get('RR_MODE') === null);
    }
    
    public function serve(InputInterface $input = null, OutputInterface $output = null): int
    {
        // Обработка консольных команд
    }
}
```

Теперь мы можем зарегистрировать эти диспетчеры в нашем приложении:

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

class AppBootloader extends Bootloader
{
    public function boot(
      KernelInterface $kernel, 
      HttpDispatcher $http,
      ConsoleDispatcher $console,
    ): void  {
        $kernel->addDispatcher($http)
        $kernel->addDispatcher($console);
    }
}
```

И точка входа `app.php` нашего приложения будет выглядеть так:

```php app.php
use App\Application\Kernel;

\mb_internal_encoding('UTF-8');
\error_reporting(E_ALL | E_STRICT ^ E_DEPRECATED);
\ini_set('display_errors', 'stderr');

require __DIR__ . '/vendor/autoload.php';
$app = Kernel::create(
    directories: ['root' => __DIR__],
)->run();

$code = (int)$app->serve();  // <========== Запустит соответствующий диспетчер на основе текущего окружения
exit($code);
```

Когда мы запускаем наше приложение, соответствующий диспетчер будет выбран на основе текущего окружения. Например, если мы выполним следующую команду:

```terminal
php app.php db:migrate
```

Фреймворк переберет список зарегистрированных диспетчеров и вызовет метод `canServe` для каждого диспетчера. Этот метод является способом для диспетчера сообщить фреймворку, может ли он обработать запрос на основе текущего окружения. Фреймворк будет использовать первый диспетчер, который вернет `true`.

`ConsoleDispatcher` вернет `true`, если текущее окружение является `cli`. Когда RoadRunner запускает HTTP плагин, плагин запустит рабочий процесс и передаст переменную окружения `RR_MODE=http` работнику. В этом случае будет выбран `HttpDispatcher`.

Если ни один диспетчер не вернет `true`, фреймворк выбросит исключение.

## Доступные диспетчеры

Spiral поставляется с несколькими встроенными диспетчерами:

- [Console dispatcher](https://github.com/spiral/framework/blob/master/src/Framework/Console/ConsoleDispatcher.php): отвечает за обработку консольных команд в вашем приложении. Это полезно, если вы хотите создавать пользовательские команды, которые можно запускать из командной строки.

- [RoadRunner HTTP dispatcher](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/Http/Internal/Dispatcher.php): отвечает за обработку входящих HTTP запросов и маршрутизацию их к соответствующему действию контроллера или функции. Это диспетчер, который используется, когда ваше приложение работает как HTTP сервис.

- [RoadRunner GRPC dispatcher](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/GRPC/Internal/Dispatcher.php): отвечает за обработку входящих GRPC запросов и маршрутизацию их к соответствующим сервисам. Это диспетчер, который используется, когда ваше приложение работает как GRPC сервис.

- [RoadRunner TCP dispatcher](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/Tcp/Internal/Dispatcher.php): отвечает за обработку входящих TCP соединений и маршрутизацию их к соответствующему обработчику. Это диспетчер, который используется, когда ваше приложение работает как TCP сервис.

- [Temporal dispatcher](https://github.com/spiral/temporal-bridge/blob/2.0/src/Dispatcher.php): отвечает за обработку входящих активностей рабочих процессов. Temporal - это распределенный, масштабируемый и отказоустойчивый движок рабочих процессов, который используется для построения и оркестрации долго выполняющейся бизнес-логики.

- [RoadRunner Queue dispatcher](https://github.com/spiral/roadrunner-bridge/blob/4.x/src/Queue/Internal/Dispatcher.php): позволяет вам потреблять сообщения из очереди и маршрутизировать их к соответствующему обработчику. Это полезно, если вы хотите построить систему, основанную на архитектуре, управляемой сообщениями. Это диспетчер, который используется, когда ваше приложение работает как сервис потребителя очереди.

> **Примечание**
> Прочитайте, как создать пользовательский диспетчер [здесь](../cookbook/custom-dispatcher.md)

Диспетчеры ядра Spiral предоставляют гибкий и мощный способ маршрутизации входящих запросов к соответствующему обработчику. Они являются важным компонентом и важной частью общего дизайна фреймворка.
