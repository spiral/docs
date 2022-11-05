# Views - Plain PHP

You can use plain php files as your views without any compilation or caching layer. This engine is enabled by default and is
available in the Web application skeleton.

## Creating view

To create a plain PHP view, simply put the file with `.php` extension into the `views` directory. The template can be rendered by it's filename:

```php
<?php // test.php
echo "hello world";
```

```php
public function index(ViewsInterface $views): string
{
    return $views->render('test'); // no need to specify the extension
}
```

All the passed parameters will be available as PHP variables:

```php
public function index(ViewsInterface $views): string
{
    return $views->render('test', ['name' => 'Antony']); 
}
```

Where `test.php`:

```php
Hello , <?= $name ?>!
```

## Container

You can access the container in your view files via `$this->container`:

```php
Hello world, <?= $name ?>!

<?php dump($this->container->get(MyService::class)); ?>
```
