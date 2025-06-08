# Начало работы — Первая CLI команда

Spiral предлагает удобный подход к созданию консольных приложений. Он поставляется со встроенной поддержкой консольных команд, позволяя разрабатывать интерфейсы командной строки (CLI) для вашего приложения. Консольные команды дают возможность автоматизировать задачи, выполнять операции обслуживания и взаимодействовать с приложением способами, выходящими за рамки стандартного веб-интерфейса.

Работа с консольными командами в Spiral невероятно проста. Фреймворк предоставляет удобный интерфейс, который использует мощь пакета `symfony/console`.

Давайте пройдем через основные шаги создания консольной команды.

## Создание команды

Чтобы легко создать первую команду, используйте команду скаффолдинга:

```terminal
php app.php create:command CurrentDate
```

> **Примечание**
> Подробнее о скаффолдинге читайте в разделе [Основы — Скаффолдинг](../basics/scaffolding.md#console-command).

После выполнения этой команды следующий вывод подтвердит успешное создание:

```output
Declaration of '[32mCurrentDateCommand[39m' has been successfully written into '[33mapp/src/Endpoint/Console/CurrentDateCommand.php[39m'.
```

Теперь давайте добавим логику в нашу свежесозданную команду.

Вот пример консольной команды, которая выводит текущую дату в консоль:

```php app/src/App/Endpoint/Console/CurrentDateCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'current:date')]
final class CurrentDateCommand extends Command
{
    #[Argument(description: 'Date format')]
    public string $format = 'Y-m-d';

    public function __invoke(): int
    {
        $this->writeln(\date($this->format));

        return self::SUCCESS;
    }
}
```

По умолчанию Spiral настроен для автоматического обнаружения команд, расположенных в директории `app/src`, через [компонент статического анализа](../advanced/tokenizer.md). Это означает, что вам не нужно вручную регистрировать команды или создавать отдельный файл конфигурации для них.

## Запуск команды

Чтобы получить справочную информацию для вашей команды, выполните следующую команду в терминале:

```terminal
php app.php help current:date
```

Это отобразит сигнатуру команды, описание и любые доступные аргументы или опции.

```output
[33mDescription:[39m
  Get current date

[33mUsage:[39m
  current:date [<format>]

[33mArguments:[39m
  [32mformat[39m                Date format[33m [default: "Y-m-d"][39m

[33mOptions:[39m
  [32m-h, --help[39m            Display help for the given command. When no command is given display help for the [32mlist[39m command
  [32m-q, --quiet[39m           Do not output any message
  [32m-V, --version[39m         Display this application version
  [32m    --ansi|--no-ansi[39m  Force (or disable --no-ansi) ANSI output
  [32m-n, --no-interaction[39m  Do not ask any interactive question
  [32m-v|vv|vvv, --verbose[39m  Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

<br>

**Вот и все! Вы успешно настроили свою первую консольную команду в Spiral.**

<hr>

## Что дальше?

Теперь углубитесь в основы, прочитав следующие статьи:

* [Конфигурация CLI](../console/configuration.md)
* [Создание команды](../console/commands.md)
* [Перехватчики (Interceptors)](../console/interceptors.md)
* [Валидация ввода команды](../cookbook/console-validation.md)
* [Скаффолдинг](../basics/scaffolding.md)