# Translator in Views
The ViewManager component include set of view processors, one of them `TranslateProcessor` is compatible
with twig and stempler templates, wrap your string with `[[]]` to automatically translate:

```php
<extends:layouts.basic title="[[This is our title]]"/>

<block:content>
    [[Hello World!]]
</block:content>
```

> Attention, translation process of your view file will happen at moment of view compilation and use different cache versions for different locales, you not limited in performance so translate as much text in your views as you want. See [Views and Engines](/old/views/overview.md)

Include processor as **modifier** to your engine to capture strings in relation to original view file,
not composed one:

```php
/*
 * Process view source before provide it to Twig, modifiers MUST only depend on view enviroment. 
 */
'modifiers'  => [
    //Automatically replaces [[string]] with their translations
    Processors\TranslateProcessor::class,

    //Mounts view environment variables using @{name} pattern.
    Processors\EnvironmentProcessor::class,

    /*{{twig.modifiers}}*/
],
```
