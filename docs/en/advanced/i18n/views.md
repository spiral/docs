# Internalization - View Localization

The Spiral Framework includes a view process available for `Twig` and `Stempler` engines to translate view source code.
The translated view will be stored in a separate view cache and it provides the ability to translate views without any performance penalty.

## View Translation

To activate view translation, enable the bootloader `Spiral\Bootloader\Views\TranslatedCacheBootloader`. Make sure to add this bootloader before viewing engine bootloaders:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Views\Bootloader\ViewsBootloader::class,
    \Spiral\Bootloader\Views\TranslatedCacheBootloader::class,
    // ...
];
```

Embrace the string to be translated with `[[ string ]]` in your template:

```html
[[hello world]]
```

> **Note**
> Change the locale in your application to switch translation in the view.
