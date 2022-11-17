# Stempler - Basics

Each view is based on a template defined and stored inside the `app/views` directory (or another one configured
via the `ViewsBootloader`). Stempler templates must have the extension `.dark.php`.

To render a view template, store the file in `app/views/welcome.dark.php`.

```php
hello world
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

To pass the value to the template, pass an array as the second argument of the `render` function.

```php
return $this->views->render('welcome', [
    'name' => 'User'
]);
```

Stempler templates support PHP underlying syntax (`app/views/welcome.dark.php`):

```php
hello, <?=$name?>
```

> **Note**
> You can always use fallback to PHP when needed. Warning, this method does not prevent XSS injections!

### Auto-Escaping

Use alternative `{{ $value }}` syntax to echo the value with automatic escaping:

```php
hello, {{ $name }}
``` 

> **Note**
> The Stempler echo and directive syntax is similar to Laravel Blade.

### Context-Aware escaping

The escape strategy will change depending on where you echo your value. You can echo/embed your values inside `script` tags:

```php
<script>
    var value = {{ $name }};
</script>
```

> **Note**
> Do not use quotes whiles doing so.

### Disable Escaping

To echo value as is (**without anti-XSS**), use the alternative syntax `{!! $value !!}`:

```php
{!! $value !!}
```

## Directives

Besides classic echo constructions, the Stempler supports several Blade-like directives to control the business logic of
your templates:

```php
@foreach($values as $value)
    {{ $value }}
@endforeach
```

Read more about available directives and custom directive creation [here](/stempler/directives.md).
