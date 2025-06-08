# Консоль — Последовательности команд

Spiral предоставляет способ группировки и выполнения серии консольных команд или замыканий в определенном порядке. Эти группы команд и замыканий называются "последовательностями", и существует два типа: последовательности `configure` и `update`.

Использование консольных последовательностей может быть удобным способом автоматизации общих задач или операций в вашем приложении и может помочь обеспечить их выполнение последовательно и в правильном порядке.

## Регистрация команд

### Последовательности Configure

Последовательности Configure предназначены для использования в задачах, связанных с настройкой или конфигурированием приложения. Эти команды выполняются после вызова команды `php app.php configure`.

Вот пример последовательности configure:

```php
use Symfony\Component\Console\Output\OutputInterface;
use Psr\Container\ContainerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ConsoleBootloader $console): void
    {
        $console->addConfigureSequence(
            sequence: 'generate:keys', 
            header: '<info>Generating SSH keys for the application...</info>'
        );
        
        // Добавление замыкания в последовательность
        // Поддерживает автоматическое внедрение аргументов
        $console->addConfigureSequence(
            static function(OutputInterface $output, ContainerInterface $container): void {
                // выполнить что-то
            }, 
            '<info>Caching something...</info>'
        );
    }
}
```

### Последовательности Update

Последовательности Update предназначены для использования в задачах, связанных с обновлением или модификацией приложения. Эти команды выполняются после вызова команды `php app.php update`.

Вот пример последовательности update:

```php
use Symfony\Component\Console\Output\OutputInterface;
use Psr\Container\ContainerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ConsoleBootloader $console): void
    {
        $console->addUpdateSequence('db:migrate', '<info>Database migration...</info>');
        
        // Добавление замыкания в последовательность
        // Поддерживает автоматическое внедрение аргументов
        $console->addConfigureSequence(
            static function(OutputInterface $output, ContainerInterface $container): void {
                // выполнить что-то
            }, 
            '<info>Caching something...</info>'
        );
    }
}
```

## Пользовательские последовательности

В дополнение к этим предопределенным последовательностям, Spiral также позволяет создавать пользовательские последовательности. Пользовательские консольные последовательности — это группы консольных команд или замыканий, которые предназначены для выполнения в определенном порядке и создаются и управляются пользователем.

Вот пример последовательности update:

```php app/src/Application/Bootloader/AppBootloader.php
use Symfony\Component\Console\Output\OutputInterface;
use Psr\Container\ContainerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ConsoleBootloader $console): void
    {
        $console->addSequence(
            name: 'cache_everything', 
            sequence: 'route:cache',
            header: '<info>Route caching...</info>',
            footer: '<info>Route caching completed.</info>'
        );
         
        $console->addSequence(
            name: 'cache_everything', 
            sequence: 'config:cache',
            header: '<info>Config caching...</info>',
            footer: '<info>Config caching completed.</info>'
        );
        
        // ... 
    }
}
```

Пользовательские последовательности могут выполняться из консольной команды, которая расширяет класс `Spiral\Console\Sequence\SequenceCommand`.

Вот пример команды пользовательской последовательности:

```php
namespace App\Command;

use Psr\Container\ContainerInterface;
use Spiral\Console\Config\ConsoleConfig;

final class CacheEverythingCommand extends SequenceCommand
{
    protected const NAME = 'cache:everything';
    protected const DESCRIPTION = 'Cache everything in the project';

    public function perform(ConsoleConfig $config, ContainerInterface $container): int
    {
        $this->info('Caching everything in the project...');
        $this->newLine();

        return $this->runSequence($config->getSequence('cache_everything'), $container);
    }
}
```

Пользовательские последовательности могут быть полезным инструментом для автоматизации задач или операций в вашем приложении и могут помочь упростить процессы разработки и обслуживания.

## Выполнение последовательности

После того как вы добавили свои команды и замыкания в последовательность, вы можете выполнить последовательность, используя команду `php app.php configure` для последовательности configure или команду `php app.php update` для последовательности update. Консоль будет отображать описание каждой команды или замыкания по мере их выполнения, а также любой вывод, генерируемый командой или замыканием.
