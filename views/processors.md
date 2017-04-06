# View Processors
Both Twig and Stempler engines include support for `Spiral\Views\ProcessorInterface` which works similar to http middlewares and modifies raw view source before or after it been pushed though render engine.

## ProcessorInterface
```php
interface ProcessorInterface
{
    /**
     * @param EnvironmentInterface $environment
     * @param ViewSource           $view
     * @param string               $code
     *
     * @return string
     */
    public function modify(
        EnvironmentInterface $environment,
        ViewSource $view,
        string $code
    ): string;
}
```

## Default Processors
Descriptions of existed view processors.
 
### TranslateProcessor
Replaces all `[[strings]]` with locale specific translation.
 
> Note that by default View environment depends on current locale value, switching language will
cause new cache location and view recompilation.

### EnvironmentProcessor
Replaces all `@{constructs|default}` constructions with environment dependency value, for example:

View config:
```php
'environment' => [
    'language' => ['translator', 'getLocale'],
    'basePath' => ['http', 'basePath'],
    /*{{environment}}*/
],
```

View:
```html
Current language @{language}
```

### ExpressionsProcessors
Creates set of macros to handle php extraction for toolkit and EvaluateProcessor.

> This is internal plugin.

### EvaluateProcessor
Provides ability to execute php blocks with "#compile" comment in compilation mode (only once).

```php
This view is compiled at <?= date('c') #compile ?>
```

> Blocks marked as #compile does not have access to view data.

### PrettifyProcessor
Replaces extra lines in view and drops empty html attributes.