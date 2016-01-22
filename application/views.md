# Application Views and Templates
By default you can locate your layouts and page templates into `app/views` directory which is associated
with defult [view namespace](/components/views.md), once template has been created (for example `app/views/templater.php`) 
you can render it using `ViewsInterface` or short bindings `views` in your services.

```php
public function indexAction()
{
    return $this->views->render('template', [
        'variable' => 'Some value'
    ]);
}
```

> Check out guide for [View Component](/components/views.md) and [Stemple Engine](/stemplers/basics.md).
