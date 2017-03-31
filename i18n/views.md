
## Translating view files
Most of your location process will happen inside your view files, instead of forcing to use translator interface
or short function every time you want to mark your text to be translated simply embrace your text with [[ and ]]
characters:

```php
<extends:layouts.basic title="[[This is our title]]"/>

<block:content>
    [[Hello World!]]
</block:content>
```

> Attention, translation process of your view file will happen at moment of compilation and will use different cached
versions for different locales, meaning are you not limited in perfomance so you can translate as much text in your views
as you want. See [Views and Engines](/components/views.md)

You can also use such methodic in you twig files, view messages will be automatically registered at moment of compilation.
