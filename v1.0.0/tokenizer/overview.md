# Tokenizer / Static Analysis
The Spiral Framework ships with convenient way to automatically analyze your application in order to pre-compile some of it's functions.

> Tokenizer extension is currently based on legacy way of code analysis - php token which limits it's functionality in some aspects. Feel free to contribute to implement AST based reflections.

## Configuration
In order to make tokenizer locate classes and method/function invocation we have to register set of directories to be iterated, such operation can be done inside tokenizer config:

```php
return [
    /*
     * Tokenizer will be performing class and invocation lookup in a following directories. Less
     * directories - faster Tokenizer will work.
     */
    'directories' => [
        directory('application'),

        //Default set of spiral console commands
        directory('framework') . 'Commands/',

        //Needed to allow Translator locate i18n validation messages
        directory('framework') . 'Validation/',

        //External modules
        directory('libraries') . 'spiral/ide-helper/source/',
        directory('libraries') . 'spiral/scaffolder/source/',
        /*{{directories}}*/
    ],
    /*
     * Such paths are excluded from tokenization. You can use format compatible with Symfony Finder.
     */
    'exclude'     => [
        'config',
        'resources',
        'migrations',
        /*{{exclude}}*/
    ]
];
```

Directories and exclude patters can be specified in Symfony Finder format.

## Performance Considerations
Please note that code analysis is very slow operation, do not execute it based on user requests but rather move it into console command.

Following components use Tokenizer:
- ORM - automatic location of Record, RecordEntity and Repository models
- ODM - automatic location of Document and DocumentEntity and Repository models
- Console - automatic location of available commands
- Translator - automatic location of used i18n strings
