# Application Views
By default your locate your layouts and page templates into `app/views` directory, once template 
created (for example `app/views/templater.php`) you can render it's content using `ViewsInterface` 
or short bindings views in your services.

```php
public function indexAction()
{
    return $this->views->render('template', [
        'variable' => 'Some value'
    ]);
}
```

> Read more about how views work [here](/components/views.md). To find guide for Spiral Stemple engine, go [here](/stemplers/basics.md).
