# Stempler - Directives

In addition to the classic echo constructions, Stempler supports many Blade-like directives to control the business logic of
your templates.

Unlike Blade or Twig, Stempler directives are only responsible for managing business logic.

See [Components and Props](../stempler/components.md) and [Inheritance](../stempler/inheritance.md) to check how to extend 
your templates and implement virtual components.

## Escaping control '@' letter

Just double 'at' letter like

```php
@@ // -> will be rendered as '@'
```

## Loop Directives

To loop over a list of template variables, use the following directives.

> **Note**
> The directive declaration is similar to native PHP syntax.

### Embed PHP

To embed PHP logic in your template, use the classic `<?php` and `?>` tags or alternative `@php` and `@endphp`:

```php
@php
    echo "hello world";
@endphp
```

### For

Use the directive `@for` and `@endfor` to render the loop:

```php
@for($i=0; $i<100; $i++)
    hello
@endfor
```

### While

Use the directive `@while` and `@endwhile` to render `while` loop:

```php
@php $i = 0; @endphp
@while($i < 10)
    hello world
    @php $i++; @endphp
@endwhile
```

### Break and Continue

Use the `@break` and `@continue` directives to interrupt your loops:

```php
@php $i = 0; @endphp
@while(true)
    hello world
    @if($i++>10)
        @break
    @endif
@endwhile
```

> **Note**
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

To create a conditional statement, use the `@if` and `@endif` directives:

```php
@if($value === 123)
    {{ "hello world" }}
@endif
```

To create an `else` condition, use the `@else` directive:

```php
@if($value !== 123)
    {{ "value is not 123" }}
@else
    {{ "value is 123" }}
@endif
```

To create a conditional `else` use `elseif`:

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

Use the `@unless` directive to declare a negative condition:

```php
@unless($value === 124)
    {{ "value is not 124" }}
@endunless
```

> **Note**
> You can use `@else` and `@elseif` with the `@unless` directive.

### Empty and Isset

Use the `@empty` and `@isset` conditions and `@endempty`, `@endisset` accordingly:

```php
@empty($value)
    value is empty
@endempty

@isset($value)
    value is set
@endisset
```

> **Note**
> You can combine this condition with `@else`, `@elseif`.

### Switch case

To create more complex conditions, use the `@swich`, `@case`, `@break` and `@endswitch` statements.

```php
@switch($value)
    @case(123) 123 @break
    @case(124) 124 @break
    @case(125) 125 @break
@endswitch
```

## Json Directive

To render JSON on a page, use the `@json` directive:

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

Alternatively, you can use the contextual echo via the `{{ }}` statement:

```php
<script type="text/javascript">
    var value = {{ $value  }};
    console.log(value.key);
</script>
``` 

In both cases, the generated view will look like this:

```php
<script type="text/javascript">
    var value = {"key":"value"};
    console.log(value.key);
</script>
```

## Framework specific directives

Spiral provides a number of framework-specific directives.

### Container

To invoke a container dependency into a template, use the `@inject($variable, "class")` directive:

```php
@inject($app, App\App::class)
{{ get_class($app) }}
```

### Route

To create a route, use the directive `@route`:

```php
<a href="@route('home:index')">click me</a>
```

You can use the `controller:action` pattern for targets handled by a `default route` or route name:

```php
$router->addRoute(
    'html',
    new Route('/<action>.html', new Controller(HomeController::class))
);
```

Pass arguments using the second parameter:

```php
<a href="@route('html', ['action' => 'index'])">click me</a>
```

The parameters will be automatically slugified into the route url. Those parameters that are not found in the route pattern will be passed as query parameters:

```php
<a href="@route('html', ['action' => 'index', 'id' => 10])">click me</a>
```

The result `/index.html?id=10`.

> Read more about routing and named routes [here](../http/routing.md).

## Custom Directives

You can declare and register custom directives. To create a custom directive, create a class that extends
`Spiral\Stempler\Directive\AbstractDirective`. Directive methods must be prefixed with `render` and accept
`Spiral\Stempler\Node\Dynamic\Directive` as a parameter:

```php
namespace App\Directive;

use Spiral\Stempler\Directive\AbstractDirective;
use Spiral\Stempler\Node\Dynamic\Directive;

class CustomDirective extends AbstractDirective
{
    public function renderCustom(Directive $directive): string
    {
        return '<?php echo "custom" ?>';
    }
}
```

> **Note**
> You can also implement `Spiral\Stempler\Directive\DirectiveRendererInterface` to gain lower-level access to
> functionality.

Register your directive in one of your bootloaders via the `StemplerBootloader`->`addDirective` method:

```php
namespace App\Bootloader;

use App\Directive\CustomDirective;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class CustomDirectiveBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        StemplerBootloader::class
    ];

    public function boot(StemplerBootloader $stempler): void
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

To access values passed to a directive, use the `body` and `values` properties of `Directive` retrospectively:

```php
class CustomDirective extends AbstractDirective
{
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

> **Note**
> Make sure to check if `body` is not NULL.

To access specific directive values separated by `,`:

```php
class CustomDirective extends AbstractDirective
{
    public function renderCustom(Directive $directive): string
    {
        return \sprintf(
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

Directive values will be supplied in their original PHP form, you must escape the values manually. The following PHP will
be generated for the Directive above:

```php
<?php echo (3 > 2) ? "first value larger or equals": "second value larger" ?>
```

> **Note**
> You can pass `$variables` into directives as well.

### Directive Context

To capture where a directive is invoked from, use `$directive->getContext()->getPath()`:

```php
public function renderCustom(Directive $directive): string
{
    return '<?php echo "invoked from ' . var_export($directive->getContext()->getPath(), true) . '" ?>';
}
```

> **Note**
> The output: `invoked from 'welcome'`.
