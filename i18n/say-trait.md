# Internalization - Say Trait
Is it possible to add translation abilities to any applicatio object using specialized trait `Spiral\Translator\Traits\TranslatorTrait`.

## Usage
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

## Class messages
In cases where message is defined by logic and can not be indexed use constants and/or properties to declare class messages,
every string wrapped with `[[]]` will be automatically indexed.
 
 ```php
class HomeController 
{
    use TranslatorTrait;

    protected const MESSAGES = [
        'error'   => '[[An error]]',
        'success' => '[[Success]]'
    ];

    public function index()
    {
        echo $this->say(self::MESSAGES['error']);
    }
}
 ```