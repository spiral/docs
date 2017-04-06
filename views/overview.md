# Views
The Spiral views components is represented by `ViewsInterface` and, by default, implemented by `ViewManager` class. You can get access to this component using dependency or shortcut `views`.

ViewManager component does not compile or render views directly but delegate it to engine adapters
 defined in view config.

## Quick Start
To start, create `app/views/test.php`:

```php
Hello world, <?= $name ?>!
```

This file can now be rendered and send to use browser:

```php
public function indexAction(ViewsInterface $views)
{
    dump($views->render('test', [
        'name' => 'Some name'
    ]));

    dump($this->views->render('test', [
        'name' => 'Some name'
    ]));
}
```

## ViewInstance
You can also delay rendering by requesting `ViewInterface` which represent this file.


```php
public function indexAction(ViewsInterface $views)
{
    $view = $views->get('test');
    dump($view);
    
    //Can be called multiple times
    dump($view->render('test', [
        'name' => 'Some name'
    ]));
}
```

## View Engines
Spiral ships with 3 rendering engines included by default, Twig, Stempler and Native views. You can 
connect more engines by implementing `EngineInterface` and adding such implementation into views config.

```php
'engines'     => [
     /*
              * You can always extend TwigEngine class and define your own configuration rules in it.
              */
             'twig'   => [
                 'class'      => Engines\TwigEngine::class,
                 'extension'  => 'twig',
                 'options'    => [
                     'auto_reload' => true
                 ],
     
                 /*
                 * Modifiers applied to imported or extended view source before it's getting parsed by
                 * HtmlTemplater, every modifier has to implement ModifierInterface and as result view
                 * name, namespace and filename are available for it. Modifiers is the best to connect
                 * custom syntax processors (for example Laravel's Blade).
                 */
                 'modifiers'  => [
                     //Automatically replaces [[string]] with their translations
                     Processors\TranslateProcessor::class,
     
                     //Mounts view environment variables using @{name} pattern.
                     Processors\EnvironmentProcessor::class,
     
                     /*{{twig.modifiers}}*/
                 ],
     
                 /*
                 * Here you define list of extensions to be mounted into twig engine, every extension
                 * class will be resolved using container so you can use constructor dependencies.
                 */
                 'extensions' => [
                     //Provides access to dump() and spiral() functions inside twig templates
                     Engines\Twig\Extensions\SpiralExtension::class
                     /*{{twig.extension}}*/
                 ]
             ],
             /*
              * Stempler does not provide any custom command syntax (however you can connect one using
              * modifiers section), instead it compose templates together using html tags based on
              * defined syntax (in our case "Dark").
              */
             'dark'   => [
                 'class'      => Engines\StemplerEngine::class,
     
                 /*
                  * Do not change this extension, it used across spiral toolkit, profiler and
                  * administration modules.
                  */
                 'extension'  => 'dark.php',
     
                 /*
                  * Modifiers applied to imported or extended view source before it's getting parsed by
                  * HtmlTemplater, every modifier has to implement ModifierInterface and as result view
                  * name, namespace and filename are available for it. Modifiers one of the options to
                  * connect custom syntax processors (for example Laravel's Blade or Nette Latte).
                  */
                 'modifiers'  => [
                     //Automatically replaces [[string]] with their translations
                     Processors\TranslateProcessor::class,
     
                     //Mounts view environment variables using @{name} pattern.
                     Processors\EnvironmentProcessor::class,
     
                     //This modifier automatically replace some php constructors with evaluated php code,
                     //such modifier used in spiral toolkit to simplify widget includes (see documentation
                     //and examples).
                     Processors\ExpressionsProcessors::class,
     
                     /*{{dark.modifiers}}*/
                 ],
     
                 /*
                  * Processors applied to compiled view source after templating work is done and view is
                  * fully composited.
                  */
                 'processors' => [
                     //Evaluates php block with #compile comment at moment of template compilation
                     Processors\EvaluateProcessor::class,
     
                     //Drops empty lines and normalize attributes
                     Processors\PrettifyProcessor::class,
     
                     /*{{dark.processors}}*/
                 ]
             ],
             /*
              * Native engine simply executes php file without any additional features. You can access
              * NativeView object using variable $this from your view code, to get instance of view
              * container use $this->container.
              */
             'native' => [
                 'class'     => Engines\NativeEngine::class,
                 'extension' => 'php'
             ],
             /*{{engines}}*/
]
```

View engines applied automatically based on a file extension, for example we can rename our `test.php` view file into `test.dark.php` which will allows us to use powerful Stempler markup. 

Same way can be used for Twig engine (`sample.twig`):

```twig
{% extends "layouts/layout.twig" %}

{% block body %}
    <div class="wrapper">
        <div class="placeholder">
            You can access spiral container in twig templates as well:
            {{ spiral('request').getAttribute('csrfToken') }};
        
            [[You can use translator tags even in twig.]]
        </div>
    </div>
{% endblock %}
```

> Please note, that due extension based engine resolution it is possible to have views collisions. Specify full view path in order to avoid that.

## Namespaces
ViewManage and it's engines support ability to store view files in a multiple locations associated with
set of view namespaces, namespace/directory link is located in views config, section `namespaces`:

```php
'namespaces'  => [
    /*
     * This is default application namespace which can be used without any prefix.
     */
    'default'  => [
        directory("application") . 'views/',
        /*{{namespaces.default}}*/
    ],
    /*
     * This namespace contain few framework views like http error pages and exception view
     * used in snapshots. In addition, same namespace used by Toolkit module to share it's
     * views and widgets.
     */
    'spiral'   => [
        directory("libraries") . 'spiral/framework/source/views/',
        directory('libraries') . 'spiral/toolkit/source/views/',
        /*{{namespaces.spiral}}*/
    ],
    'profiler' => [
        directory('libraries') . 'spiral/profiler/source/views/',
        /*{{namespaces.profiler}}*/
    ],
    'vault'    => [
        directory('application') . 'views/vault/',
        directory('libraries') . 'spiral/vault/source/views/',
        /*{{namespaces.vault}}*/
    ],
    /*{{namespaces}}*/
],
```

> You can use placeholder `/*{{namespaces}}*/` to allow your modules to add more namespaces automatically.

You can freely use namespaces in both render method and inside your template extend and import constructions:

```php
public function indexAction()
{
    return $this->views->render('module:test', [
        'name' => 'Some name'
    ]);
}
```

```php
<dark:user bundle="spiral:bundle"/>

<spiral:form>
  our ajax form
</spiral:form>
```

> Syntaxes like "namespace:file" and "@namespace/file" are supported.

### Namespace example 
Usually namespaces used to represent set of view files provided by external module, for example - [Vault extension](https://github.com/spiral-modules/vault).

![Dashboard](https://raw.githubusercontent.com/spiral/guide/master/resources/vault-layout.png)

After installation such module namespace will appear in our view config, you can pre-pend it with user
directory (i.e. `directory("application") . 'views/vault/'`) to overwrite some of it's view files:

```php
'vault'    => [
    directory("application") . 'views/vault/',
    directory('libraries') . 'spiral/vault/source/views/',
    /*{{namespaces.vault}}*/
],
```

## View Cache
All default view engines follows save cache configuration options located in views config:

```php
'cache'       => [
    /*
     * Indicates that view engines must enable caching for their templates, you can reset existed
     * view cache by executing command "view:compile"
     */
    'enabled'   => env('VIEW_CACHE'),
    
    /*
     * Location where view cache has to be stored into. By default you can use
     * app/runtime/cache/views directory.
     */
    'directory' => directory("cache") . 'views/'
],
```

> As you can see you can enable and disable view cache using your .env file. Note, that native php
views do not use caching.

### Cache Environment
ViewManager component uses set of values provided by external components in order to create multiple
cache versions. The view environment dependencies are defined in view config in a form of function signature:
 
```php
'environment' => [
    'language' => ['translator', 'getLocale'],
    'basePath' => ['http', 'basePath'],
    /*{{environment}}*/
],
```

For example, switching application to different locale will force cache location to cache, such approach
makes it possible to compile view localization instead of calling it on every requests.

> You can define your own dependencies, for example: [auth, isAuthorized], [cms, isEditable] and etc.