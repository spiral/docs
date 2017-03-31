# Translator in Views
The ViewManager component include set of view processors, one of them `TranslateProcessor` is compatible
with twig and stempler templates and provides ability to include proper i18n string at compilation stage.

Wrap your string with `[[]]` to automatically translate it.

```php
<extends:layouts.basic title="[[This is our title]]"/>

<block:content>
    [[Hello World!]]
</block:content>
```

> Attention, translation process of your view file will happen at moment of compilation and will use different cached versions for different locales, meaning are you not limited in performance so you can translate as much text in your views as you want. See [Views and Engines](/views/overview.md)

Include processor as modifier to your engine to capture strings in relation to original view file,
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