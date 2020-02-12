# Internalization - Say Trait
Is it possible to add translation abilities to any applicatio object using specialized trait `Spiral\Translator\Traits\TranslatorTrait`.

## Example
We can add the translator trait to the controller:

```php
namespace App\Controller;

use Spiral\Translator\Traits\TranslatorTrait;

class HomeController
{
    use TranslatorTrait;

    public function index()
    {
        return $this->say("hello world!");
    }
}
```

> Run 'i18n:index' to index the string invocation.