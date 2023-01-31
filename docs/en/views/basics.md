# Views — Basics

The Spiral Framework provides a `spiral/views` component that serves as an abstraction layer for handling views. It
provides a set of interfaces for registering template engines and using them to render templates.

## ViewInterface

It provides methods for obtaining a view instance associated with a given path, compiling a cache version of a view,
resetting the view cache, and rendering a template.

> **Note**
> The component is available via the prototyped property `views`. Read more about prototype trait in
> the [The Basics — Prototyping](../basics/prototype.md) section.

### Rendering

To render a view in the controller or other service, simply invoke the `render` method. The view name does not need to 
include an extension or namespace (default to be used).

```php app/src/Interface/Controller/HomeController.php
namespace App\Interface\Controller;

use Spiral\Views\ViewsInterface;

class HomeController
{
    public function __construct(
      private readonly ViewsInterface $views
    ) {
    }
    
    public function index(): string
    {
        return $this->views->render('home');
    }
}
```

To render the view with the passed data, use the second array argument:

```php app/src/Interface/Controller/HomeController.php
public function index(): string
{
    return $this->views->render('home', [
        'key' => 'value'
    ]);
}
```

### Obtaining a View Instance

In some cases it might be more performant to cache the view in a stateless component. To obtain an instance of a view,
you can use the `get` method.

```php
use Spiral\Views\ViewsInterface;

public function index(): string
{
    /** @var \Spiral\Views\ViewInterface $view */
    $view = $this->views->get('profile-card');
    
    $card1 = $view->render('name' => 'John']);
    $card2 = $view->render(['name' => 'Jane']);
    
    return "<html><body>{$card1} {$card2}</body></html>";
}
```

### Compiling a Cache

The `compile` method can be used to compile one of multiple cache versions of a view. This method takes a view path as 
an argument and compiles the cache version of the view.

```php app/src/Interface/Cli/Command/CompileViewsCommand.php
namespace App\Api\Cli\Command;

use Spiral\Console\Command;
use Spiral\Views\LoaderInterface;
use Spiral\Views\ViewsInterface;

class WarmupViewsCommand extends Command
{
    protected const NAME = 'views:compile';
    protected const DESCRIPTION = 'Compile all views in default namespace';

    public function __invoke(LoaderInterface $loader, ViewsInterface $views): void
    {
        $templates = $loader->withExtension('dark.php')->list();
        // or
        $templates = $loader->withExtension('twig')->list(namespace: 'my-package');

        foreach ($templates as $template) {
            $views->compile($template);
        }
    }
}
```

### Resetting the View Cache

The `reset` method can be used to reset the cache of a view. This method takes a view path as an argument and resets 
the cache for the view.

```php
use Spiral\Views\ViewsInterface;

$views->reset('home');
```

## LoaderInterface

The` Spiral\Views\LoaderInterface` is responsible for loading the source of a view. A view loader can be used to check
if a view exists, load its source, and list all available views.

> **Note**
> By default LoaderInterface is implemented by `Spiral\Views\ViewLoader` class that provides file-based view loading.

In order to use the methods defined in the `LoaderInterface`, you first need to specify an extension for the loader.
This can be done by calling the `withExtension` method and passing in the desired extension.

> **Warning**
> The `withExtension` method is immutable. It will return a new instance of `LoaderInterface`.

```php
use Spiral\Views\LoaderInterface;

public function index(LoaderInterface $loader): string
{
    $loader = $loader->withExtension('dark.php');
    
    // ...
}
```

### Check If a View Exists

The `exists` method can be used to check if a view is available for rendering.

```php
if (!$loader->exists('home')) {
    throw new \RuntimeException('View not found');
}

// ...
```

You can also check if a view exists in a specific namespace.

```php
$loader->exists('my-package:home')
````

### Loading View Source

The `load` method can be used to retrieve the source of a view. It will return the source of the view as
a `Spiral\Views\ViewSource` instance.

```php
/** @var \Spiral\Views\ViewSource $source */
$source = $loader->load('home');

$source->getNamespace(); // default
$source->getName(); // home
$source->getFilename(); // /app/views/home.dark.php
$source->getCode(); // <html>...</html>
```

In some cases, you want to replace the source code of a view. To do this, you can use the `withCode` method.

```php
/** @var \Spiral\Views\ViewSource $source */
$source = $loader->load('home');

$source = $source->withCode('<div>...</div>');
$source->getCode(); // <div>...</div>
```

> **Warning**
> The `withCode` method is immutable. It will return a new instance of `ViewSource`.

### Listing Available Views

The `list` method can be used to list all available views. It will return an array of view paths.

```php
$views = $loader->list();
```

You can also list all available views in a specific namespace.

```php
$views = $loader->list(namespace: 'my-package');
```

### Custom View Loader

You can create your own loader by implementing the `Spiral\Views\LoaderInterface` interface.

Here is an example of a loader that loads views from a database.

> **Warning**
> This is just an example, it is not tested and should not be used as is.

```php app/src/Application/Loader/DatabaseLoader.php
namespace App\Application\Loader;

use Spiral\Views\LoaderInterface;
use Spiral\Views\Loader\PathParser;
use Spiral\Views\ViewSource;

final class DatabaseLoader implements LoaderInterface
{
    private ?PathParser $parser = null;
    
    public function __construct(
        private readonly DatabaseInterface $database,
        private readonly string $defaultNamespace = self::DEFAULT_NAMESPACE,
    ) {}
    
    public function withExtension(string $extension): LoaderInterface
    {
        $loader = clone $this;
        $loader->parser = new PathParser($this->defaultNamespace, $extension);

        return $loader;
    }
    
    public function getExtension(): ?string
    {
        return $this->parser?->getExtension();
    }
    
    public function exists(string $path, string &$filename = null, ViewPath &$parsed = null): bool
    {
        if ($this->parser === null) {
            throw new LoaderException(
                'Unable to locate view source, no extension has been associated.'
            );
        }
        
        $parsed = $this->parser->parse($path);
        if ($parsed === null) {
            return false;
        }
        
        if (!isset($this->namespaces[$parsed->getNamespace()])) {
            return false;
        }
        
        // Iterate over all registered namespaces and check if the view exists in 
        // any of them.
        foreach ((array)$this->namespaces[$parsed->getNamespace()] as $namespace) {
            $isExists = $this->getTemplate($namespace, $parsed->getBasename()) !== null;
            if ($isExists) {
                $filename = $parsed->getBasename();
                return true;
            }
        }
        
        return false;
    }
    
    public function load(string $path): ViewSource
    {
        if (!$this->exists($path, $filename, $parsed)) {
            throw new LoaderException(
                \sprintf('Unable to load view `%s`, file does not exist.', $path)
            );
        }
        
        // View variable contains an array of record from the database with the 
        // following structure:
        // ['html' => '<div>...</div>', 'path' => '...', 'namespace' => '...']
        $view = $this->getTemplate($parsed->getNamespace(), $parsed->getBasename());
        
        return (new ViewSource(
            $filename,
            $parsed->getNamespace(),
            $parsed->getName()
        ))->withCode($view['html']);
    }
    
    public function list(string $namespace = null): array
    {
        $views = [];
        foreach ($this->namespaces as $namespace) {
            $templates = $this->database->select()
                ->from('views')
                ->where('namespace', $namespace)
                ->fetchAll();
            
            foreach ($templates as $template) {
                $views[] = $namespace . ':' . $template['path'];
            }
        }
        
        return $views;
    }
    
    private function getTemplate(string $namespace, string $path): ?array
    {
        return $this->database->select()
            ->from('views')
            ->where('namespace', $namespace)
            ->where('path', $path)
            ->fetchOne();
    }
}
```

To use this loader, you need to replace a default one in the container.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Views\LoaderInterface;
use App\Application\Loader\DatabaseLoader;

class AppBootloader extends Bootloader 
{
    protected const SINGLETONS = [
        LoaderInterface::class => [self::class, 'initLoader'],
    ];
    
    protected function initLoader(DatabaseInterface $database): LoaderInterface
    {
        return new DatabaseLoader($database);
    }
}
```

## Global variables

You can add global variables that will be available in all views.

There are several ways to add a global variable.

:::: tabs

::: tab Config
You can use the configuration file `app/config/views.php`.

```php app/config/views.php
return [
    // ...
    'globalVariables' => [
        'some_var' => env('SOME_VALUE'),
        'other_var' => 'other_value'
    ]
];
```

:::

::: tab GlobalVariablesInterface
You can use `Spiral\Views\GlobalVariablesInterface`.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\GlobalVariablesInterface ;
use Spiral\Boot\EnvironmentInterface;

class AppBootloader extends Bootloader 
{
    public function boot(GlobalVariablesInterface $vars, EnvironmentInterface $env): void
    {
         $vars->set('some_var', $env->get('SOME_VALUE'));
         $vars->set('other_var', 'other_value');
    }
}
```

:::

::::

After that, you can use the variables in any view template including inherited templates.

```php
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <?=$some_var?>

    <?=$other_var?>
</body>
</html>
```

## Namespaces

In the Views component, namespaces are used to organize and categorize views. Views can be separated into
different namespaces based on their purpose or associated feature. This makes it easier to manage views, avoid naming
conflicts, and reuse common templates.

### Registering Namespaces

To register a namespace, you can use the `addDirectory` method of the `ViewsBootloader` class. This method takes two
arguments: the first is the namespace name, and the second is the directory path associated with that namespace.

For example, if you have a directory called views in the `my-package` directory of your application, you can register
it with the `my-package` namespace as follows:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\Bootloader\ViewsBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ViewsBootloader $views): void
    {
        $views->addDirectory('my-package', directory('root') . 'my-package/views');
    }
}
```

### Accessing Namespaces

With this namespace registered, you can now refer to views in the views directory using the `my-package` namespace.
For example, if you have a view file called `home.dark.php` in the `my-package/views` directory, you can render it as
follows:

```php
$views->render('my-package:home');
```

## Template Engine

### Available

By default, the Views component provides only one template engine - [Plain PHP templating](./templates.md). But there
are also other template engines available:

- [Stempler](./stempler.md)
- [Twig](./twig.md)

### Custom Template Engine

The Component provides the ability to register a custom template engine. To use a custom template engine you need to
implement the `Spiral\Views\EngineInterface`.

The following is a brief explanation of `EngineInterface` methods:

#### Loader

The `withLoader` method is used to set and configure the views loader for the engine. This is the place where you can
define the extension of the view files that will be used by the engine.

```php
use Spiral\Views\EngineInterface;
use Spiral\Views\LoaderInterface;

final class BladeEngine implements EngineInterface
{
    private ?LoaderInterface $loader = null;
    
    public function withLoader(LoaderInterface $loader): EngineInterface
    {
        $engine = clone $this;
        $engine->loader = $loader->withExtension('blade.php');
    
        return $engine;
    }
    
    public function getLoader(): LoaderInterface
    {
        return $this->loader;
    }
}
```

#### Compile and Reset Cache

The `compile` and `reset` methods compiles (and resets cache) for the given view path in a provided context. This
methods will be called each time the view needs to be recompiled.

```php
use Spiral\Views\ContextInterface;
use Illuminate\View\Compilers\CompilerEngine;

private readonly CompilerEngine $blade;

public function compile(string $path, ContextInterface $context): mixed
{
    $filepath = $this->getLoader()->load($path)->getFilename();
    
    // Run the compilation process for the given view path
    $this->blade->getCompiler()->compile($filepath);
}

public function reset(string $path, ContextInterface $context): void
{
    $filepath = $this->getLoader()->load($path)->getFilename();
    
    // Reset the cache for the given view path
    \unlink($this->blade->getCompiler()->getCompiledPath($filepath));
}
```

#### Get view instance

The `get` method returns an instance of the view class associated with the view path.

```php
use Spiral\Views\ViewInterface;
use Illuminate\View\Engines\CompilerEngine;

private readonly CompilerEngine $blade;

public function get(string $path, ContextInterface $context): ViewInterface
{
    $filepath = $this->getLoader()->load($path)->getFilename();
    
    return new BladeView($this->blade, $filepath);
}
```

```php
use Spiral\Views\ViewInterface;
use Illuminate\View\Engines\CompilerEngine;

class BladeView implements ViewInterface
{
    public function __construct(
        private readonly CompilerEngine $blade, 
        private readonly string $filepath
    ) { 
    }
    
    public function render(array $data = []): string
    {
        return $this->blade->get($this->filepath, $data);
    }
}
```

### Registering Custom Template Engine

The `addEngine` method of `Spiral\Views\Bootloader\ViewsBootloader` allows you to add your custom template engine to the
Views component and make it available for use in your application.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\Bootloader\ViewsBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ViewsBootloader $views): void
    {
        $views->addEngine(MyEngine::class);
    }
}
```

## Events

| Event                           | Description                                         |
|---------------------------------|-----------------------------------------------------|
| Spiral\Views\Event\ViewNotFound | The Event will be fired when the view is not found. |

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
