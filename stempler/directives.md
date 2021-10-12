# Stempler - Directives
Besides classic echo constructions, the Stempler supports many Blade-like directives to control the business logic of your templates.

Unlike Blade or Twig, the Stempler directives only responsible for business logic management. See [Components and Props](/stempler/components.md) 
and [Inheritance](/stempler/inheritance.md) to check how to extend your templates and implement virtual components.

## Escaping control '@' letter

Just double 'at' letter like
```php
@@ // -> will be rendered as '@'
```


## Loop Directives
To loop over a list of template variables, use the following directives.

> Directive declaration is similar to native PHP syntax.

### Embed PHP
To embed PHP logic in your template use classic `<?php` and `?>` tags or alternative `@php` and `@endphp`:

```php
@php
    echo "hello world";
@endphp
```

### For
Use directive `@for` and `@endfor` to render the loop:

```php
@for($i=0; $i<100; $i++)
    hello
@endfor
```

### While
Use directive `@while` and `@endwhile` to render `while` loop:

```php
@php $i = 0; @endphp
@while($i < 10)
    hello world
    @php $i++; @endphp
@endwhile
```

### Break and Continue
Use `@break` and `@continue` directives to interrupt your loops:

```php
@php $i = 0; @endphp
@while(true)
    hello world
    @if($i++>10)
        @break
    @endif
@endwhile
```

> `@break(2)` is equivalent to `break 2`. Read more about `if` directives below.

## Conditional Directives
Stempler provides some conditional directives which transcribed into native PHP code.

The examples are given with the following variables:

```php
return $this->views->render('welcome', [
    'value' => 123
]);
```

### If and Else
To create conditional statement use `@if` and `@endif` directives:

```php
@if($value === 123)
    {{ "hello world" }}
@endif
```

To create `else` condition use `@else` directive:

```php
@if($value !== 123)
    {{ "value is not 123" }}
@else
    {{ "value is 123" }}
@endif
```

To create conditional `else` use `elseif`:

```php
@if($value === 124)
    {{ "value is 124" }}
@elseif($value === 123)
    {{ "value is 123" }}
@else
    {{ "another value" }}
@endif
```

### Unless
Use `@unless` directive to declare negative condition:

```php
@unless($value === 124)
    {{ "value is not 124" }}
@endunless
```

> You can use `@else` and `@elseif` with a `@unless` directive.

### Empty and Isset
Use `@empty` and `@isset` conditions and `@endempty`, `@endisset` accordingly:

```php
@empty($value)
    value is empty
@endempty

@isset($value)
    value is set
@endisset
```

> You can combine this condition with `@else`, `@elseif`.

### Switch case
To create more complex conditions use `@swich`, `@case`, `@break` and `@endswitch` statements.

```php
@switch($value)
    @case(123) 123 @break
    @case(124) 124 @break
    @case(125) 125 @break
@endswitch
```

## Json Directive
To render JSON on a page use `@json` directive:

```php
@json($value)
```

You can embed json inside JavaScript statements:

```php
return $this->views->render('welcome', [
    'value' => ['key' => 'value']
]);
```

In your template:

```php
<script type="text/javascript">
    var value = @json($value);
    console.log(value.key);
</script>
```

Alternatively you can use contextual echo via `{{ }}` statement:

```php
<script type="text/javascript">
    var value = {{ $value  }};
    console.log(value.key);
</script>
``` 

In both cases the generated view will look like:

```php
<script type="text/javascript">
    var value = {"key":"value"};
    console.log(value.key);
</script>
```

## Framework specific directives
Spiral provides a number of framework-specific directives.

### Container
To invoke container dependency into template use `@inject($variable, "class")` directive:

```php
@inject($app, App\App::class)
{{ get_class($app) }}
```

### Route
To create route use directive `@route`:

```php
<a href="@route('home:index')">click me</a>
```

You can use `controller:action` pattern for targets handled by `default route` or route name:

```php
$router->addRoute(
    'html',
    new Route('/<action>.html', new Controller(HomeController::class))
);
```

Pass arguments using second parameter:

```php
<a href="@route('html', ['action' => 'index'])">click me</a>
```

Parameters will be automatically slugified into route url. Parameters not found in route pattern will be passed as
query parameters:

```php
<a href="@route('html', ['action' => 'index', 'id' => 10])">click me</a>
```

The result `/index.html?id=10`.

> Read more about routing and named routes [here](/http/routing.md).

## Custom Directives
You can declare and register custom directives. To create custom directive create a class which extends 
`Spiral\Stempler\Directive\AbstractDirective`. Directive methods must be prefixed with `render` and accept
`Spiral\Stempler\Node\Dynamic\Directive` as parameter:

```php
namespace App\Directive;

use Spiral\Stempler\Directive\AbstractDirective;
use Spiral\Stempler\Node\Dynamic\Directive;

class CustomDirective extends AbstractDirective
{
    /**
     * @param Directive $directive
     * @return string
     */
    public function renderCustom(Directive $directive): string
    {
        return '<?php echo "custom" ?>';
    }
}
```

> You can also implement `Spiral\Stempler\Directive\DirectiveRendererInterface` to gain lower-level access to functionality.

Register your directive in one of your bootloaders via `StemplerBootloader`->`addDirective` method:

```php
namespace App\Bootloader;

use App\Directive\CustomDirective;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class CustomDirectiveBootloader extends Bootloader
{
    protected const DEPENDENCIES = [StemplerBootloader::class];

    public function boot(StemplerBootloader $stempler)
    {
        $stempler->addDirective(CustomDirective::class);
    }
}
```

Invoke the directive in your template:

```php
@custom
```

### Passing values
To access values passed to directive use `body` and `values` properties of `Directive` retrospectively:

```php
class CustomDirective extends AbstractDirective
{
    /**
     * @param Directive $directive
     * @return string
     */
    public function renderCustom(Directive $directive): string
    {
        return $directive->body;
    }
}
```

Example:

```php
@custom(1, "hello world")
```

Output:

```php
"hello world"
```

> Make sure to check if `body` not NULL.

To access specific directive values separated by `,`:

```php
class CustomDirective extends AbstractDirective
{
    /**
     * @param Directive $directive
     * @return string
     */
    public function renderCustom(Directive $directive): string
    {
        return sprintf(
            '<?php echo (%s > %s) ? "first value larger or equals": "second value larger" ?>',
            $directive->values[0],
            $directive->values[1]
        );
    }
}
```

Example:

```php
@custom(1, 2)
```

Directive values will be supplied in their original PHP form, you must escape values manually. The following PHP will
be generated for the Directive above:

```php
<?php echo (3 > 2) ? "first value larger or equals": "second value larger" ?>
```

> You can pass `$variables` into directives as well.

### Directive Context
To capture where directive is invoked from use `$directive->getContext()->getPath()`:

```php
/**
 * @param Directive $directive
 * @return string
 */
public function renderCustom(Directive $directive): string
{
    return '<?php echo "invoked from ' . var_export($directive->getContext()->getPath(), true) . '" ?>';
}
```

> The output: `invoked from 'welcome'`.
