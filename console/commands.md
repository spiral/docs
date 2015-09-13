# Console Dispatcher and Embedded commands
Spiral Console dispatcher built at top of powerful [Symfony Console](http://symfony.com/doc/current/components/console/introduction.html). You can controller work with 
any symfony compatible command. Spiral will automatically index every available command in your application and make it accessible with CLI interface.

## Running CLI
To get access to spiral CLI you must execute `php webroot/index.php {command}` from your root folder. There is two file shortcuts for Windows and Unix systems which will
simplify that command to `spiral {command}` or `./spiral {command}`.

> Do not forget to give executable permission to 'spiral' file.

## Execute commands outside of CLI environment
As in case with HttpDispatcher you can get access to ConsoleDispather and it's commands at any moment, let's try to execute 'help' command from our controller:

```php
public function index()
{
    dump($this->console->command('help'));
}
```

You can provide custom Input and Output interfaces into command method or even specify parameters via array:

```php
  dump($this->console->command('command', [
    'name' => 'Some value' 
  ]));
```

## Embedded Commands
Spiral provides big set of commands you might use if your application:

| Command            | Description                                                              |
| ---                | ---                                                                      |
| configure          | Configure file permissions, install modules and render view files.       |    
| inspect            | Inspect ORM and ODM models to locate unprotected field and rules.        |
| migrate            | Perform one or all outstanding migrations.                               |
| modules            | Get list of all available and installed modules.                         |
| server             | Run Spiral Development server on specified host and port.                |
| update             | Perform application schemas and cache update.                            |
| console:refresh    | **Reindex console commands (run after creating command in application).**|
| environment        | Show/change application environment (data/environment.php).              |
| app:key            | Update encryption key for current environment.                           |
| app:reset          | Reset application runtime cache and invalidate configs.                  |
| app:touch          | Touch configuration files to reset their cached state.                   |
| create:command     | Generate new command.                                                    |
| create:controller  | Generate new controller.                                                 |
| create:document    | Generate new ODM document.                                               |
| create:middleware  | Generate new http middleware.                                            |
| create:migration   | Generate new migration.                                                  |
| create:record      | Generate new ORM Record.                                                 |
| create:request     | Generate new request filter.                                             |
| create:service     | Generate new service.                                                    |
| db:describe        | Describe table schema of specific database.                              |
| db:list            | Get list of available databases, their tables and records count.         |
| document:phpstorm  | Create "virtual" documentation and ORM/ODM tooltips for PHPStorm.        |
| i18n:export        | Export specified language to GetText PO file.                            |
| i18n:import        | Import GetText PO file to application bundles.                           |
| i18n:index         | Index all declared translation strings and usages.                       |
| inspect:entity     | Inspect specified entity schema to locate unprotected field and rules.   |
| inspect:fillable   | List every available entity associated with it's fillable fields.        |
| inspect:public     | List every available entity associated with it's public fields.          |
| migrate:init       | Configure migrations (create migrations table).                          |
| migrate:replay     | Replay (down, up) one or multiple migrations.                            |
| migrate:rollback   | Rollback one (default) or multiple migrations.                           |
| migrate:status     | Get list of all available migrations and their statuses.                 |
| modules:install    | Install available module(s) and mount it's resources.                    |
| modules:update     | Update public resources of already installed modules.                    |
| odm:schema         | Update ODM schema.                                                       |
| odm:uml            | Export ODM schema to UML.                                                |
| orm:schema         | Update ORM schema.                                                       |
| views:compile      | Compile every available view file.                                       |
| views:reset        | Clear view cache for all environments.                                   |

> You can get list of currently available commands by executing: `ConsoleDisptacher->getCommands()`.
