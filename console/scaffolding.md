# Custom Commands
You can add your custom commands to your own application or even module by simply making sure that command class can be loaded and executing command `console:refresh` to find and index your command.

To generate blank command we can execute `create:command name`, as result we will get something like that:

```php
class SomeCommand extends Command
{
    /**
     * @var string
     */
    protected $name = 'some';
        
    /**
     * @var string
     */
    protected $description = '';
        
    /**
     * @var string
     */
    protected $arguments = [
        
    ];
        
    /**
     * @var string
     */
    protected $options = [
        
    ];
        
    /**
     * Perform command.
     */
    public function perform()
    {
    }
}
```

You can put your command arguments and options into $arguments and $options proterties accordingly. Defintion format is identical to symfony console:

```php
protected $arguments = [
    ['argument', InputArgument::REQUIRED, 'Argument name.']
];
    
protected $options = [
    ['option', 'c', InputOption::VALUE_NONE, 'Some option.']
];
```

> Perform method are executed using Container and support method injection.