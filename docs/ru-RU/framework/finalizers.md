# Framework — Финализаторы (Finalizer)

Большинство компонентов фреймворка не требуют сброса после завершения запроса. Однако существует множество случаев использования, когда вам может потребоваться сбросить вашу библиотеку после завершения пользовательского запроса.

> **Примечание**
> Отдавайте приоритет использованию областей видимости IoC над финализаторами.

## FinalizerInterface

Используйте `Spiral\Boot\FinalizerInterface`

```php
/**
 * Используется для закрытия ресурсов и соединений для долго работающих процессов.
 */
interface FinalizerInterface
{
    /**
     * Финализаторы выполняются после каждого запроса и используются для сборки мусора
     * или для закрытия открытых соединений.
     *
     * @param callable $finalizer
     */
    public function addFinalizer(callable $finalizer);
    
    /**
     * Финализировать выполнение.
     *
     * @param bool $terminate Установите в true, если финализация вызвана при завершении приложения.
     */
    public function finalize(bool $terminate = false);
}
```

Все диспетчеры приложения будут вызывать финализатор. Во время:

* завершения HTTP запроса
* сбоя HTTP запроса с ошибкой
* завершения задачи
* сбоя задачи с ошибкой
* завершения GRPC вызова
* сбоя GRPC вызова с ошибкой
* завершения консольной команды

> **Предупреждение**
> Финализатор будет вызван только если конкретный диспетчер был запущен. Вы можете свободно вызывать команды приложения и HTTP методы без использования диспетчера напрямую и без сброса ваших сервисов после каждого запроса.

Ваш обработчик получит первый bool аргумент, который указывает, собирается ли приложение завершиться после запроса.

> **Примечание**
> Избегайте сброса настроек IoC в финализаторе, так как это может привести к тому, что некоторые singleton сервисы закешируют предыдущую версию сервиса.

## Пример финализатора

Мы можем использовать финализатор, чтобы продемонстрировать, как автоматически закрывать соединение с базой данных после каждого запроса. Это может быть полезно, если вы запускаете много работников (или lambda функций) и не хотите потреблять все сокеты базы данных.

```php
// в загрузчике
use Spiral\Boot\FinalizerInterface;
use Psr\Container\ContainerInterface;
use Cycle\Database\DatabaseManager;

public function boot(FinalizerInterface $finalizer, ContainerInterface $container): void
{
    $finalizer->addFinalizer(function () use ($container) {
        /** @var DatabaseManager $dbal */
        $dbal = $container->get(DatabaseManager::class);
 
        foreach ($dbal->getDrivers() as $driver) {
            $driver->disconnect();
        }
    });
}
```

> **Примечание**
> Вы можете найти такой загрузчик уже включенным в пакет [`spiral\cycle-bridge`](https://github.com/spiral/cycle-bridge/blob/master/src/Bootloader/DisconnectsBootloader.php) и доступным как `Spiral\Cycle\Bootloader\DisconnectsBootloader`.
