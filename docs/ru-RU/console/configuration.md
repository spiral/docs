# Console — Начало работы

Компонент Console упрощает создание и управление консольными командами в вашем приложении. Используя мощь 
пакета symfony/console, компонент Console предоставляет удобный интерфейс для работы с консольными командами.

Все предоставляемые скелеты приложений включают компонент Console по умолчанию. Чтобы включить компонент в
альтернативных сборках, убедитесь, что требуется composer пакет `spiral/console` и измените загрузчик (bootloader) приложения:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\CommandBootloader::class,
        // ...
    ];
}
```

Подробнее о загрузчиках (bootloaders) читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\CommandBootloader::class,
    // ...
];
```

Подробнее о загрузчиках (bootloaders) читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

Чтобы вызвать команду приложения, просто выполните:

```terminal
php app.php command:name
```

Чтобы получить список доступных команд:

```terminal
php app.php list
```

Чтобы получить справку по конкретной команде:

```terminal
php app.php help command:name
```

## Вызов в приложении

Возможно вызывать консольные команды внутри вашего приложения или тестов приложения. Это может быть полезным подходом
для создания тестовых данных для тестов, автоматической предварительной настройки базы данных или выполнения других задач, требующих
использования консольных команд.

Чтобы вызвать консольную команду из вашего приложения или тестов, вы можете использовать сервис `Spiral\Console\Console`.
Этот сервис предоставляет метод `run()`, который позволяет выполнить консольную команду по её имени с аргументами.

Вот пример того, как вы можете использовать его для вызова консольной команды внутри вашего приложения:

```php
use Spiral\Console\Console;
use Symfony\Component\Console\Input\ArrayInput;
use Symfony\Component\Console\Output\BufferedOutput;

// ...

public function test(Console $console): string
{
    $input = new ArrayInput([
        '--mount' => '.env',
        '-p' => '{encrypt-key}'
    ]);
    
    $output = new BufferedOutput();
    
    return $console->run('encrypt:key', $input, $output)->fetch();
}
```

## Symfony/Console

Диспетчер Spiral Console построен на основе
мощного компонента [Symfony Console](http://symfony.com/doc/current/components/console/introduction.html).

> **Примечание**
> Вы можете регистрировать нативные команды Symfony в вашем CLI приложении.

## Конфигурация

Чтобы применить пользовательскую конфигурацию к компоненту Console, используйте `Spiral\Config\ConfiguratorInterface` или создайте файл конфигурации
в `app/config/console.php`:

```php
return [
     // название приложения
     'name'      => null,
     
     // версия приложения
     'version'   => null,
     
     // список команд приложения (если автоматическое обнаружение отключено)
     'commands'  => [],
     
     // список команд и последовательностей для выполнения в `app configure`
     'configure' => [],
     
     // список команд и последовательностей для выполнения в `app update`
     'update'    => []
];
```

Вы можете изменить некоторые из этих значений во время загрузки приложения через `Spiral\Bootloader\ConsoleBootloader`. Чтобы зарегистрировать
новую пользовательскую команду:

```php
public function boot(ConsoleBootloader $console): void
{
    $console->addCommand(MyCommand::class);
}
```

> **Примечание**
> Компонент по умолчанию настроен на автоматическое обнаружение команд, расположенных в директории `app/src`.

## Связь с RoadRunner

**Обратите внимание, что консольные команды вызываются вне сервера RoadRunner. Убедитесь, что запущен экземпляр сервера приложения,
если любая из ваших команд должна связываться с ним.**
