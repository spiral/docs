# Views - Rendering Views
In order to render view you must obtain an instance of `Spiral\Views\ViewsInterface`.

> The component is available via prototyped property `views`.

## Rendering
To render view in controller or other service simply invoke `render` method of `ViewsInterface`. The view name
does not need to include extension or namespace (default to be used).

```php
use Spiral\Views\ViewsInterface;

// ...

public function index(ViewsInterface $views)
{
    return $views->render('home');
}
```

To render view with passed data use second array argument:

```php
use Spiral\Views\ViewsInterface;

// ...

public function index(ViewsInterface $views)
{
    return $views->render('home', [
            'key' => 'value'
    ]);
}
```

#### Namespaces
To render view from specific namespace prepend it to view name using `:` separator:

```php
return $views->render('namespace:home', [
    'key' => 'value'
]);
```

## View Object
In some cases it might be more performant to cache the view in stateless component, obtain view object (`Spiral\Views\ViewInterface`)
using method `get` of `ViewsInterface`:

```php
$view = $views->get('home');
```  

To render obtained view call `render` method:

```php
return $view->render([
    'key' => 'value'
]);
```
