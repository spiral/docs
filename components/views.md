# Views
Spiral view manager counted as part of framework bundle and not components set. Default implementation includes support for multiple rendering engines (copiler + renerer), caching, cache dependencies and **view namespaces**.

## Examples
Before we will start digging into features and functionality of ViewManager let's try to view few examples. First of all we have to create view file to be rendered, as in many other frameworks you can pass variables into such file, we are going to name our file "demo" and place it into "application/views" directory.

```php
This is demo view file: <?= $value ?>
```

Next we can render this file using `ViewsInterface` dependency or short binding "views" (we are going to work inside controller):

```php
public function index()
{
    return $this->views->render('demo', [
        'value' => mt_rand(0, 1000)
    ]);
}
```

If we will open our action in browser we will be able to see rendered file with random value in it. Every view file rendered using `ViewInterface` classes, default spiral implementation of such interface located in `Spiral\Views\View` and it provides us ability to access to view object and container located into it. Let's update both view file and our controller code.

Let's use view container to access some service or class inside our view:

```php
This is demo view file: <?= $value ?>

<?php
/**
 * @var \Spiral\Debug\Dumper $dumper
 */
$dumper = $this->container->get(\Spiral\Debug\Dumper::class);
$dumper->dump($value);
?>
```

In addition to that we can use 'get' method of our ViewManager to receive and instance of View and, for example, configure it before rendering:

```php
public function index()
{
    /**
     * @var View $view
     */
    $view = $this->views->get('demo');
    $view->set('value', mt_rand(0, 1000));

    //We can do return $view; due MiddlewarePipeline
    //converts every response into string
    return $view->render();
}
```

Provided code is fully identical to short 'render' method, hovewer it gives us ability to manipulate with View class before rendering. Due spiral provides ability to register additional engines associated with specified file extension and custom view rendered such ability can be very useful in some cases.

## View Namespaces
One of the core idea of ViewManager is to store view filenames into multiple locations aggregated by namespace. While rendering/compilation phase ViewManager will walk though every namespace directory until view file not found. View file identified purelly by it's name, when extension used to locate what rendering/compilation engine to be used. Let's look into our view configuration to check how namespaces defined:

```php
'namespaces'   => [
    'default'  => [
        directory("application") . '/views'
    ],
    'spiral'   => [
        directory("application") . '/views/spiral',
        directory("libraries") . '/spiral/framework/source/views'
    ],
    'profiler' => [
        directory("libraries") . '/spiral/profiler/source/views'
    ]
],
```

You can specify view name with namespace using ":" separator. If no namespace were entered "default" will be used instead, this makes us possible to rewrite first example:

```php
public function index()
{
    return $this->views->render('default:demo', [
        'value' => mt_rand(0, 1000)
    ]);
}
```

The beauty about namespaces that due ViewManager will be looking for file in order of declared directories we can change views used by other spiral components or modules. In our case let's remember http configuration:

```php
'httpErrors'   => [
    400 => 'spiral:http/badRequest',
    403 => 'spiral:http/forbidden',
    404 => 'spiral:http/notFound',
    500 => 'spiral:http/serverError',
]
```

As you can see, in case of soft http error system will to render view located in "spiral" namespace, which is associated with two directories:

```php
'spiral'   => [
    directory("application") . '/views/spiral',
    directory("libraries") . '/spiral/framework/source/views'
],
```

Let's try to alter HTTP 404 error page by creating view in 'application/views/spiral/http' directory. We are going to overwrite 'notFound' page:

```php
This page can not be found.
```

> We can also simply point http to another view. :)

Now, if we will cause 404 error our view file will be rendered. If you already caused 404 error before it might be already pre-compiled with original view, we can reset it by either disabling cache, flushing content of "application/runtime/cache/views" (default cache directory) or by executing command "views:compile" which will re-compile available view files.

## View Engines


## Cache Dependencies


## Spiral View Compiler and Renderer