# Custom Commands
You can add any custom commands to your own application or even module by simply making sure that command class can be loaded (i.e. located in tokenizer directories) and then execute`console:reload` to find and index your command.

> You can also add command manually into your console config.

To generate blank command execute `create:command name` (spiral/scaffolder module is required), as result following class will be created for us (based on scaffolder settings):

```php
class SomeCommand extends Command
{
    const NAME        = 'some';
    const DESCRIPTION = '';

    const ATTRIBUTES = [];
    const OPTIONS = [];

    /**
     * Perform command
     */
    protected function perform()
    {
    }
}
```

> You still able to use native Symfony commands.

Spiral `Command` class makes easy to define needed arguments and options:

```php
const ARGUMENTS = [
    ['argument', InputArgument::REQUIRED, 'Argument name.']
];
    
const OPTIONS = [
    ['option', 'c', InputOption::VALUE_NONE, 'Some option.']
];
```

> Perform method are executed using Container and support method injection.

