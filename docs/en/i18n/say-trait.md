# Internalization - Say Trait

It is possible to add translation abilities to any application object using specialized trait 
`Spiral\Translator\Traits\TranslatorTrait`.

## Usage

We can add the translator trait to the controller:

```php
namespace App\Controller;

use Spiral\Translator\Traits\TranslatorTrait;

class HomeController
{
    use TranslatorTrait;

    public function index(): string
    {
        return $this->say('hello world!');
    }
}
```

> **Note**
> Run 'i18n:index' to index the string invocation.

## Class messages

In cases where a message defined by logic and can not be indexed use constants or properties to declare class messages,
every string wrapped with `[[]]` will be automatically indexed.
 
 ```php
class HomeController 
{
    use TranslatorTrait;

    protected const MESSAGES = [
        'error'   => '[[An error]]',
        'success' => '[[Success]]'
    ];

    public function index(): string
    {
        echo $this->say(self::MESSAGES['error']);
    }
}
 ```
