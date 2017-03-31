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
Descriptions of existed view processors:
 
### TranslateProcessor

### EnvironmentProcessor

### ExpressionsProcessors

### EvaluateProcessor

### PrettifyProcessor