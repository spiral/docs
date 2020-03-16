# Internalization - View Localization
The framework bundle includes the view process available for Twig and Stempler engines to translate view source code.
The translated view will be stored in a separate view cache and provides the ability to translate views without the performance 
penalty.

## View Translation
To activate view translation enable bootloader `Spiral\Bootloader\Views\TranslatedCacheBootloader`. Make sure to add
this bootloader before view engine bootloaders:

```php
protected const LOAD = [
    // ...
    Framework\Views\ViewsBootloader::class,
    Framework\Views\TranslatedCacheBootloader::class,
    // ...
];
```

Embrace the string to be translated with `[[ string ]]` in your template:

```html
[[hello world]]
```

> Change the locale in your application to switch translation in view.
