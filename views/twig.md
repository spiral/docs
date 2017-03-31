# Twig Views
The Spiral application bundle includes Twig rendering engine by default. 

```php
$this->views->render('@application/home.twig');
```

You can skip extension when no view conflicts are expected and/or use default namespace notation:

```php
$this->views->render('application:home');
```

> Twig engine follows same cache settings as other engines, see views config.

## Definition
TwigEngine defined in views config:
```php
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
```

Use `options` array to pass values into `Twig_Environment`. View translation using `[[]]` and environment values `@{name}` are allowed in twig templates due to `TranslateProcessor` and `EnvironmentProcessor`.

## Twig Extension
Use `extensions` section or twig config to define `Twig_Extension` list (resolved via container).

Default `SpiralExtension` extension provides access to IoC scope and `spiral` function:

```twig
 <script>
    window.csrfToken = "{{ spiral('request').getAttribute('csrfToken') }}";
</script>
```

> `dump` function is available as well.