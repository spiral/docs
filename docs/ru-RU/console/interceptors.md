# Консоль — Перехватчики (Interceptor)

Перехватчики (Interceptor) позволяют перехватывать выполнение консольной команды и выполнять некоторую логику до или после выполнения команды.

## Создание перехватчика (Interceptor)

Для создания перехватчика вам нужно создать класс и реализовать интерфейс `Spiral\Core\CoreInterceptorInterface`.

```php
namespace App;

use Spiral\Console\Command;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class CustomInterceptor implements CoreInterceptorInterface
{

    /**
     * @param array{
     *     input: InputInterface, 
     *     output: OutputInterface, 
     *     command: Command
     * }|array<empty, empty> $parameters
     */
    public function process(
        string $commandClass, 
        string $method, 
        array $parameters, CoreInterface $core
    ): int {
        // ...

        $result = $core->callAction($commandClass, $method, $parameters);

        // ...

        return $result;
    }
}
```

## Регистрация нового перехватчика (Interceptor)

Перехватчик должен быть зарегистрирован в приложении для правильной работы. Существует несколько способов добавления нового перехватчика.

### Через конфигурацию

Добавьте его в файл конфигурации `app/config/console.php`.

```php
use App\CustomInterceptor;
use Spiral\Core\Container\Autowire;

return [    
    /**
     * -------------------------------------------------------------------------
     *  Список всех перехватчиков
     * -------------------------------------------------------------------------
     */
    'interceptors' => [
        // через полное имя класса
        CustomInterceptor::class,
        
        // через Autowire
        new Autowire(CustomInterceptor::class),
    ],
];
```

### Через ConsoleBootloader

Вызовите метод `addInterceptor` в классе `Spiral\Console\Bootloader\ConsoleBootloader`.

```php
namespace App\Application\Bootloader;

use App\CustomInterceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ConsoleBootloader $console): void
    {
        $console->addInterceptor(CustomInterceptor::class);
    }
}
```
