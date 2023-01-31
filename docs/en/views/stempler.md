# Views — Stempler templating

The Stempler engine provides a powerful and flexible template engine with an ability to customize it on the lexer,
parser, and AST compilation levels. By default, the driver is enabled with the web build of spiral skeleton application
and provides support for Blade-like directives and echoing, HTML components, stacks, and more.

## Basics usage

In this section, we will walk you through the steps of creating and rendering a basic view using Stempler.

### Create a view

The first step is to create the view file. The view file should be saved in the `app/views` directory (or any other
directory configured in the `ViewsBootloader`). The file extension for Stempler templates must be `.dark.php`.

let's create a view file `welcome.dark.php` with the following content:

```php app/views/welcome.dark.php
Hello, {{ $name }}!
```

And store it in the `app/views` directory.

### Render the view

Now we can render the view from the controller.

:::: tabs

::: tab Prototyped
In our example, we will use the `PrototypeTrait` to simplify getting the `ViewsInterface` instance from the container.

> **See more**
> Read more about prototype trait in the [The Basics — Prototyping](../basics/prototype.md) section.

```php app/src/Interface/Controller/HomeController.php
namespace App\Interface\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): string
    {
        return $this->views->render('welcome', [
            'name' => 'John',
        ]);
    }
}
```

:::

::: tab ViewInterface
You can also access the `Spiral\Views\ViewsInterface` instance directly from the container.

```php app/src/Interface/Controller/HomeController.php
namespace App\Interface\Controller;

use Spiral\Views\ViewsInterface;

class HomeController
{
    public function __construct(
        private readonly ViewsInterface $views
    ) {
    }

    public function index(): string
    {
        return $this->views->render('welcome', [
            'name' => 'John',
        ]);
    }
}
```

:::

::::

You should see `Hello, John!` on your screen.

Stempler templates also support PHP underlying syntax:

```php app/views/welcome.dark.php
Hello, <?= $name ?>!
```

> **Danger**
> It's important to note that the syntax `{{ $name }}` provides automatic escaping, which helps to prevent
> security issues such as [XSS attacks](https://owasp.org/www-community/attacks/xss/). On the other hand, the
> traditional PHP syntax `<?= $name ?>` does not provide automatic escaping. If you choose to use the traditional PHP
> syntax, it is recommended to manually escape the variables to ensure the security of your application.

#### Context-Aware escaping

The escape strategy will change depending on where you echo your value. You can echo/embed your values inside `script`
tags:

```php app/views/welcome.dark.php
<script>
    const value = {{ $name }};
</script>
```

It will be rendered differently depending on the type of the value:

:::: tabs

::: tab String

In case of a string value `['name' => 'John']`, the value will be automatically quoted:

```html
<script>
    const value = "John";
</script>
```

:::

::: tab Number

In case of a number value `['name' => 123]`:

```html
<script>
    const value = 123;
</script>
```

:::

::: tab Null

In case of null value `['name' => null]`:

```html
<script>
    const value = null;
</script>
```

:::

::: tab List

In case of an array `['name' => ['John']]` value:

```html
<script>
    const value = ["John"];
</script>
```

:::

::: tab Array

In case of an associative array `['name' => ['first' => 'John', 'last' => 'Doe']]` value:

```html
<script>
    const value = {"first": "John", "last": "Doe"};
</script>
```

:::

::::

#### Disable Escaping

To output a value without any automatic escaping, you can use the alternative syntax.

```php
{!! $value !!}
```

This can be useful when you want to output HTML content or other types of content that should not be escaped.

Here is an example:

```php app/src/Interface/Controller/HomeController.php
public function index(): string
{
    return $this->views->render('welcome', [
        'html' => '<div>Hello world</div>'
    ]);
}
```

View file:

```php app/views/welcome.dark.php
{!! $html !!}
```

And an output:

:::: tabs

::: tab Unescaped
With disabled escaping HTML content will be outputted as is, without any automatic escaping.

```html
<div>Hello world</div>
```

:::

::: tab Escaped
On the other hand, if you use the syntax `{{ $html }}` with automatic escaping enabled, the HTML tags will be escaped.

```html
&lt;div&gt;Hello world&lt;/div&gt;gt;
```

:::

::::

<hr>

## Directives

In addition to the classic echo constructions, Stempler supports many Blade-like directives to control the business
logic of your templates.

Unlike Blade or Twig, Stempler directives are only responsible for managing business logic.

> **Note**
> See [Components and Props](#components-and-props) and [Inheritance](#inheritance-and-stacks) to check how to extend
> your templates and implement virtual components.

### Loop Directives

Stempler provides several loop directives to help you manage the rendering of repetitive elements in your templates.
These directives make it easy to incorporate dynamic content into your templates.

> **Note**
> The directive declaration is similar to native PHP syntax.

#### Foreach

Use the directive `@foreach` and `@endforeach` to render the loop:

```php
<ul>
    @foreach($items as $item)
    <li>{{ $item }}</li>
    @endforeach
</ul>
```

#### For

Use the directive `@for` and `@endfor` to render the loop:

```php
<ul>
    @for($i = 0; $i < 10; $i++)
    <li>{{ $i }}</li>
    @endfor
</ul>
```

#### While

Use the directive `@while` and `@endwhile` to render `while` loop:

```php
<ul>
    @while($i < 10)
    <li>{{ $i }}</li>
    @php $i++; @endphp
    @endwhile
</ul>
```

#### Break and Continue

Use the `@break` and `@continue` directives to interrupt your loops:

```php
<ul>
    @while(true)
    <li>{{ $i }}</li>
    @if($i++ > 10)
        @break
    @endif
    @endwhile
</ul>
```

> **Note**
> `@break(2)` is equivalent to `break 2`. Read more about `if` directives below.

### Conditional Directives

Stempler provides several directives for creating conditional statements in your templates. These directives are
transcribed into native PHP code and offer a more readable and efficient way to handle conditions in your templates.

The examples are given with the following variables:

```php
return $this->views->render('welcome', [
    'value' => 123
]);
```

#### If and Else

To create a simple conditional statement, use the `@if` and `@endif` directives.

```php
@if($value === 123)
    Hello World
@endif
```

To add an `else` condition, use the `@else` directive.

```php
@if($value !== 123)
    Value is not 123
@else
    
@endif
```

For more complex conditions, use the `@elseif` directive.

```php
@if($value === 124)
    Value is not 124
@elseif($value === 123)
    Value is 123
@else
    Another value
@endif
```

#### Unless

The `@unless` directive allows you to create a negative condition, and can be used with `@else` and `@elseif` like
the `@if` directive.

```php
@unless($value === 124)
    Value is not 124
@endunless
```

> **Note**
> You can use `@else` and `@elseif` with the `@unless` directive.

#### Empty and Isset

Use the `@empty` and `@isset` conditions to check if a variable is empty or set, respectively.

:::: tabs

::: tab Empty

```php
@empty($value)
    Value is empty
@endempty
```

:::

::: tab Isset

```php
@isset($value)
    Value is set
@endisset
```

:::

::::

#### Switch case

For more complex conditions, you can use the `@switch`, `@case` and `@break` statements.

```php
@switch($value)
    @case(123) value is 123 @break
    @case(124) value is 124 @break
    @case(125) value is 125 @break
@endswitch
```

### Json Directive

The `@json` directive allows you to render JSON data within a page. To use it, simply pass a variable to the directive,
like this:

```php
@json($value)
```

> **Note**
> The `@json` directive is equivalent to `json_encode($value)`.

And setting a variable:

```php
return $this->views->render('welcome', [
    'value' => ...
]);
```

And the output will be:

:::: tabs

::: tab String

In case of a string value `['value' => 'Hello world']`:

```html
"Hello world"
```

:::

::: tab Number

In case of a number value `['value' => 123]`:

```html
123
```

:::

::: tab Null

In case of null value `['value' => null]`:

```html
null
```

:::

::: tab List

In case of an array `['value' => ['John']]` value:

```html
["John"]
```

:::

::: tab Array
In case of an associative array `['value' => ['first' => 'John', 'last' => 'Doe']]` value:

```html
{"first":"John","last":"Doe"}
```

:::
::::

#### Embedding JSON data

It can be useful to embed JSON data inside JavaScript statements:

Here is an example of a view template with value `['value' => ['key' => 'value']]`:

```php
<script type="text/javascript">
    var value = @json($value);
    console.log(value.key);
</script>
```

The generated view will then look like this:

```php
<script type="text/javascript">
    var value = {"key":"value"};
    console.log(value.key);
</script>
```

### Framework specific directives

The Spiral Framework provides a number of framework-specific directives to be used in templates, including:

#### Container

To invoke a container dependency into a template, use the `@inject($variable, "class")` directive:

```php
@inject($app, App\App::class)
{{ get_class($app) }}
```

#### Route

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

The parameters will be automatically slugified into the route url. Those parameters that are not found in the route
pattern will be passed as query parameters:

```php
<a href="@route('html', ['action' => 'index', 'id' => 10])">click me</a>
```

The result `/index.html?id=10`.

> **See more**
> Read more about routing and named routes in the [HTTP — Routing](../http/routing.md) section.

### Raw PHP

To embed PHP logic in your template, use the classic `<?php` and `?>` tags or alternative `@php` and `@endphp`:

```php
@php
    echo "hello world";
@endphp
```

### Escaping control '@' letter

Just double 'at' letter like

```php
@@ // -> will be rendered as '@'
```

### Custom Directives

Stempler provides a way to extend its functionality through custom directives. A custom directive is a class that
extends the `Spiral\Stempler\Directive\AbstractDirective` class and implements a render method that accepts
a `Spiral\Stempler\Node\Dynamic\Directive` parameter.

To create a custom directive, follow these steps:

#### Create a directive class

Create a class that extends `Spiral\Stempler\Directive\AbstractDirective` and implements the render method with the
desired functionality.

```php app/src/Application/Stempler/DatetimeDirective.php
namespace App\Application\Stempler;

use Spiral\Stempler\Directive\AbstractDirective;
use Spiral\Stempler\Node\Dynamic\Directive;

final class DatetimeDirective extends AbstractDirective
{
    public function renderDateTime(Directive $directive): string
    {
        return '<?php echo date("Y-m-d H:i:s"); ?>';
    }
}
```

> **Note**
> It's also possible to implement the `Spiral\Stempler\Directive\DirectiveRendererInterface` for lower-level access to
> the rendering process.

#### Register the directive

Register the custom directive using the `StemplerBootloader::addDirective()` method in a bootloader class.

```php app/src/Application/Bootloader/CustomDirectiveBootloader.php
namespace App\Application\Bootloader;

use App\Application\Stempler\DatetimeDirective;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

final class CustomDirectiveBootloader extends Bootloader
{
    public function boot(StemplerBootloader $stempler): void
    {
        $stempler->addDirective(DatetimeDirective::class);
    }
}
```

#### Use the directive

The custom directive can be used in the template by invoking it with the appropriate syntax.

Here is the template code:

```php
<div>
    @dateTime
</div>
```

And this is the final PHP code generated by the directive:

```php
<div>
    <?php echo date("Y-m-d H:i:s"); ?>
</div>
```

By using the custom directive, you can add custom functionality to the template engine and reuse it across different
templates.

#### Passing values

You can pass values to a custom directive by using the `body` and `values` properties of the `Directive` object. These
properties can be used to access the values passed to the directive. This allows you to pass dynamic values to the
directive, making it more flexible and reusable.

Here is an example of using the `body` property:

```php app/src/Application/Stempler/DatetimeDirective.php
public function renderDateTime(Directive $directive): string
{
    return \sprintf('<?php echo date(%s ?? "Y-m-d H:i:s"); ?>', $directive->body);
}
```

Example:

:::: tabs

::: tab String

```php
<div>
    @dateTime('l')
</div>
```

This directive will generate the following PHP code:

```php
<div>
    <?php echo date('l'); ?>
</div>
```

:::

::: tab Variable

```php
@php
$format = 'l';
@endphp
<div>
    @dateTime($format)
</div>
```

This directive will generate the following PHP code:

```php
<?php
$format = 'l';
?>

<div>
   <?php echo date($format); ?>
</div>
```

:::

::: tab Constant

```php
@dateTime(DATE_RFC2822)
```

This directive will generate the following PHP code:

```php
<div>
   <?php echo date(DATE_RFC2822); ?>
</div>
```

:::

::::

To access specific values passed to the directive separated by a comma:

```php app/src/Application/Stempler/DatetimeDirective.php
public function renderDateTime(Directive $directive): string
{
    return \sprintf(
        '<?php echo date(%s, %s); ?>',
        $directive->values[0], // first value passed to the directive
        $directive->values[1] // second value passed to the directive
    );
}
```

Example:

```php
@php
$format = 'Y-m-d H:i:s';
$timestamp = 199999999;
@endphp

<div>
    @dateTime($format, $timestamp)
</div>
```

This directive will generate the following PHP code:

```php
<?php
$format = 'Y-m-d H:i:s';
$timestamp = 199999999;
?>

<div>
    <?php echo date($format, $timestamp); ?>
</div>
```

> **Warning**
> The values are not automatically escaped, so you must escape them manually before using them.

#### Accessing Directive Context

To get information about where a directive is invoked from, use the `$directive->getContext()->getPath()` method:

```php
public function renderDateTime(Directive $directive): string
{
    return \sprintf(
        <<<PHP
<?php echo date(%s, %s); ?>
<!-- invoked from "%s" template -->
PHP,
        $directive->values[0],
        $directive->values[1],
        $directive->getContext()->getPath()
    );
}
```

When this directive is processed, it will generate the following PHP code:

```php
<div>
    <?php echo date($format, $timestamp); ?>
    <!-- invoked from "welcome" template -->
</div>
```

## Inheritance and Stacks

As your views get more complex, it's crucial to separate pages and layout specific content between templates properly.
Stempler provides several control statements to achieve this.

### Extend Layout

Firts, let's create a standard HTML template for our page:

```html app/views/home.dark.php
<!DOCTYPE html>
<html>
    <head>
        <title>This is homepage.</title>
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </head>
    <body>
        Page content
    </body>
</html>
```

Most likely, your application will contain more than a one-page template. To avoid code duplication, Stempler
provides an ability to inherit the parent layout.

> **Note**
> Stempler will compile the template and parent layout into an optimized PHP code. You can exclude as many layouts as
> you want without a performance penalty.

Create a layout:

```html app/views/layout/base.dark.php
<!DOCTYPE html>
    <html>
    <head>
        <title>This is homepage.</title>
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </head>
    <body>
        Page content
    </body>
</html>
```

Now, we can extend this layout using the `home.dark.php` via `extends:path` tag:

```html app/views/home.dark.php
<extends:layout.base/>
```

> **Note**
> Use the separator `.` to include the directory name into your template.

Alternatively, use the syntax:

```html app/views/home.dark.php
<extends path="layout/base"/>
```

> **Note**
> You can use view namespaces in such a declaration, for example: `<extends path="default:layout/base"/>`.

#### Replace Blocks

Extending the parent layout does not make much sense unless we can redefine some of its content. To define a
replaceable block, use the tag `<block:name>`. Change the `layout/base.dark.php` accordingly:

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title><block:title/></title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body>
<block:content>
    Default content body.
</block:content>
</body>
</html>
```

> **Note**
> You can include the default block content inside the `<block:name></block:name>` tag pair.

To redefine the block values, use `block:name` or similar tags in the `home.dark.php` template:

```html app/views/home.dark.php
<extends:layout.base/>

<block:title>Homepage</block:title>

<block:content>
    This is homepage content.
</block:content>
```

#### Short Syntax

In cases when your block define a short string or operates as a tag argument, use the alternative
syntaxt `${name|default}`. Change the layout to:

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
</html>
```

Short syntax values can be supplied to the parent layout via `<block:name>value</block:name>` tags.

```html app/views/home.dark.php
<extends:layout.base/>

<block:title>Homepage</block:title>

<block:body-class>homepage</block:body-class>

<block:content>
    This is homepage content.
</block:content>
```

You can pass some block values using the `extends` tag attributes to avoid large child templates,
change `app/views/home.dark.php` accordingly:

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage"/>

<block:content>
    This is homepage content.
</block:content>
```

In both cases, the produced HTML will look like this:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage">
This is homepage content.
</body>
</html>
```

#### Invoke Parent Content

To leave the parent block content, use `<block:parent/>` in any place of the redefined block:

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage"/>

<block:content>
    This is homepage content.
    <block:parent/>
</block:content>
```

The produced HTML:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage">
This is homepage content.
Default content.
</body>
</html>
```

Use `${parent}`, to achieve the same goal in short block definitions:

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<block:content>
    This is homepage content.
    <block:parent/>
</block:content>
```

The output:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage default">
This is homepage content.
Default content.
</body>
</html>
```

#### Nested Layouts

It is possible to create layouts based on other layouts, create `app/views/layout/page.dark.php`:

```html app/views/layout/page.dark.php
<extends:layout.base body-class="page ${parent}"/>

<block:content>
    <div class="page-wrapper">
        <block:page/>
    </div>
</block:content>
```

> **Note**
> Extend tags always require full path specification, make sure to include the `layout` directory.

You can extend this layout instead of `base` in `app/views/home.dark.php`:

```html app/views/home.dark.php
<extends:layout.page title="Homepage" body-class="homepage ${parent}"/>

<block:page>
    Page content.
</block:page>
```

The produced HTML:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage page default">
<div class="page-wrapper">
    Page content.
</div>
</body>
</html>
```

> **Note**
> You can nest as many templates as you need, it will only affect the compilation speed.

### Stacks

Stempler includes the ability to aggregate multiple blocks defined within the template.

#### Classic Approach

You would often need to add a custom JS or CSS resource to your layout. To achieve it, use the `block` directives,
wrap the necessary resources in a block and append content to it in your child template.

Modify `app/views/layout/base.dark.php` as:

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <block:styles>
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </block:styles>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
</html>
``` 

To add a custom style resource in your page template:

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<block:styles>
    <block:parent/>
    <link rel="stylesheet" href="/styles/homepage.css"/>
</block:styles>

<block:page>
    Page content.
</block:page>
```

The produced HTML:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
    <link rel="stylesheet" href="/styles/homepage.css"/>
</head>
<body class="homepage default">
Default content.
</body>
</html>
```

#### Create Stack

To demonstrate how the following can be achieved using stacks, we should start with a simple example
in `app/views/home.dark.php`. Create a stack placeholder using `<stack:collect name="name"/>`:

```html app/views/home.dark.php
collect name="my-stack">
    default content
</stack:collect>
```

To append a value to stack:

```html
<stack:collect name="my-stack">
    default content
</stack:collect>

<stack:push name="my-stack">
    my value
</stack:push>
```

The resulted HTML:

```html
  default content
my value
```

To prepend a value to stack:

```html
<stack:collect name="my-stack">
    default content
</stack:collect>

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

The output:

```html
  my value
default content
```

You can locate stack definition before or after the `push` and `prepend` tags:

```html
<stack:prepend name="my-stack">
    my value
</stack:prepend>

<stack:collect name="my-stack">
    default content
</stack:collect>
```

#### Deep Stacks

The stack tag will only aggregate `push` and `prepend` values if it's located on the same tag tree level.

For example, this will work:

```html
<stack:collect name="my-stack">
    default content
</stack:collect>

// stack my-stack is active here
<div>
    // and here
    <stack:prepend name="my-stack">
        my value
    </stack:prepend>
</div>
```

While this example won't work:

```html
<div>
    // stack my-stack is active here
    <stack:collect name="my-stack">
        default content
    </stack:collect>
    // and here
</div>

// stack my-stack is not active at this level

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

> **Note**
> This limitation is caused by the AST nature of stack collectors.

To bypass this limitation without moving the placeholder level higher, use the`stack:collect` attribute `level`:

```html
<div>
    <stack:collect name="my-stack" level="1">
        default content
    </stack:collect>
</div>

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

The attribute `level` configures the stack to be multiple active levels higher. For example, this
example **won't work**:

```html
<div>
    // stack my-stack is active here
    <div>
        // stack my-stack is active here
        <stack:collect name="my-stack" level="1">
            default content
        </stack:collect>
    </div>
</div>

// stack my-stack is no active at this level

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

But this one will:

```html
<div>
    // stack my-stack is active here
    <div>
        // stack my-stack is active here
        <stack:collect name="my-stack" level="2">
            default content
        </stack:collect>
    </div>
</div>

// stack my-stack is active here

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

#### Stacks in Layouts

You can push values to stacks defined in parent layouts. Modify `app/views/layout/base.dark.php` accordingly:

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <stack:collect name="styles" level="2">
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </stack:collect>
    <block:resources/>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
</html>
```

Now you can push the value from `app/views/home.dark.php`:

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<block:resources>
    <stack:push name="styles">
        <link rel="stylesheet" href="/styles/homepage.css"/>
    </stack:push>
</block:resources>

<block:page>
    Page content.
</block:page>
```

> **Note**
> You have to make sure that `stack:push` is located in one of the extended blocks. See how to bypass it below.

### Context and Hidden content

As you can see in the previous example, it's not convenient to use both the stack and blocks at the same time. This is
because that stack collection happens after the extension of the parent layout. Keeping the stack outside of any `block`
will leave it out of the template.

All the stempler blocks that are defined in the child template outside of the `block` tag will appear in the system
block `context`. We can modify the parent layout `app/views/layout/base.dark.php` like this:

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <stack:collect name="styles" level="2">
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </stack:collect>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
<block:context/>
</html>
```

Now we can define the stack in `app/views/home.dark.php` like this:

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<stack:push name="styles">
    <link rel="stylesheet" href="/styles/homepage.css"/>
</stack:push>

some random string

<block:page>
    Page content.
</block:page>
```

To understand how context works, take a look at the generated HTML:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
    <link rel="stylesheet" href="/styles/homepage.css"/>
</head>
<body class="homepage default">
Default content.
</body>
some random string
</html>
```

Notice that `some random string` is added instead of `block:context`, this content was declared
by `app/views/home.dark.php`. You will most likely use the areas between block definitions of your templates for
comments and other control directives.
To hide such content from end use, use the `<hidden></hidden>` tag in `app/views/layout/base.dark.php`:

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <stack:collect name="styles" level="2">
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </stack:collect>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
<hidden>
    <block:context/>
</hidden>
</html>
```

Now, stacking will work as before. However, `some random string` won't appear on a page.

> **Note**
> Combine stacks with inheritance and [components](#components-and-props) to create domain specific rendering DSL.

## Components and Props

Stempler provides an ability to create developer-driven template components as virtual tags.

### Simple Component

In many cases, your templates will not only reuse the parent layout, but also template partials, for example:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>

<block:content>
    This is the homepage.

    <div class="article">
        <div class="title">Article title</div>
        <div class="preview">article preview</div>
    </div>

    <div class="article">
        <div class="title">Article title 2</div>
        <div class="preview">article preview 2</div>
    </div>

    <div class="article">
        <div class="title">Article title 3</div>
        <div class="preview">article preview 3</div>
    </div>
</block:content>
```

We can move the article div into a separate template `app/views/partial/article.dark.php`:

```html app/views/partial/article.dark.php
<div class="article">
    <div class="title">Article title</div>
    <div class="preview">article preview</div>
</div>
```

To use this partial on your page, first import it using the `<use:element path=""/>` control tag:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:element path="partial/article"/>

<block:content>
    This is the homepage.

    <article/>
    <article/>
    <article/>
</block:content>
```   

> **See more**
> Read more about mass-importing partials below.

#### Props

It's is not very useful to create partials without the ability to configure their content. Use the `block:name`
or `${name|default}` syntax (similar to the one described [here](../stempler/inheritance.md)) to define replaceable
parts:

In our partial `app/views/partial/article.dark.php`:

```html app/views/partial/article.dark.php
<div class="article">
    <div class="title">${title}</div>
    <div class="preview">
        <block:preview>
            default preview
        </block:preview>
    </div>
</div>
```

You can pass values similar way as in the `extend` control tag:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:element path="partial/article"/>

<block:content>
    This is the homepage.

    <article>
        <block:title>Article 1 title</block:title>
        <block:preview>
            This is article 1 preview.
        </block:preview>
    </article>

    <article title="Article 2">
        <block:preview>
            <block:parent/>
            This is article 1 preview.
        </block:preview>
    </article>

    <article title="Article 3" preview="This is article 3 preview."/>
</block:content>
```

> **Note**
> You can include the original block content using the `block:parent` tag. Component expansion is also allowed.

The resulted HTML:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="default">
This is homepage.
<div class="article">
    <div class="title">Article 1 title</div>
    <div class="preview">
        This is article 1 preview.
    </div>
</div>

<div class="article">
    <div class="title">Article 2</div>
    <div class="preview">
        Default content.
        This is article 1 preview.
    </div>
</div>

<div class="article">
    <div class="title">Article 3</div>
    <div class="preview">
        This is article 3 preview.
    </div>
</div>
</body>
</html>
```

> **Note**
> Components do not cause any performance penalty, use as many components as you need.

### Import Components

Stempler provides several options for importing components into your template.

#### Import Element

To import a single component, use `<use:element path=""/>` before component invocation.

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:element path="partial/article"/>

<block:content>
    <article title="Article" preview="This is article preview."/>
</block:content>
```

The component will be available using the filename, in this case it's `article`. To define a custom import alias, use
the tag attribute `as`:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:element path="partial/article" as="custom-article"/>

<block:content>
    <custom-article title="Article" preview="This is article preview."/>
</block:content>
```

#### Import Directory

To import all the partials from a given directory, use `<use:dir dir="" ns=""/>`. You must specify a namespace prefix to
avoid collisions with other components and default HTML tags:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:dir dir="partial" ns="partials"/>

<block:content>
    <partials:article title="Article" preview="This is article preview."/>
</block:content>
```

#### Inline Import

To define a component specific to a given template without creating a physical view file,
use the `<use:inline name=""></use:inline>` control tag. In `app/views/home.dark.php`:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>

<use:inline name="article">
    <div class="article">
        <div class="title">${title}</div>
        <div class="preview">${preview}</div>
    </div>
</use:inline>

<block:content>
    <article title="Article" preview="This is article preview."/>
</block:content>
```

#### Bundle Import

Import multiple directories, components and/or inline components using bundled import via `<use:bundle path="">`.

Create a view file `app/views/my-bundle.dark.php` to define your bundle:

```html app/views/my-bundle.dark.php
<use:element path="partial/article" as="article"/>

<use:inline name="article-alt">
    <div class="article">
        <div class="title">${title}</div>
        <div class="preview">${preview}</div>
    </div>
</use:inline>
```

You can use any of the defined components in your `app/views/home.dark.php` template:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:bundle path="my-bundle"/>

<block:content>
    <article title="Article" preview="This is article preview."/>
    <article-alt title="Article" preview="This is article preview."/>
</block:content>
```

To isolate an imported bundle via the prefix, use the `ns` attribute of the `use:bundle` tag:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:bundle path="my-bundle" ns="my"/>

<block:content>
    <my:article title="Article" preview="This is article preview."/>
    <my:article-alt title="Article" preview="This is article preview."/>
</block:content>
```

### Props

The ability to pass values into components makes it possible to create complex elements that are condensed into simple
tags.
You are allowed to pass PHP values and echoes to your components.

Modify your controller to invoke the template like this:

```php
return $this->views->render('home', ['value' => 'Hello&world!']);
```

Create `app/views/partial/input.dark.php`:

```html app/views/partial/input.dark.php
<div class="input">
    <label>${label}</label>
    <input type="text" value="${value}"/>
</div>
```

You can invoke this component in your template with a user supplied value:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:element path="partial/input" as="my:input"/>

<block:content>
    <my:input label="Some Value" value="{{ $value }}"/>
</block:content>
```

The generated PHP:

```php
...
<body class="default">
    <div class="input">
        <label>Some Value</label>
        <input type="text" value="<?php echo htmlspecialchars($value, ENT_QUOTES | ENT_SUBSTITUTE, 'utf-8'); ?>"/>
    </div>
</body>
...
```

#### PHP in Components

Not only can you inject values into plain HTML, but you can also inject source code into a PHP component. It can be
achieved using an AST modification of the underlying template via the macro function `inject("name", default)`.

> **Note**
> The injection will automatically extract the variable or statement from the passed `{{ echo }}`, `<?php $variable ?>`
> or `<?=$variable?>` attributes.

To demonstrate it, modify `app/views/partial/input.dark.php`:

```html app/views/partial/input.dark.php
<div class="input">
    <label>${label}</label>
    <input type="text" value="{{ strtoupper(inject('value')) }}"/>
</div>
```

Now the generated code will look like this:

```html
<body class="default">
<div class="input">
    <label>Some Value</label>
    <input type="text"
           value="<?php echo htmlspecialchars(strtoupper($value), ENT_QUOTES | ENT_SUBSTITUTE, 'utf-8'); ?>"/>
</div>
</body>
```

You can pass PHP values in combination with string prefixes, in `app/views/home.dark.php`:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:element path="partial/input" as="my:input"/>

<block:content>
    <my:input label="Some Value" value="hello {{ $value }} world"/>
</block:content>
```

The compiled template:

```html
<body class="default">
<div class="input">
    <label>Some Value</label>
    <input type="text"
           value="<?php echo htmlspecialchars(strtoupper('hello '.$value.' world'), ENT_QUOTES | ENT_SUBSTITUTE, 'utf-8'); ?>"/>
</div>
</body>
```

#### Complex Props

You can inject your props not only in echo statements, but also in any PHP code of your component. Let's create
the `select` component `app/views/partial/select.dark.php`:

```html app/views/partial/select.dark.php
<select name="${name}">
    @foreach(inject('values', []) as $key => $label)
    <option value="{{ $key }}">{{ $label }}</option>
    @endforeach
</select>
```

Modify your controller to pass an array:

```php
return $this->views->render('home', [
    'values' => [1 => 'first', 2 => 'second']
]);
```

You can use this component in your template:

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>
<use:element path="partial/select" as="my:select"/>

<block:content>
    <my:select name="My Select" values="{{ $values }}"/>
</block:content>
```

The generated template:

```html
<body class="default">
<select name="My Select">
    <?php foreach($values as $key => $label): ?>
    <option value="<?php echo htmlspecialchars($key, ENT_QUOTES | ENT_SUBSTITUTE, 'utf-8'); ?>">
        <?php echo htmlspecialchars($label, ENT_QUOTES | ENT_SUBSTITUTE, 'utf-8'); ?>
    </option>
    <?php endforeach; ?>
</select>
</body>
```

You are allowed to inject PHP blocks into default PHP tags. `app/views/partial/select.dark.php` can be changed like
this:

```html app/views/partial/select.dark.php
<select name="${name}">
    <?php
  $selectValues = array_map('strtoupper', inject('values', []));
  ?>
    @foreach($selectValues as $key => $label)
    <option value="{{ $key }}">{{ $label }}</option>
    @endforeach
</select>
```

The generated template:

```html
<body class="default">
<select name="My Select">
    <?php
    $selectValues = array_map('strtoupper', $values);
    ?>
    <?php foreach($selectValues as $key => $label): ?>
    <option value="<?php echo htmlspecialchars($key, ENT_QUOTES | ENT_SUBSTITUTE, 'utf-8'); ?>">
        <?php echo htmlspecialchars($label, ENT_QUOTES | ENT_SUBSTITUTE, 'utf-8'); ?>
    </option>
    <?php endforeach; ?>
</select>
</body>
```

> **Note**
> Attention, make sure to escape your values properly!

#### Dynamic Attributes

In some cases, you might want to bypass some attributes into elements directly. For example, to a allow user-driven
`style` attribute for select, we have to do the following:

```html
<select name="${name}" style="${style}">
    @foreach(inject('values', []) as $key => $label)
    <option value="{{ $key }}">{{ $label }}</option>
    @endforeach
</select>
```

Use `attr:aggregate` to scale this approach:

```html
<select name="${name}" attr:aggregate>
    @foreach(inject('values', []) as $key => $label)
    <option value="{{ $key }}">{{ $label }}</option>
    @endforeach
</select>
```

Now we can pass arbitrary attributes to our component from `app/views/home.dark.php`:

```html
<extends:layout.base title="Homepage"/>
<use:element path="partial/select" as="my:select"/>

<block:content>
    <my:select name="My Select" values="{{ $values }}" style="color: red" class="custom-select"/>
</block:content>
```

The resulted HTML:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="default">
<select name="My Select" style="color: red" class="custom-select">
    <option value="1">first</option>
    <option value="2">second</option>
</select>
</body>
</html>
```

## Advancing DSL

Combine Stempler features such as props, AST code injections, stacks, and inheritance to develop a feature-rich domain
language for page definitions.

> **Note**
> Make sure to learn all the other aspects of Stempler before jumping into this section. You will also need a good cup
> of coffee.

The approach described in this article is possible because stacks can be defined within imported components.

### Grid

Let's create a grid component to describe how to use stacks in combination with partials. The grid will consist of
multiple view files.

#### Table

Create a root element for the component `app/views/grid/table.dark.php`:

```html app/views/grid/table.dark.php
<table class="grid-table">
    <thead>
    <stack:collect name="thead" level="2"/>
    </thead>
    <tbody>
    <stack:collect name="tbody" level="2"/>
    </tbody>
    <hidden>${context}</hidden>
</table>
```

> **Note**
> Note  `<hidden>${context}</hidden>`, it allows the component to handle the content declared between the open and
> close tags without the need for `block` tags.

#### Cell

Create an element `app/views/grid/cell.dark.php` to declare a single table cell with its header and value:

```html app/views/grid/cell.dark.php
<stack:push name="thead">
    <th>${title}</th>
</stack:push>

<stack:push name="tbody">
    <td>${value}${context}</td>
</stack:push>
```

> Note the `${context}`.

#### Bundle

We can pack our elements into the bundle `app/views/grid/bundle.dark.php`:

```html app/views/grid/bundle.dark.php
<use:element path="grid/table" as="grid:table"/>
<use:element path="grid/cell" as="grid:cell"/>
```

#### Example

To render the grid in our template:

```html
<extends:layout.base title="Homepage"/>
<use:bundle path="grid/bundle"/>

<block:content>
    <grid:table>
        <grid:cell title="First cell" value="my value"/>
        <grid:cell title="Second cell" value="another value"/>
    </grid:table>
</block:content>
```

You can directly pass values into the cell `context`:

```html
<extends:layout.base title="Homepage"/>
<use:bundle path="grid/bundle"/>

<block:content>
    <grid:table>
        <grid:cell title="First cell">my value</grid:cell>
        <grid:cell title="Second cell">second value</grid:cell>
    </grid:table>
</block:content>
```

In both cases, the produced HTML is:

```html
<body class="default">
<table class="grid-table">
    <thead>
    <th>First cell</th>
    <th>Second cell</th>
    </thead>
    <tbody>
    <td>my value</td>
    <td>second value</td>
    </tbody>
</table>
</body>
```

### UI Assembly

Another example is related to the ability to assemble complex UI interfaces using custom DSL.

#### Base Layout

To demonstrate complex UI assembly, we are going to create an interface with the ability to easily push CSS, JS
resources, and define context using multiple tabs instead of a single content block.

Create `app/views/tabs/layout.dark.php`:

```html app/views/tabs/layout.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <stack:collect name="styles" level="2"/>
</head>
<body>
<div class="tab-headers">
    <stack:collect name="tab-headers" level="2"/>
</div>
<div class="tab-body">
    <stack:collect name="tab-body" level="2"/>
</div>
</body>
<stack:collect name="scripts" level="2"/>
<hidden>${context}</hidden>
</html>
```

### Style and Script elements

To simplify registration of style and script elements, create the components `app/views/tabs/script.dark.php`
and `app/views/tabs/style.dark.php`:

```html app/views/tabs/script.dark.php
<stack:push name="scripts">
    <script src="${src}"></script>
</stack:push>
```

Style:

```html app/views/tabs/style.dark.php
<stack:push name="styles">
    <link rel="stylesheet" href="${src}"/>
</stack:push>
```

#### Tab Element

Create the tab element similar to `grid:cell` in `app/views/tabs/tab.dark.php`:

```html app/views/tabs/tab.dark.php
<stack:push name="tab-headers">
    <div class="tab-head" data-tab-body="${id}">${title}</div>
</stack:push>

<stack:push name="tab-body">
    <div class="tab-body" id="body-${id}">${context}</div>
</stack:push>
```

#### Bundle

Create a bundle to represent the DSL for your UI framework `app/views/tabs/bundle.dark.php`:

```html app/views/tabs/bundle.dark.php
# resources
<use:element path="tabs/style" as="import:style"/>
<use:element path="tabs/script" as="import:script"/>

# tab
<use:element path="tabs/tab" as="ui:tab"/>
```

#### Example

To render complex UI, modify `app/views/home.dark.php`.

> **Note**
> You can import more than one bundle.

```html app/views/home.dark.php
<extends:tabs.layout title="Homepage"/>
<use:bundle path="grid/bundle"/>
<use:bundle path="tabs/bundle"/>

<import:style src="/resources/my-style.css"/>
<import:script src="/resources/my-script.js"/>

<ui:tab id="first" title="Information">
    Hello world!
</ui:tab>

<ui:tab id="second" title="Some Grid">
    <grid:table>
        <grid:cell title="First cell">my value</grid:cell>
        <grid:cell title="Second cell">second value</grid:cell>
    </grid:table>
</ui:tab>
```

The generated HTML:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/resources/my-style.css"/>
</head>
<body>
<div class="tab-headers">
    <div class="tab-head" data-tab-body="first">Information</div>
    <div class="tab-head" data-tab-body="second">Some Grid</div>
</div>
<div class="tab-body">
    <div class="tab-body" id="body-first">
        Hello world!
    </div>
    <div class="tab-body" id="body-second">
        <table class="grid-table">
            <thead>
            <th>First cell</th>
            <th>Second cell</th>
            </thead>
            <tbody>
            <td>my value</td>
            <td>second value</td>
            </tbody>
        </table>
    </div>
</div>
</body>
<script src="/resources/my-script.js"></script>
</html>
```

## AST Modifications

The Stempler template engine fully exposes template AST (DOM) and provides an API for modifications similar to
https://github.com/nikic/PHP-Parser.

You can create magical (in both ways) workflows and helpers by implementing your Node Visitors.

> **See more**
> You can read more about how traversing
>
works [here](https://github.com/nikic/PHP-Parser/blob/master/doc/2_Usage_of_basic_components.markdown#node-traversation).

### Create Visitor

To create an AST visitor, you must implement the interface provided by the Stempler engine

- `Spiral\Stempler\VisitorInterface`.

We will try to create a visitor that automatically adds an `alt` attribute to all `img` tags found in your templates:

```php
use Spiral\Stempler\Node\HTML;
use Spiral\Stempler\VisitorContext;
use Spiral\Stempler\VisitorInterface;

class AltImageVisitor implements VisitorInterface
{
    public function enterNode($node, VisitorContext $ctx)
    {
    }

    public function leaveNode($node, VisitorContext $ctx)
    {
        if ($node instanceof HTML\Tag && $node->name === 'img') {
            $alt = null;
            foreach ($node->attrs as $attr) {
                if ($attr->name === 'alt') {
                    $alt = $attr;
                    break;
                }
            }

            if ($alt === null) {
                $node->attrs[] = new HTML\Attr('alt', '"this is image alt"');
            }
        }
    }
}
```

> **Note**
> You can inject other tags or even PHP into your templates.

### Register Visitor

Call `StemplerBootloader`->`addVistitor` to register visitors in the template engine. We can do it with the application
bootloader:

```php app/src/Application/Bootloader/AltImageBootloader.php
namespace App\Application\Bootloader;

use App\Visitor\AltImageVisitor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class AltImageBootloader extends Bootloader
{
    public function boot(StemplerBootloader $stempler)
    {
        $stempler->addVisitor(AltImageVisitor::class);
    }
}
```

> **Note**
> You have to clean view cache to view newly applied changes.

Now all the `img` tags will always include the `alt` attribute.

### Standalone Usage

You can use Stempler to process any HTML content.

```php
use Spiral\Stempler;

$parser = new Stempler\Parser();
$parser->addSyntax(
    new Stempler\Lexer\Grammar\HTMLGrammar(), 
    new Stempler\Parser\Syntax\HTMLSyntax()
);

$template = $parser->parse(new Stempler\Lexer\StringStream("<BODY>content</BODY>"));

$traverser = new Stempler\Traverser();
$traverser->addVisitor(new CustomVisitor());

$template->nodes = $traverser->traverse($template->nodes);

$compiler = new Stempler\Compiler();
$compiler->addRenderer(new Stempler\Compiler\Renderer\CoreRenderer());
$compiler->addRenderer(new Stempler\Compiler\Renderer\HTMLRenderer());

dump($compiler->compile($template)->getContent());
```

## Installation and Configuration

To install the extensions in alternative bundles:

```terminal
composer require spiral/stempler-bridge
```

The `Spiral\Stempler\Bootloader\StemplerBootloader` bootloader must be added to the application kernel to enable its
usage.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Stempler\Bootloader\StemplerBootloader::class,
    // ...
];
```

The Stempler bridge comes pre-configured. To replace and alter the default configuration, create a
file `app/config/views/stempler.php`.

<details>
    <summary>Click to show configuration file content</summary>

```php app/config/views/stempler.php
use Spiral\Stempler\Builder;
use Spiral\Stempler\Directive;
use Spiral\Stempler\Transform\Finalizer;
use Spiral\Stempler\Transform\Visitor;
use Spiral\Views\Processor;

return [
    'directives' => [
        // available Blade-style directives
        Directive\PHPDirective::class,
        Directive\RouteDirective::class,
        Directive\LoopDirective::class,
        Directive\JsonDirective::class,
        Directive\ConditionalDirective::class,
        Directive\ContainerDirective::class
    ],
    'processors' => [
        // cache depended source processors (i.e. LocaleProcessor)
        Processor\ContextProcessor::class
    ],
    'visitors'   => [
        Builder::STAGE_PREPARE   => [
            // visitors to be invoked before transformations
            Visitor\DefineBlocks::class,
            Visitor\DefineAttributes::class,
            Visitor\DefineHidden::class
        ],
        Builder::STAGE_TRANSFORM => [
            // visitors to be invoked during transformations
        ],
        Builder::STAGE_FINALIZE  => [
            // visitors to be invoked on after the transformations is over
            Visitor\DefineStacks::class,
            Finalizer\StackCollector::class,
        ],
        Builder::STAGE_COMPILE   => [
            // visitors to be invoked on compilation stage
        ]
    ]
];
```

</details>

> **Note**
> Prefer to use the `Spiral\Stempler\Bootloader\StemplerBootloader` bootloader to configure the engine.

## Pretty Printing

The bridge comes with an additional bootloader called `Spiral\Stempler\Bootloader\PrettyPrintBootloader`, which is
responsible for pretty printing the HTML content of your templates.
