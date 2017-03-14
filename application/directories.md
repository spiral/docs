# Directory structure
Spiral application does not require any specific structure or namespace for it's files until every application class can be loaded by composer, though spiral scaffolding and cache directories comes pre-configured. Default directory structure:

Directory                         | Description                    
---                               | ---       
/                                 | Base project directory (alias **root**).
**/app/**                         | Base application directory with templates, classes, runtime cache and etc (alias **application**)
/app/classes/                     | PSR4 root in your application composer file.               
/app/classes/Bootloaders/         | Application and Service Bootloaders.
/app/classes/Commands/            | Application specific console commands.   
/app/classes/Controllers/         | Controllers.          
/app/classes/Database/            | Database models and repositories.                       
/app/classes/Middlewares/         | Http Middlewares.
/app/classes/Requests/            | RequestFilters (Validators).    
/app/classes/Models/              | Up to you.
**/app/config/**                  | Application and components configuration (alias **config**).    
/app/resources/                   | Backend resources (alias **resources**).
/app/resources/locales/           | Translations (alias **locales**).
**/app/views/**                   | You can locate view files belongs to "default" namespace here (see views config).
/runtime/                         | Runtime cache, logs and etc (alias **runtime**).
/runtime/cache/                   | Runtime cache (i.e. shared memory) (alias **cache**).     
/runtime/logs/                    | Logs handled by LogManager component.                 
/vendor/                          | Default composer directory.              
**/webroot/**                     | Public web directory (including index.php) (alias **public**).

> Spiral Framework bundle does not force any of default application namespace or strict structure except for http routing (see AppBootloader to change that), be free to add more abstractions when you need them.

## Directory aliases
Many of listed paths can be accessed using aliases, you can get directory alias using two similar methods:

```php
public function indexAction(DirectoriesInterface $directories)
{
   //You can use DirectoriesInterface
   dump($directories->directory('runtime'));
   
   //Works only in IoC scope
   dump(directory('resources'));
}
```

> You can overwrite directories at moment of core initialization.