# Консоль — Пользовательские команды

Вы можете добавлять новые консольные команды в ваше приложение или регистрировать команды плагинов с помощью загрузчиков (Bootloader). Компонент настроен по умолчанию для автоматического обнаружения команд, расположенных в директории `app/src`.

Эта страница документации проведет вас через процесс создания и использования консольных команд в вашем приложении. Независимо от того, являетесь ли вы новичком или опытным разработчиком, вы найдете необходимую информацию для начала работы.

## Класс команды

Для создания новой команды вы можете расширить либо `Symfony\Component\Console\Command\Command`, либо `Spiral\Console\Command`. Расширение класса `Spiral\Console\Command` предоставляет дополнительный синтаксический сахар и удобные методы, которые могут облегчить создание вашей команды.

> **Примечание**
> Какой вариант вы выберете, будет зависеть от ваших потребностей и предпочтений. Оба подхода являются действительными и могут использоваться для создания функциональных консольных команд.

Для легкого создания вашей первой команды используйте команду генерации:

```terminal
php app.php create:command My
```

> **Примечание**
> Подробнее о генерации кода читайте в разделе [Основы — Генерация кода](../basics/scaffolding.md#console-command).

После выполнения этой команды следующий вывод подтвердит успешное создание:

```output
Declaration of '[32mMyCommand[39m' has been successfully written into '[33mapp/src/Endpoint/Console/MyCommand.php[39m'.
```

```php app/src/App/Endpoint/Console/MyCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'my')]
final class MyCommand extends Command
{
    public function __invoke(): int
    {
        // Разместите логику вашей команды здесь
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

Для вызова вашей команды выполните следующую команду в консоли:

```terminal
php app.php my
```

## Атрибуты (Attribute)

Начиная с версии 3.6, Spiral предлагает возможность определения консольных команд с использованием PHP атрибутов. Это позволяет использовать более интуитивный и упорядоченный подход к определению команд с четким разделением задач.

Вот пример определения консольной команды с использованием атрибутов:

```php
namespace App\Api\Cli\Command;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;
use Symfony\Component\Console\Input\InputOption;

#[AsCommand(
    name: 'app:create:user', 
    description: 'Creates a user with the given data')
]
final class CreateUser extends Command
{
    #[Argument]
    private string $email;

    #[Argument(description: 'User password')]
    private string $password;

    #[Argument(name: 'username', description: 'The user name')]
    private string $userName;

    #[Option(shortcut: 'a', name: 'admin', description: 'Set the user as admin')]
    private bool $isAdmin = false;

    public function __invoke(): int
    {
        $user = new User(
            email: $this->email,
            password: $this->password,
        );
        
        $user->setIsAdmin($this->isAdmin);
        
        // Сохранение пользователя в базу данных...

        return self::SUCCESS;
    }
}
```

Для определения имени и описания консольной команды вы можете использовать либо атрибут `Spiral\Console\Attribute\AsCommand`, либо `Symfony\Component\Console\Attribute\AsCommand`.

### Аргументы

Вы можете пометить свойство класса как аргумент для консольной команды, используя атрибут `Spiral\Console\Attribute\Argument`.

```php
#[Argument]
private string $email;
```

Атрибут `Argument` без дополнительных параметров, указывающий, что он будет использовать имя свойства в качестве имени аргумента и не будет включать описание.

```php
#[Argument(description: 'User password')]
private string $password;
```

Используйте параметр description для предоставления краткого описания аргумента.

```php
#[Argument(name: 'username', description: 'The user name')]
private string $username;
```

Параметр name позволяет вам указать пользовательское имя аргумента, отличающееся от имени свойства.

> **Примечание**
> Любое свойство, помеченное атрибутом Argument, должно иметь скалярную типизацию, например `string`, чтобы указать тип значения, который должен принимать аргумент.

Если вы хотите установить значение по умолчанию для аргумента, вы можете указать значение по умолчанию как значение свойства по умолчанию:

```php
#[Argument]
private string $username = 'guest';
```

В этом случае, если значение для аргумента не предоставлено во время вызова команды, будет использовано значение по умолчанию.

В некоторых случаях вы можете захотеть сделать аргумент необязательным в консольной команде. Для этого вы можете использовать nullable типизацию для свойства, представляющего аргумент, и указать значение по умолчанию как `null`.

```php
#[Argument]
private ?string $username = null;
```

### Опции

Опции — это дополнительные параметры, которые пользователи могут указать при запуске команды. Они определяются как свойства в вашем классе команды, аннотированные атрибутом #[\Spiral\Console\Attribute\Option].

```php
use Spiral\Console\Attribute\Option;

#[Option(name: 'admin', description: 'Set the user as admin')]
private bool $isAdmin = false;
```

Параметр `name` позволяет вам указать пользовательское имя опции, отличающееся от имени свойства.

```terminal
php app.php app:create:user --admin
```

Вы также можете указать сокращение для опции, используя параметр shortcut:

```php
#[Option(shortcut: 'a', ...)]
private bool $isAdmin = false;
```

Эта опция может быть вызвана с сокращением `-a` вместо ввода `--admin`.

Опции могут быть определены как обязательные или необязательные. Чтобы сделать опцию обязательной, вам не нужно указывать значение по умолчанию для свойства:

```php
#[Option(...)]
private string $status;
```

Если вы хотите сделать опцию необязательной, вы можете указать значение по умолчанию как значение свойства по умолчанию:

```php
#[Option(...)]
private ?string $status = null;
```

Используя PHP enum, значение опции будет валидироваться против значений enum.

```php
#[Option(description: 'Set the user status')]
private Status $status = Status::new;
```

Для принятия множественных значений для опции вы можете использовать типизацию `array` для свойства, представляющего опцию:

```php
#[Option(name: 'role', description: 'Set the user roles')]
private array $role = [];
```

```terminal
php app.php app:create:user --role=foo --role=bar
```

### Вопросы

По умолчанию, когда вы вызываете консольную команду без передачи обязательных аргументов, приложение попросит вас предоставить значение для каждого отсутствующего аргумента. Однако вы можете настроить сообщение запроса для каждого аргумента, используя атрибут `Spiral\Console\Attribute\Question`.

Он может использоваться как атрибут свойства, так и как атрибут класса.

```php
#[Option(
    mode: \Symfony\Component\Console\Input\InputOption::VALUE_REQUIRED, 
    ...
)]
private bool $isAdmin = false;
```

При использовании в качестве атрибута класса вам нужно определить соответствующее имя `argument` для каждого свойства, которое требует пользовательского запроса.

```php
#[Question(question: 'Provide user email', argument: 'email')]
final class CreateUserCommand extends Command
{
    #[Argument]
    private string $email;

    // ...
}
```

Устанавливая пользовательский запрос для каждого аргумента, вы можете предоставить более конкретную и релевантную информацию пользователю, что может сделать взаимодействие более интуитивным и упорядоченным. Хорошей практикой является использование запросов, которые являются ясными, краткими и простыми для понимания.

## Сигнатура

Константа `SIGNATURE` предоставляет альтернативный способ определения **имени** вашей команды, а также её **аргументов** и **опций**. Это может быть удобным способом указать всю эту информацию в одном месте, вместо отдельного определения имени, аргументов и опций.

```php
class SomeCommand extends Command 
{
    protected const SIGNATURE = <<<CMD
        check:http 
            {url : Site url} 
            {--S|skip-ssl-errors : Skip SSL errors}
        CMD;


    public function perform(): int
    {
        $url = $this->argument('url');
        $skipErrors = $this->option('skip-ssl-errors');

        // ...
    }
}
```

В этом примере константа `SIGNATURE` определяет команду с именем `check:http` с обязательным аргументом `url` и необязательной опцией `skip-ssl-errors`.

### Аргументы

Вы можете сделать аргумент необязательным, включив символ `?` после его имени. Например, `check:http {url?}` определяет необязательный аргумент.

Вы также можете определить значение по умолчанию для аргумента, включив символ `=`, за которым следует значение по умолчанию после имени аргумента. Например, `check:http {url=foo}` определяет аргумент со значением по умолчанию.

Если вы хотите определить **аргумент**, который ожидает множественные входные значения, вы можете использовать символ `[]`. Например, `check:colors {colors[]}` определяет аргумент, который ожидает множественные значения.

Если вы хотите сделать аргумент необязательным, вы можете включить символ `?` после символов `[]`, как в `check:colors {colors[]?}`. Это позволит пользователю опустить аргумент при желании.

### Опции

Опции полезны для указания дополнительной информации или изменения поведения команды. Они могут использоваться для включения или отключения определенных функций, указания файла конфигурации или другого входного файла, или установки других параметров, влияющих на поведение команды.

Опции — это форма пользовательского ввода, которая предваряется двумя дефисами `--` при указании в командной строке. В константе `SIGNATURE` опции определяются с использованием синтаксиса `{--name}`, `{--name=value}` или `{--n|name}`, если вы хотите использовать сокращение.

Вы можете использовать сокращения для опций, чтобы облегчить пользователям указание опции при вызове команды.

Например, `{--S|skip-ssl-errors}` определяет опцию с именем `skip-ssl-errors` с сокращением `S`. Это означает, что опция может быть указана с использованием либо `--skip-ssl-errors`, либо `--S` в командной строке.

Использование сокращений для опций может сделать указание опций, которые пользователи хотят использовать при вызове вашей команды, более легким и удобным. Хорошая идея — выбирать ясные и краткие имена сокращений, которые легко запомнить и набрать.

Если вы хотите определить **опцию**, которая ожидает множественные входные значения, вы можете использовать символ `[]`. Например, `check:colors {--colors[]=}` определяет опцию, которая ожидает множественные значения.

### Описание

Хорошая идея — включить описание для каждого `argument` и `option`, которые вы определяете, поскольку это поможет пользователям понять назначение и формат ввода, ожидаемого вашей командой. Это может облегчить пользователям эффективное использование вашей команды и избежание ошибок.

Вот пример того, как вы можете использовать аргумент с описанием в консольной команде:

```php
check:http 
    {url : Site url} 
    {--S|skip-ssl-errors : Skip SSL errors}
```

## Метод perform

Вы можете поместить ваш пользовательский код в метод `perform`. Метод `perform` поддерживает внедрение методов и предоставляет свойства `$this->input` и `$this->output` для работы с пользовательским вводом.

```php
protected function perform(MyService $service): int
{
    $this->output->writeln($service->doSomething());
    
    return self::SUCCESS;
}
```

Для получения данных пользователя через аргументы и/или опции вы можете использовать `$this->argument("argName")` или `$this->option("optName")`.

> **Примечание**
> Дополнительно [здесь](/cookbook/console-validation.md) вы можете найти информацию о том, как использовать компонент `spiral/filters` для консольных команд.

## Вспомогательные методы

Вы можете использовать набор вспомогательных методов, доступных внутри `Spiral\Console\Command`. Приведенные примеры предназначены для вызова в методе `perform`.

Для записи в вывод:

```php
$this->writeln('hello world');
```

Для записи в вывод без перехода на новую строку:

```php
$this->write('hello world');
```

Для записи форматированного вывода без перехода на новую строку:

```php
$this->sprintf('Hello, <comment>%s</comment>', $name);
```

> **Примечание**
> Этот метод совместим с определением `sprintf`.

Для проверки, выше ли текущий режим подробности, чем `OutputInterface::VERBOSITY_VERBOSE`:

```php
dump($this->isVerbose());
```

> **Примечание**
> Вы можете свободно использовать метод `dump` в консольных командах.

Для отображения таблицы:

```php
$table = $this->table([
    'Column #1:',
    'Column #2:',
]);

foreach ($data as $row)
{
    $table->addRow([
        $row[1],
        $row[2]
    ]);
}

$table->render();
```

Для добавления разделителя таблицы:

```php
$table->addRow(new Symfony\Component\Console\Helper\TableSeparator());
```

Для определения наличия опции ввода:

```php
$this->hasOption(...);
```

Для определения наличия аргумента ввода:

```php
$this->hasArgument(...);
```

Для запроса подтверждения:

```php
$status = $this->confirm('Are you sure?', default: false);
```

Для задания вопроса:

```php
$status = $this->ask('Are you sure?', default: 'no');
```

Для задания вопроса с множественным выбором:

```php
$name = $this->choiceQuestion(
    'Which of the following is package manager?',
    ['composer', 'django', 'phoenix', 'maven', 'symfony'],
    default: 2,
    allowMultipleSelections: true
);
```

Для запроса у пользователя ввода с сокрытием ответа из консоли:

```php
$status = $this->secret('User password');
```

Для записи сообщения как информационный вывод:

```php
$this->info('Some message');
```

Для записи сообщения как комментарий:

```php
$this->comment('Some message');
```

Для записи сообщения как вопрос:

```php
$this->question('Some question');
```

Для записи сообщения как ошибка:

```php
$this->error('Some error');
```

Для записи сообщения как предупреждение:

```php
$this->warning('Some warning');
```

Для записи сообщения как предупреждение:

```php
$this->alert('Some alert');
```

Для записи пустой строки:

```php
$this->newLine();
$this->newLine(count: 5);
```

## ApplicationInProduction

Класс `Spiral\Console\Confirmation\ApplicationInProduction`, предоставляемый компонентом, упрощает запрос подтверждения у пользователя перед выполнением команды, если приложение работает в производственном режиме. Это может помочь предотвратить случайные или непреднамеренные изменения в производственной среде.

Для его использования вы можете внедрить его экземпляр в метод `perform()` вашей команды. Затем вы можете использовать метод `confirmToProceed()` для запроса подтверждения у пользователя перед продолжением выполнения команды.

```php
use Spiral\Console\Confirmation\ApplicationInProduction;

final class MigrateCommand extends Command
{
    protected const NAME = 'db:migrate';

    public function perform(ApplicationInProduction $confirmation): int
    {
        if (!$confirmation->confirmToProceed()) {
            return self::FAILURE;
        }
        
        // выполнение миграций...
    }
}
```

## Аргументы и опции

Класс Command упрощает определение аргументов и опций для вашей консольной команды. Установив константы `ARGUMENTS` и `OPTIONS`:

```php
const ARGUMENTS = [
    ['argName', InputArgument::REQUIRED, 'Argument name.']
];
    
const OPTIONS = [
    ['optName', 'c', InputOption::VALUE_NONE, 'Some option.']
];
```

## События

| Событие                              | Описание                                                       |
|--------------------------------------|----------------------------------------------------------------|
| Spiral\Console\Event\CommandStarting | Событие будет запущено `перед` выполнением консольной команды. |
| Spiral\Console\Event\CommandFinished | Событие будет запущено `после` выполнения консольной команды.  |

> **Примечание**
> Чтобы узнать больше о диспетчеризации событий, смотрите раздел [События](../advanced/events.md) в нашей документации.
