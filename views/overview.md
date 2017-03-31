# Views
Being able to render html/xml/json file based on a given set of variables is one of the most important part of any framework. Spiral provides ability to route rendering request to variour engines (including Stempler and Twig) using simple file aliases and namespaces.

## Quick Start
Before jumping into details, let's try to see how we can render simple html file with included php code. For that purposes let's create sample view in `app/views/test.php`:

```php
Hello world, <?= $name ?>!
```

To render this file we only have to invoke `render` method or `ViewsInterface` with a given value for our name. Since Views components associated with short binding "views" we can also call it using `$this->views` in our services and controllers:

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

## View Engines
As mention at top of this page spiral view component can work and automatically select multiple view engines. Default skeleton application includes 3 view engines declared in a view config file:

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
            Modifiers\TranslateModifier::class,

            //Mounts view environment variables using @{name} pattern.
            Modifiers\EnvironmentModifier::class,

            /*{{twig.modifiers}}*/
        ],

        /*
        * Here you define list of extensions to be mounted into twig engine, every extension
        * class will be resolved using container so you can use constructor dependencies.
        */
        'extensions' => [
            //Provides access to dump() and spiral() functions inside twig templates
            Engines\Twig\Extensions\SpiralExtension::class,
            
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
         * Class to be used for syntax definitions. Do not change (create new engine instead).
         */
        'syntax'     => Stempler\Syntaxes\DarkSyntax::class,

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
            Modifiers\TranslateModifier::class,

            //Mounts view environment variables using @{name} pattern.
            Modifiers\EnvironmentModifier::class,

            //This modifier automatically replace some php constructors with evaluated php code,
            //such modifier used in spiral toolkit to simplify widget includes (see documentation
            //and examples).
            Modifiers\EvaluatorExpressions::class,

            /*{{dark.modifiers}}*/
        ],

        /*
         * Processors applied to compiled view source after templating work is done and view is
         * fully composited.
         */
        'processors' => [

            //Evaluates php block with #compile comment at moment of template compilation
            Processors\EvaluateProcessor::class,

            //Provides ability to automatically include js and css requested by widgets and tags
            Spiral\Toolkit\AssetManager::class,

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

View engines applied automatically based on a file extension, for example we can rename our `test.php` view file into `test.dark.php` which will allows us to use powerful Stempler compiler. Same way can be used for Twig engine (`sample.twig`):

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

> Attention, at this moment view component only detects view engine based on file existention in order of defined engines, meaning that having two view names under same name and different extension will be treated as invalid behaviour.

## Namespaces
View component and most of mounted view engines (Twig and Stempler) support working with multiple view namespaces. Primary idea of using namespaces is to be able to isolate set of view files defined in an extension, and (in some cases), being able to redefine such files on application layer. Let's try to create our first view namespace linked to a directory `app/module/views`:

```php
'namespaces'  => [
    /*
     * This is default application namespace which can be used without any prefix.
     */
    'default'  => [
        directory("application") . 'views/',
        /*{{namespaces.default}}*/
    ],
    
    'module' => [
        directory("application") . 'module/views/',
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
    'security' => [
        directory('libraries') . 'spiral/security/source/views/',
    ],
    'vault' => [
       directory('libraries') . 'spiral/vault/source/views/',
       /*{{namespaces.vault}}*/
    ],
    /*{{namespaces}}*/
],
```

> As you can see view config already have decent amount of view namespaces mounted by modules and framework, check about module installers to find out how to automatically add your namespace at moment of module registration.

Now we can move our test view into needed directory and render it using namespace prefix:

```php
public function indexAction()
{
    return $this->views->render('module:test', [
        'name' => 'Some name'
    ]);
}
```

> Stempler and Twig also allows you to import and include views defined in other namespaces/modules, such technique used a lot in Dark widgets, for example:

```php
<dark:user bundle="spiral:bundle"/>

<spiral:form>
  our ajax form
</spiral:form>
```

One of the cool features about namespaces is that you can link one namespace to multiple directories. Let's try to check how we can use namespaces to redefine layout defined by external module, for this purposes let's try to install [Vault extension](https://github.com/spiral-modules/vault).

Once you have vault installed, let's try to open it's dashboard view using url "/vault", it will look like that:

![Dashboard](https://raw.githubusercontent.com/spiral/guide/master/resources/vault-layout.png)

For obvious reasons your might want to change layout colors, logotypes and etc, we can achieve that by mounting additional directory into vault view namespace:

```php
'vault'    => [
    directory("application") . 'views/vault/',
    directory('libraries') . 'spiral/vault/source/views/',
    /*{{namespaces.vault}}*/
],
```

As you can see this directory located before original vault extension which gives us ability to redefine our layout settings by creating file in `app/views/vault/layout.dark.php`:

```php
<dark:extends path="vault:layouts/root"/>

<!--Admin spefific elements-->
<dark:use bundle="admin:bundle"/>

<define:resources>
   <block:resources/>
   //Project specific vault styles and scripts including custom layout colors
</define:resources>

<define:brand>
    <a href="/" class="brand-logo">
        <img src="@{basePath}resources/images/project-logo.jpg" alt="Project name">
    </a>
</define:brand>

<!--Project specific actor-->
<define:user-block>
    <a href="#" class="user-link">
        <?= get_class(spiral(\Spiral\Security\ActorInterface::class)) ?>
    </a>
    <a href="#" class="user-logout hide">[[Log Out]]</a>
</define:user-block>
```

## View Cache
Spiral framework Stempler engine is pretty slow compiler, to improve your project performance view component provides engines universal settings to enable and disable compilication cache, such setting is located in a view config:

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

> As you can see you can enable and disable view cache using your .env file.

## View Environent and Modifies (cachable processors)
If you will carefully read view configuration file, you will notice one section which might look weird:

```php
'environment' => [
    'language' => ['translator', 'getLocale'],
    'basePath' => ['http', 'basePath'],
    /*{{environment}}*/
],
```

This section define set of enviroment dependencies for a view cache, in other words when any of this values (value resolved using contrainer: `['translator', 'getLocale']` = `$container->get('translator')->getLocale()`) changes - new cache version will be created.

On a practice it allows us, to perform some operations on view source **before** it will be given to view engine. For example, the most benefitial use of this feature is supplied by `TranslateModifier` which will replace every string embraced with `[[` and `]]` symbols into locate specific text.

```php
<extends:layouts.html5 title="[[Welcome To Spiral]]" git="https://github.com/spiral"/>

<define:body>
   [[Test text]
</define:body>
```

You can define your own modifier by implementing `Spiral\Views\ModifierInterface`:

```php
interface ModifierInterface
{
    /**
     * All modifiers should be requested using FactoryInterface.
     *
     * @param EnvironmentInterface $environment
     */
    //public function __construct(EnvironmentInterface $environment);

    /**
     * Modify given source.
     *
     * @param string $source    Source.
     * @param string $namespace View namespace.
     * @param string $name      View name (no extension included).
     * @return mixed
     */
    public function modify($source, $namespace, $name);
}
```

> Since modifiers are resolved using FactoryInterface so you can define any needed injection in your constuctor + [EnvironmentInterface $environment].