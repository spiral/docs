# Scaffolding
Spiral framework includes set of console commands intented to simplify development by providing quick scaffolding mechanisms for the most common parts of application.

## Command
Blank console command can be created using spiral command `create:command name`. Generated class will be located in application/classes/Commands directory under name "{name}Command" and namespace "Commands".

Example: `create:command test`

```php
<?php
/**
 * {project-name}
 * 
 * @author    {author-name}
 */
namespace Commands;
use Spiral\Console\Command;

class TestCommand extends Command 
{
    /**
     * @var string
     */
    protected $name = 'test';

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

You are able to edit command name, description, arguments and options by modifying class properties. To register command in your application simply run `console:refresh`.

> Command method `perform` support dependency injection which makes you able to request needed instances of services and components.

## ORM Record

## ODM Document

#### DocumentEntity or Complex Type

## Service

## Service with entity repository

## Request 

#### Request with entity fields

## Middleware

## Controller

#### Controllers with Service

##### Controller with Service and RequestFilter

## Migration

## Blank View
