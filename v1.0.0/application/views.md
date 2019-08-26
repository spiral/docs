# Application Views and Templates
Spiral does provide ViewModel implementation by default, but rather simply access to rendering functions of view templates.

By default your application view templates are located in 'app/views' directory and can be invoked by file name.

> Read more about [view component](/v1.0.0/views/overview.md) to understand how to configure view namespaces and connect more rendering engines.

You can render any of your template using `ViewsInterface` dependency or shortcut `views`:

```php
public function indexAction()
{
    return $this->views->render('template', [
        'variable' => 'Some value'
    ]);
}
```

## ViewModels
Be free to implement your own buffer class between rendering engine and controllers when needed:

```php
class ViewModel extends Service 
{
    private $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function __toString()
    {
        return $this->views->render('template', [
            'name' => $this->name
        ]);
    }
}
```

```php
public function indexAction()
{
    return new ViewModel('some name');
}
```