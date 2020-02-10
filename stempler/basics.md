# Stempler - Basics
Each view is based on a template defined and stored inside `app/views` directory (or other configured via `ViewsBootloader`).
The Stempler templates must have an extension `.dark.php`.

To render view template store file in `app/views/welcome.dark.php`.

```php
hello word
```

Invoke in controller:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): string
    {
        return $this->views->render('welcome');
    }
}
```

You should see `hello world` on your screen.

## Echo Value
To pass value to the template pass an array as second argument of `render` function.

```php
return $this->views->render('welcome', [
    'name' => 'User'
]);
```

Stempler templates support PHP underlying syntax (`app/views/welcome.dark.php`):

```php
hello, <?=$name?>
```

> You can always fallback to PHP when needed. Attention, this way does not prevent XSS injections!

### Auto-Escaping
Use alternative `{{ $value }}` syntax to echo the value with automatic escaping:

```php
hello, {{ $name }}
``` 

> The Stempler echo and directive syntax is similar to Laravel Blade.

### Context Aware escaping
The escape strategy will change based on where you echo your value. You can echo/embed your values inside `script` tags:

```php
<script>
    var value = {{ $name }};
</script>
```

> Do not use quotes whiles doing so.

### Disable Escaping
To echo value as is (**without anti-XSS**) use alternative syntax `{!! $value !!}`:

```php
{!! $value !!}
```

## Directives
Besides classic echo constructions the Stempler supports a number of Blade-like directives to control the business 
logic of your templates:

```php
@foreach($values as $value)
    {{$value}}
@endforeach
```

Read more about available directives and custom directive creation [here](/stempler/directives.md).
